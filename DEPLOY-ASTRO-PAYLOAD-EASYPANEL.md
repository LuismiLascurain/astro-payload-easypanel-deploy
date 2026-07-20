# Manual de despliegue — Astro 5 + Payload CMS 3 en Easypanel

> **Nota de versiones (julio 2026):** este manual se escribió y validó con
> **Astro 5 + Payload 3 + Node 22**. Astro 6 y 7 exigen **Node 22.12+** (los
> Dockerfiles de aquí ya usan `node:22-slim`, verifica la minor). He validado
> Astro 7 en sitios estáticos sin CMS sin ninguna incidencia.
>
> **Actualización (laskurain.es, 2026-07-07): la combinación completa
> Astro 7 + Payload 3 está VALIDADA en producción** con este patrón (monorepo
> pnpm, Next 16 webpack, `node:22-alpine`). El despliegue destapó **cuatro
> trampas nuevas**, todas en §6.15–§6.18: el marcador `dev` de las migraciones
> colgando `migrate` (§6.15), el type-check de `next build` más estricto que
> dev (§6.16), la config de Payload importando de otro workspace y reventando
> en runtime (§6.17), y la verificación de formularios por navegador vs curl
> (§6.18). Puntos que ya venían señalados y se confirman: el compilador estricto
> y el type-check de producción (§6.16).

Manual reutilizable para desplegar la combinación Astro 5 (sitio estático) +
Payload CMS 3 (Next 15) + Postgres en Easypanel sobre un VPS propio. Surge del
despliegue real de un proyecto de cliente — todas las trampas listadas son cosas que
nos costaron tiempo durante esa sesión.

---

## 1. Resumen ejecutivo

| Capa | Pieza |
|---|---|
| Sitio público | Astro 5 con `output: 'static'`, fetcha el CMS **en build** |
| CMS | Payload 3 sobre Next.js 15 (App Router puro, sin Express) |
| Base de datos | Postgres 16 |
| Runtime del web | Caddy 2 Alpine sirviendo `dist/` estático |
| Plataforma | Easypanel (sobre un VPS propio en nuestro caso) |
| Build context | Raíz del monorepo (pnpm workspaces) |
| Repositorio | Monorepo con `apps/cms/` + `apps/web/` |

**Tiempo realista de despliegue:**

- **~30 min** si ya conoces este manual y la infraestructura está montada.
- **3–4 h** la primera vez (la mayor parte se va en trampas, no en código).

---

## 2. Arquitectura objetivo

Diagrama textual de servicios en Easypanel:

```
┌──────────────────────────────────────────────────────────────────┐
│                         Easypanel (VPS)                           │
│                                                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐   │
│  │   Postgres   │◄───┤     CMS      │    │       Web         │   │
│  │  (servicio)  │    │  (Payload 3) │    │   (Astro static)  │   │
│  │              │    │   Next 15    │    │   Caddy :3000     │   │
│  │   :5432      │    │   :3000      │    │                   │   │
│  └──────────────┘    └──────────────┘    └──────────────────┘   │
│                            ▲                      │              │
│                            │                      │              │
│                            │  build-time fetch    │              │
│                            └──────────────────────┘              │
│                                                                   │
│         Caddy de borde (lo gestiona Easypanel, no tú)            │
│         · TLS / Let's Encrypt                                    │
│         · Dominio público → :3000 del contenedor                 │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                   Internet (cms.dominio.com,
                             www.dominio.com)
```

**Cómo se hablan entre ellos:**

- `CMS ↔ Postgres`: por red interna del proyecto Easypanel. El `DATABASE_URI`
  del CMS apunta al hostname interno del servicio Postgres (no a `localhost`,
  no a la IP del VPS).
- `Web ↔ CMS`: **solo durante el build del web**. Astro hace `fetch()` a la
  API REST de Payload en `getStaticPaths` y en algunos `frontmatter`, hornea
  el resultado en HTML, y el contenedor en runtime ya no habla con el CMS
  para nada — es Caddy sirviendo archivos.

**Implicación clave** (no obvia hasta que muerde):

> Cualquier cambio en el CMS (publicar un artículo, editar una página legal,
> cambiar un texto del global `site-settings`) requiere **redeploy del
> servicio web** para que el cambio aparezca en producción.

La solución a largo plazo es un webhook `afterChange` en las collections
relevantes que dispare el rebuild de Easypanel automáticamente. Las
variables `REVALIDATE_WEBHOOK_URL` y `REVALIDATE_WEBHOOK_SECRET` del CMS
(§4.2) están pensadas exactamente para esto: el hook `afterChange` haría
POST a esa URL con el secret como shared-secret. **En aquel primer proyecto la
config de las variables está presente pero el endpoint en el web nunca
se llegó a implementar** — quedó como reserva. Por eso el `afterChange`
sigue como TODO inline en `apps/cms/src/collections/Articles.ts`: el
plumbing de variables existe, falta el código que las consume.

> **Actualización (luismilascurain.com, 2026-05): RESUELTO de verdad — ver §6.14.**
> Dos hallazgos que cambian el plan de arriba: (1) el deploy hook simple hace
> build **con caché** y no rehornea el contenido → hay que usar la **API tRPC
> `services.app.deployService` con `forceRebuild:true`**, no el webhook; (2) desde
> un contenedor del VPS solo es alcanzable el **dominio HTTPS del panel** (ni la
> IP ni los gateways internos de Swarm). El `REVALIDATE_WEBHOOK_*` reservado aquí
> queda obsoleto; las vars reales son `EASYPANEL_API_URL/_TOKEN/_PROJECT/_WEB_SERVICE`.

---

## 3. Orden de despliegue (con justificación)

El orden importa porque hay dependencias temporales reales:

1. **Postgres** (servicio independiente).
   No depende de nada. Tiene que estar arriba antes que el CMS porque el
   CMS abre el pool de conexiones al arrancar.

2. **CMS — primer arranque + migraciones iniciales**.
   En el primer deploy la base está vacía. El `CMD` del Dockerfile aplica
   las migraciones generadas en local (ver §4.3). Después de esto la base
   tiene todas las tablas vacías y el admin de Payload responde 200.

3. **Cargar contenido inicial en el CMS**.
   Crear el primer usuario admin, dar de alta categorías, subir el primer
   artículo. **Sin contenido publicado, el build del web fallará o se
   quedará sin nada que renderizar.**

4. **Publicar el contenido** (`status: published`).
   Borradores no se exponen en `/api/articles` cuando el fetch va sin token
   (lectura pública filtra `status === 'published'` por defecto en este
   stack — verifica tus `access.read` antes de asumir).

5. **Web — primer build**.
   Astro hace fetch al CMS público (`PAYLOAD_API_URL=https://cms.dominio.com`,
   sin `/api`, ver §6.6). Si el CMS está caído, los fetchers devuelven `null`
   y el sitio renderiza fallbacks — pero las páginas dinámicas (`/escritos/[slug]`)
   no se generan si `getStaticPaths` devuelve `[]`.

6. **DNS + dominios públicos**.
   `cms.dominio.com` → servicio CMS. `www.dominio.com` y `dominio.com` →
   servicio web. Easypanel saca los certs Let's Encrypt automáticamente
   en la pestaña Dominios de cada servicio. Hasta aquí estuviste usando
   las URLs temporales de Easypanel (`<proyecto>-<servicio>.<panel>.easypanel.host`).

7. **Caddyfile con redirecciones (si hay sitio anterior)**.
   Si vienes de un dominio antiguo o con cambios de slugs, añade
   redirecciones 301 al Caddyfile del web ANTES del bloque `file_server`.
   Ver §6.13.

8. **Search Console y SEO**.
   Crear propiedad en la URL canónica (decidir antes www o sin-www y
   mantener coherencia en sitemap, canonicals, redirects y robots).
   Enviar sitemap. Solicitar indexación de URLs prioritarias.

---

## 4. Configuración del CMS

### 4.1 Dockerfile

Referencia: `apps/cms/Dockerfile`.

Estructura multi-stage:

- **Builder** (`node:22-slim`): instala dependencias del workspace
  (`pnpm install --filter cms...`), copia el código del CMS y compila Next
  (`pnpm run build`).
- **Runner** (`node:22-slim`): copia solo lo necesario, ejecuta migraciones
  y arranca Next.

Patrón base:

```dockerfile
FROM node:22-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 build-essential \
    && rm -rf /var/lib/apt/lists/*
RUN corepack enable && corepack prepare pnpm@9.15.0 --activate
WORKDIR /app

# Manifests primero (cache friendly)
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/cms/package.json ./apps/cms/
COPY apps/web/package.json ./apps/web/
COPY tsconfig.base.json ./                    # ← CRÍTICO (ver §6.2)

RUN pnpm install --frozen-lockfile --filter cms...

COPY apps/cms ./apps/cms
WORKDIR /app/apps/cms
RUN pnpm run build

FROM node:22-slim AS runner
RUN corepack enable && corepack prepare pnpm@9.15.0 --activate
WORKDIR /app

COPY --from=builder /app/package.json /app/pnpm-lock.yaml /app/pnpm-workspace.yaml ./
COPY --from=builder /app/apps/cms/package.json ./apps/cms/
COPY --from=builder /app/tsconfig.base.json ./           # ← CRÍTICO (ver §6.3)
COPY --from=builder /app/apps/cms/tsconfig.json ./apps/cms/
COPY --from=builder /app/apps/cms/.next ./apps/cms/.next
COPY --from=builder /app/apps/cms/next.config.* ./apps/cms/
COPY --from=builder /app/apps/cms/src ./apps/cms/src     # ← CRÍTICO (ver §6.3)
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/apps/cms/node_modules ./apps/cms/node_modules

WORKDIR /app/apps/cms
ENV NODE_ENV=production
ENV PORT=3000
EXPOSE 3000

CMD ["sh", "-c", "pnpm payload migrate && pnpm exec next start -p 3000"]
```

Puntos no negociables:

- **Build context = raíz del monorepo** (en Easypanel, build path `/`).
  Sin esto no puedes copiar `pnpm-lock.yaml`, `pnpm-workspace.yaml` ni los
  manifests de las dos apps. Ver §6.1.
- **`tsconfig.base.json` se copia en builder y en runner.** El CLI de Payload
  (`pnpm payload migrate`, `migrate:fresh`, etc.) lo lee en runtime porque
  `apps/cms/tsconfig.json` lo extiende. Sin esto, `getTSConfigPaths` devuelve
  null y el CLI se cae. Ver §6.2 y §6.3.
- **`src/` se copia al runner.** Payload CLI carga `payload.config.ts`
  directamente vía `tsx` en runtime para resolver collections, globals y
  paths (`@payload-config`, `@cms/types`). El `.next` compilado no le basta.
- **`CMD` ejecuta migrate antes de `next start`.** Cada deploy aplica las
  migraciones pendientes de forma idempotente. Si la base está al día,
  `payload migrate` no hace nada y exit 0. Si hay nuevas, las aplica antes
  de que Next escuche el puerto. El `&&` corta la cadena si falla — mejor
  caer ahora que arrancar con DB inconsistente.

### 4.2 Variables de entorno del CMS

| Variable | Tipo | Notas |
|---|---|---|
| `DATABASE_URI` | secret | `postgresql://user:pass@<host-postgres-interno>:5432/<db>` |
| `PAYLOAD_SECRET` | secret | String aleatorio ≥32 chars. `openssl rand -hex 32` |
| `NEXT_PUBLIC_SERVER_URL` | público | `https://cms.dominio.com` |
| `SMTP_HOST` | público | p. ej. `smtp-relay.brevo.com` |
| `SMTP_PORT` | público | p. ej. `587` |
| `SMTP_USER` | secret | login SMTP |
| `SMTP_PASS` | secret | password SMTP |
| `SMTP_FROM` | público | `no-reply@dominio.com` (dominio autenticado en el proveedor) |
| `SMTP_FROM_NAME` | público | nombre legible del remitente |
| `CONTACT_EMAIL_TO` | público | destinatario del formulario |
| `CONTACT_EMAIL_BCC` | público | opcional |
| `TURNSTILE_SECRET` | secret | server-side de Cloudflare Turnstile |
| `REVALIDATE_WEBHOOK_URL` | público | URL del rebuild hook de Easypanel del web |
| `REVALIDATE_WEBHOOK_SECRET` | secret | clave del webhook (validación shared-secret) |

Notas operativas:

- Si tu dominio de email no está autenticado (SPF + DKIM + DMARC) en el
  proveedor SMTP, los primeros envíos pueden rechazarse silenciosamente.
- Cambiar un env var del CMS requiere **redeploy del contenedor** (Next.js
  cachea las env vars al arrancar, no hay hot reload).

### 4.3 Generación de migraciones (CRÍTICO en Payload 3)

En Payload 3 con Postgres, **el schema no se sincroniza automáticamente en
producción**. La opción `push: true` del `postgresAdapter` solo está activa
en dev. En producción tienes que aplicar migraciones explícitas.

**Flujo correcto:**

1. **En local**, con la base de desarrollo apuntando a tu Postgres dockerizado:

   ```bash
   cd apps/cms
   pnpm payload migrate:create
   ```

   Esto genera `apps/cms/src/migrations/<timestamp>.ts` + `.json` + actualiza
   `index.ts`. **Commit los tres archivos.**

2. **En el primer deploy a producción**: la DB está vacía. El `CMD` del
   Dockerfile ejecuta `pnpm payload migrate`, que detecta que no hay nada
   aplicado y corre la migración inicial entera (crea todas las tablas).

3. **En deploys posteriores**: cada cambio de schema en local sigue el mismo
   ciclo (`migrate:create` → commit → push → redeploy aplica la nueva).

**No usar `migrate:fresh` en producción** salvo para una inicialización
manual deliberada de una DB ya vacía. `migrate:fresh` borra todo y recrea
desde cero, sin pasar por migraciones — perderás contenido.

**Sanity check antes del primer deploy:** abre el `.ts` generado y verifica
que el bloque `up` tiene `CREATE TABLE` para todas tus collections (`users`,
`users_roles`, `users_sessions`, articles, etc.) + globals + las internas
de Payload (`payload_migrations`, `payload_preferences`, ...).

### 4.4 Dominio y HTTPS

En el panel del servicio CMS → pestaña Dominios → añadir `cms.tudominio.com`.
Apuntar el DNS a la IP del VPS (`A` record). Easypanel saca el cert
Let's Encrypt automáticamente. Mismo flujo para el web.

---

## 5. Configuración del Web

### 5.1 Dockerfile

Referencia: `apps/web/Dockerfile` + `apps/web/Caddyfile`.

Multi-stage:

- **Builder** (`node:22-slim`): instala deps con `pnpm --filter web...`,
  inyecta variables como ARG/ENV, compila a `dist/`.
- **Runner** (`caddy:2-alpine`): copia `dist/` a `/srv`, sirve estáticos
  en `:3000`. Sin Node en runtime.

Patrón base:

```dockerfile
FROM node:22-slim AS builder
RUN corepack enable && corepack prepare pnpm@9.15.0 --activate
WORKDIR /app

COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/web/package.json ./apps/web/
COPY apps/cms/package.json ./apps/cms/
COPY tsconfig.base.json ./

RUN pnpm install --frozen-lockfile --filter web...
COPY apps/web ./apps/web

WORKDIR /app/apps/web

# Build args. Astro hornea PUBLIC_* en el HTML. PAYLOAD_API_* se consumen
# en getStaticPaths (build-time fetch al CMS).
ARG PUBLIC_SITE_URL
ARG PUBLIC_CMS_URL
ARG PUBLIC_GA4_ID
ARG PUBLIC_TURNSTILE_SITEKEY
ARG PAYLOAD_API_URL
ARG PAYLOAD_API_TOKEN
ENV PUBLIC_SITE_URL=${PUBLIC_SITE_URL}
ENV PUBLIC_CMS_URL=${PUBLIC_CMS_URL}
ENV PUBLIC_GA4_ID=${PUBLIC_GA4_ID}
ENV PUBLIC_TURNSTILE_SITEKEY=${PUBLIC_TURNSTILE_SITEKEY}
ENV PAYLOAD_API_URL=${PAYLOAD_API_URL}
ENV PAYLOAD_API_TOKEN=${PAYLOAD_API_TOKEN}

RUN pnpm run build

FROM caddy:2-alpine AS runner
COPY apps/web/Caddyfile /etc/caddy/Caddyfile
COPY --from=builder /app/apps/web/dist /srv
EXPOSE 3000
CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]
```

`Caddyfile` mínimo, en `apps/web/Caddyfile`:

```caddyfile
{
    admin off
    auto_https off
    persist_config off
}

:3000 {
    root * /srv
    encode gzip zstd
    file_server

    handle_errors {
        @404 expression `{err.status_code} == 404`
        handle @404 {
            rewrite * /404.html
            file_server
        }
    }
}
```

Decisiones intencionales:

- **`auto_https off`**: Easypanel pone su propio Caddy de borde delante
  con TLS. El Caddy interno solo habla HTTP plano.
- **404 personalizado preservando status code**: Astro genera `dist/404.html`
  en la raíz (no `dist/404/index.html`). El `handle_errors` reescribe y
  sirve, pero Caddy mantiene el status 404 — los crawlers reciben el código
  correcto, no un soft-404.
- **`file_server` sin más**: ya entiende directorios y sirve `index.html`
  automáticamente, así que `/filosofia/` → `/srv/filosofia/index.html` sin
  configuración adicional.

### 5.2 Variables críticas del Web

| Variable | Tipo | Notas |
|---|---|---|
| `PUBLIC_SITE_URL` | público | `https://dominio.com`. Sin www, sin trailing slash. Astro hornea este valor en canonicals, og:url y sitemap. La versión www se redirige vía Caddy a la canónica sin-www (ver §6.13). |
| `PUBLIC_CMS_URL` | público | URL pública del CMS desde el navegador (formulario de contacto hace POST a esta). |
| `PUBLIC_GA4_ID` | público | ID de GA4. Opcional — vacío desactiva analytics. |
| `PUBLIC_TURNSTILE_SITEKEY` | público | sitekey del widget anti-spam. |
| `PAYLOAD_API_URL` | público | URL, no secreto. **Solo el origin, sin `/api`. Ver §6.6.** |
| `PAYLOAD_API_TOKEN` | público (opcional) | Vacío por defecto. Solo necesario si quieres leer borradores o si las collections tienen `access.read` restrictivo. Si lo usas, ahí sí es secret. |

### 5.3 Puerto del proxy en Easypanel

> ⚠️ **El default de Easypanel es proxar al puerto 80 del contenedor.**
> Caddy del web escucha en `:3000`. En la pestaña Dominios del servicio
> hay que cambiar el puerto del proxy a `3000`, o no responderá nada.

Ver §6.7.

---

## 6. Trampas de Easypanel (lecciones aprendidas)

### 6.1 Build path duplicado en monorepo

**Síntoma:**
```
lstat /etc/easypanel/projects/<proyecto>/<servicio>/code/apps/cms/apps/cms/Dockerfile:
no such file or directory
```

**Causa:** En el servicio Easypanel pusiste:
- Build path: `/apps/cms`
- Dockerfile path: `apps/cms/Dockerfile`

Easypanel los **concatena**. Resultado: busca `apps/cms/apps/cms/Dockerfile`.

**Solución:**
- Build path: `/` (raíz del repo)
- Dockerfile path: `apps/cms/Dockerfile`

Esto vale tanto para CMS como para web. El Dockerfile siempre se referencia
desde la raíz del repo, no desde la carpeta de la app, porque el `COPY` del
builder necesita acceder a `pnpm-lock.yaml` y los manifests del workspace.

### 6.2 `tsconfig.base.json` no copiado al builder

**Síntoma:**
```
error TS5083: Cannot read file '/app/tsconfig.base.json'
```
Durante `pnpm run build` del CMS o `astro build` del web.

**Causa:** `apps/cms/tsconfig.json` y `apps/web/tsconfig.json` declaran
`"extends": "../../tsconfig.base.json"`. Si el Dockerfile no copia ese
archivo, TypeScript no puede resolver la cadena.

**Solución:**
```dockerfile
COPY tsconfig.base.json ./
```
En el builder, antes del `pnpm run build`.

### 6.3 Source files faltantes en el runner del CMS

**Síntoma:** El CMS arranca, Next sirve el admin, pero al ejecutar
`pnpm payload migrate` (ya sea desde el `CMD` o manualmente por SSH):

```
TypeError: Cannot read properties of null (reading 'config')
```

O variantes con `getTSConfigPaths` devolviendo null.

**Causa:** El CLI de Payload (`payload migrate`, `migrate:fresh`,
`generate:types`, etc.) **carga `payload.config.ts` en runtime** vía `tsx`
para resolver collections, globals y los path aliases (`@payload-config`,
`@cms/types`). El `.next` compilado **no le sirve** — necesita el código
fuente TypeScript en disco.

**Solución:** copiar al runner:
```dockerfile
COPY --from=builder /app/apps/cms/src ./apps/cms/src
COPY --from=builder /app/tsconfig.base.json ./
COPY --from=builder /app/apps/cms/tsconfig.json ./apps/cms/
```

No es desperdicio: son archivos pequeños y el CLI los necesita.

### 6.4 Migraciones de Payload 3 no automáticas en prod

**Síntoma:** Primer deploy verde, abres `/admin`, te pide crear usuario,
le das a "Create User":
```
relation "users" does not exist
```
o variantes con otras tablas.

**Causa:** En Payload 3 con `postgresAdapter`, la opción `push: true`
(sincroniza schema sin migraciones, modo dev) **se desactiva en
producción**. La DB recién creada está vacía y nadie crea las tablas.

**Solución completa:**

1. En local, contra tu Postgres dev:
   ```bash
   cd apps/cms
   pnpm payload migrate:create
   ```
2. Verifica que el `.ts` generado tiene `CREATE TABLE` para todas tus
   collections + globals + tablas internas de Payload.
3. Commit los tres archivos generados (`<timestamp>.ts`, `.json`, y
   `index.ts` actualizado).
4. Push y redeploy.
5. El `CMD` del Dockerfile (`pnpm payload migrate && pnpm exec next start`)
   las aplica idempotentemente en cada arranque.

Ver §4.3 para más detalle.

### 6.5 Easypanel no rebuildea con cambios de env vars

**Síntoma:** cambias `PAYLOAD_API_URL` en el panel, das a Implementar,
el build "tarda poco" sospechosamente, el sitio sigue con el comportamiento
viejo.

**Causa:** Easypanel reutiliza la imagen Docker cacheada si detecta que
el código del repo no ha cambiado. Cambios solo de env vars no invalidan
la cache.

**Soluciones (cualquiera vale):**

- **A**: borrar la imagen manualmente por SSH al VPS:
  ```bash
  docker image rm easypanel/<proyecto>/<servicio>:latest
  ```
- **B**: commit trivial al repo (un comentario, un espacio) y push,
  para forzar invalidación de cache.
- **C**: si la UI de tu versión de Easypanel tiene el toggle "Implementar
  sin caché", úsalo.

### 6.6 `PAYLOAD_API_URL` con `/api` (LA QUE MÁS TIEMPO COSTÓ)

**Síntoma:**
```
[payload] 404 Not Found en /api/articles?...
[payload] 404 Not Found en /api/globals/site-settings
...
```
Durante el build del web. Confunde porque las mismas URLs funcionan con
`curl` o desde un `docker run node:22-slim` arbitrario.

**Causa:** `apps/web/src/lib/payload.ts` concatena
```ts
const url = `${PAYLOAD_API_URL}${path}`;
```
donde **todas las funciones pasan paths que empiezan por `/api/...`**:
```ts
fetchPayload('/api/articles?...');
fetchPayload('/api/globals/site-settings');
fetchPayload('/api/legal-pages?...');
```

Si `PAYLOAD_API_URL=https://cms.dominio.com/api`, la URL final queda:
```
https://cms.dominio.com/api  +  /api/articles  =  /api/api/articles  → 404
```

Por eso solo falla en el build (donde se usa la env var); las pruebas
externas con `curl` pegan a `https://cms.dominio.com/api/articles` directo
y funcionan.

**Solución:** `PAYLOAD_API_URL` debe ser **solo el origin**:
```
PAYLOAD_API_URL=https://cms.dominio.com
```
Sin `/api`. Sin trailing slash.

**Fix defensivo aplicado en código** (`apps/web/src/lib/payload.ts`) para
que la trampa no se repita:
```ts
const PAYLOAD_BASE = (PAYLOAD_API_URL ?? '')
  .replace(/\/api\/?$/, '')
  .replace(/\/$/, '');

// usar PAYLOAD_BASE en la concatenación, no PAYLOAD_API_URL crudo.
```

Tolera los cuatro formatos (`origin`, `origin/`, `origin/api`, `origin/api/`)
y elimina la clase entera de bug.

### 6.7 Puerto del proxy de Easypanel

**Síntoma:** contenedor en verde, build OK, abres la URL temporal de
Easypanel → "Service is not reachable" o pantalla en blanco.

**Causa:** Easypanel proxa por defecto al **puerto 80** del contenedor.
Tu Caddy interno escucha en `:3000` (porque el puerto 80 requiere root,
y `caddy:2-alpine` corre como usuario sin privilegios).

**Solución:** en el servicio web → pestaña Dominios → editar el dominio
(o el dominio temporal) → cambiar **Puerto del proxy** a `3000`.

Lo mismo aplica al CMS: si tu `EXPOSE` y `next start` apuntan a `:3000`,
configura el proxy a `:3000`.

### 6.8 Easypanel sin tipo "Static Site"

Algunas versiones de Easypanel ofrecen el tipo "Static Site" como servicio
de primera clase. Otras no — solo tienen "Aplicación" con métodos de
compilación (Dockerfile, Buildpacks, Nixpacks, Railpack).

**Si tu versión no tiene Static Site dedicado:** usa "Aplicación" + método
"Dockerfile" + el patrón builder Node / runner Caddy descrito en §5.1. No
intentes meter Astro estático en Nixpacks, hay catálogos con versiones
viejas de pnpm/npm que rompen el build.

### 6.9 Easypanel sin consola web (plan gratuito / self-hosted básico)

Algunas instalaciones no exponen consola web del contenedor. Sin ella,
**necesitas SSH al VPS** para inspeccionar o ejecutar comandos:
```bash
ssh root@<ip-vps>
docker ps | grep cms
docker exec -it <nombre-contenedor> sh
```
Después dentro:
```bash
cd /app/apps/cms
pnpm payload migrate
```

Consigue acceso SSH como prerequisito del despliegue, no como afterthought.

### 6.10 Volumen persistente para media del CMS (CRÍTICO)

**Síntoma:** subes imágenes al admin del CMS, las ves correctamente, pero
al siguiente rebuild del servicio CMS (por cualquier razón: nueva env var,
fix de código, etc.) la galería aparece vacía. Las filas de la tabla
`media` siguen en Postgres pero apuntan a archivos que ya no existen.

**Causa:** por defecto los contenedores Docker tienen filesystem efímero.
Payload escribe los archivos en `/app/apps/cms/media` (relativo al CWD del
proceso, configurado vía `staticDir: 'media'` en la collection Media). Sin
un volumen persistente montado en esa ruta, cada rebuild borra los archivos.

**Solución:** en Easypanel → servicio CMS → pestaña Almacenamiento →
Agregar montaje de volumen:

- Tipo: Volume (named volume Docker)
- Nombre: `<proyecto>-cms-media` (o similar identificable)
- Mount path: `/app/apps/cms/media`

Después, Implementar. El volumen monta vacío sobre esa ruta y todo lo que
Payload escriba a partir de ahí persiste entre rebuilds.

Verificación tras montar:

```bash
docker exec <id-cms> mount | grep media
# Esperado: línea con /app/apps/cms/media

docker inspect <id-cms> --format '{{range .Mounts}}{{.Type}} {{.Name}} -> {{.Destination}}{{println}}{{end}}'
# Esperado: volume <nombre> -> /app/apps/cms/media
```

**ORDEN IMPORTANTE:** configurar el volumen ANTES de subir imágenes al
admin, no después. Si montas un volumen sobre una carpeta que ya tiene
archivos, esos archivos quedan ocultos (no borrados pero inaccesibles
desde dentro del contenedor).

Postgres ya persiste por separado (es otro servicio con su propio
almacenamiento), solo afecta a los uploads del CMS.

### 6.11 CORS de Payload bloquea formulario en producción

**Síntoma:** el formulario de contacto funciona en local pero en
producción dispara error CORS al pulsar Enviar:

```
Access to fetch at 'https://cms.dominio.com/api/contact-submissions' from
origin 'https://dominio.com' has been blocked by CORS policy.
```

**Causa:** el array `allowedOrigins` en `apps/cms/src/payload.config.ts`
(referenciado por las propiedades `cors` y `csrf`) solo incluye localhost
y, opcionalmente, una variante de producción. Si la web canónica está en
otro dominio (sin-www vs con-www, o solo tenías www y ahora hay sin-www,
etc.), el navegador bloquea por seguridad.

**Solución:** ampliar `allowedOrigins` en `payload.config.ts` para incluir
todos los dominios desde los que el navegador hará peticiones al CMS:

```ts
const allowedOrigins = [
  'http://localhost:4321',                          // web dev
  'https://dominio.com',                            // canónico
  'https://www.dominio.com',                        // variante www
  'https://<proyecto>-web.<panel>.easypanel.host',  // URL temporal
];
```

El mismo array vale para `cors` y `csrf` — una sola fuente de verdad evita
inconsistencias.

Tras editar, push + redeploy del CMS para aplicar.

### 6.12 Endpoint `/api/media/file/*` no acepta HEAD

**Síntoma:** durante diagnóstico, `curl -I https://cms.dominio.com/api/media/file/imagen.webp`
devuelve 404, pero el admin muestra la imagen correctamente y la web
pública la sirve sin problema. Crees que hay un bug del endpoint y empiezas
a buscarlo donde no está.

**Causa:** el route handler `[...slug]` de Payload sobre Next.js App Router
NO implementa el método HEAD. Solo GET. `curl -I` envía HEAD por defecto →
Next devuelve 404 porque no hay handler para ese método en esa ruta.

**Solución diagnóstica** (no fix, ajustar la verificación): para probar
endpoints de archivos de Payload usa GET explícitamente:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://cms.dominio.com/api/media/file/imagen.webp"
# Esperado: 200
```

Los navegadores y Astro siempre hacen GET para imágenes, así que en
producción real nunca es problema. Solo afecta a diagnóstico con curl.

### 6.13 Redirect www → sin-www (o viceversa) en Caddy interno

**Síntoma:** después de configurar dominios en Easypanel, tanto
`www.dominio.com` como `dominio.com` responden 200 con el mismo contenido.
Google ve contenido duplicado, los canonicals del HTML apuntan a uno y la
URL del navegador puede estar en el otro.

**Causa:** Easypanel apunta los dos dominios al mismo backend, pero no
fuerza redirect. El Caddy de borde de Easypanel sirve ambos sin discriminar.

**Solución:** añadir al Caddyfile interno del web (`apps/web/Caddyfile`) un
bloque que detecte el host www y redirija a la versión canónica sin-www (o
al revés, según hayas decidido):

```caddyfile
:3000 {
    root * /srv

    @www host www.dominio.com
    redir @www https://dominio.com{uri} 301

    # (resto de redirecciones 301 si las hay)
    redir /url-antigua /url-nueva permanent

    encode gzip zstd
    file_server

    handle_errors { ... }
}
```

El orden importa: el redirect del www tiene que ir antes del `file_server`
para que se evalúe primero. Y antes de las redirecciones 301 individuales
para no encadenar saltos innecesarios.

Verificación:

```bash
curl -I https://www.dominio.com/
# Esperado: HTTP/2 301 + location: https://dominio.com/
```

**Coherencia:** `PUBLIC_SITE_URL` en build args del web tiene que ser la
versión canónica (sin-www en este caso). Si no, los canonicals horneados
en HTML salen mal y mandas a Google señales contradictorias.

### 6.14 Rebuild-on-publish real: el deploy hook NO sirve, usar la API tRPC `forceRebuild`

**Contexto:** la web es estática (Astro hornea el contenido del CMS en HTML
durante el build). Publicar contenido en el CMS no aparece hasta relanzar el
build del web (ver §2). En aquel primer proyecto esto quedó como TODO y **nunca se cableó** — el
plumbing de `REVALIDATE_WEBHOOK_*` se reservó pero el código que lo consume no
se escribió. **En luismilascurain.com se resolvió de verdad** (botón "Publicar
cambios en la web" en el admin de Payload → server-side → Easypanel). Dos
trampas, ambas costaron tiempo:

**Trampa A — el deploy hook simple hace build CON caché (la clave).**
Easypanel expone por servicio un *deploy webhook* (`/api/deploy/<token>`, en el
panel: servicio → "Activación de implementación"). Pegarle un POST dispara el
**"Deploy" normal, que reusa la imagen Docker cacheada y se salta el build si el
repo no cambió** (§6.5). Como publicar no cambia git, la capa `RUN astro build`
—donde Astro hace fetch al CMS y hornea el HTML— queda cacheada: el deploy dura
segundos y **sirve la versión vieja**. El deploy hook simple **NO sirve para
rebuild-on-publish de contenido.**

La solución es un build **SIN caché**, que es lo que hace "Forzar
reconstrucción" del panel: la **API tRPC de Easypanel**, mutación
`services.app.deployService` con `forceRebuild: true`.

```
POST {PANEL_URL}/api/trpc/services.app.deployService
Authorization: Bearer <EASYPANEL_API_TOKEN>
Content-Type: application/json

{"json":{"projectName":"<proyecto>","serviceName":"<servicio-web>","forceRebuild":true}}
```

El `{"json": …}` es el formato del data transformer de tRPC. La respuesta de
éxito es `{"result":{"data":…}}`; en error, `{"error":{…,"data":{"code":"UNAUTHORIZED|BAD_REQUEST",…}}}`
(puede venir envuelto en `.json`). **No fiarse solo del status HTTP**: parsear
el body y tratar la presencia de `error` como fallo. Build sin caché ≈ 19s vs
los segundos del deploy cacheado — esa diferencia de tiempo es la señal de que
está rehornenando.

**Trampa B — desde un contenedor del propio VPS, NO uses la IP ni IPs internas.**
La URL del deploy hook que muestra el panel es la IP **pública** del host
(`http://<IP>:3000/api/deploy/<token>`). Llamarla desde un contenedor del mismo
VPS falla por **NAT hairpin** (`fetch failed`, `err.cause.code` = ETIMEDOUT /
EHOSTUNREACH). Y bajo **Swarm** (lo que usa Easypanel) **las IPs internas
tampoco funcionan**: el gateway de la red del proyecto (10.x) y `docker_gwbridge`
(172.18.0.1) dan ECONNREFUSED/timeout — el panel no es alcanzable por ahí desde
el servicio. **Lo que funciona: el dominio HTTPS del panel** (`https://panel.tu-dominio`),
que sale por el proxy de borde (443) y enruta. Regla: **para hablar con la API
de Easypanel desde un contenedor del VPS, usa el dominio HTTPS del panel, no la
IP ni gateways internos.**

**Variables (env del CMS, lado servidor — el token NUNCA llega al navegador):**
`EASYPANEL_API_URL` (`https://panel.tu-dominio`), `EASYPANEL_API_TOKEN` (secret,
Settings → API), `EASYPANEL_PROJECT`, `EASYPANEL_WEB_SERVICE`. Sustituyen al
`REVALIDATE_WEBHOOK_*` reservado en §4.2, que era para el deploy hook simple.

**Seguridad (relevante para webs de cliente):** el API token de Easypanel es de
**cuenta completa** (controla todo el panel), no scoped a un servicio como el
deploy hook. Para la propia web es aceptable; al dárselo a un CMS de cliente,
**aislar** (panel/instancia de Easypanel por cliente) o interponer un **relay
scoped** (endpoint mínimo que guarda el token y solo permite `forceRebuild` de
un servicio concreto; el CMS del cliente solo conoce un shared-secret).

**Patrón del botón en Payload 3 (admin):** componente client en
`admin.components.beforeDashboard` (regenerar `importMap` y commitearlo, §crítico
en Payload 3.84) + endpoint propio (`POST /api/rebuild`) con guard `req.user`
que hace la llamada tRPC server-side. El disparo automático (`afterChange`) se
dejó como evolución futura (sin generación automática de contenido, hoy sería
código especulativo).

---

> **§6.15–§6.18 — trampas del debut Astro 7 + Payload 3 (laskurain.es, 2026-07-07).**
> Las cuatro salieron en el primer despliegue de la combinación completa. Las dos
> primeras (build) y las dos siguientes (runtime + método de verificación) no las
> caza el `docker build` a secas: exigen reproducir el arranque real y probar por
> navegador. Regla transversal: **antes del primer deploy, reproduce en local el
> build Y el arranque del contenedor** (`docker build` + `docker run` del CMD real).

### 6.15 El marcador `dev` de `payload_migrations` cuelga `payload migrate` sin TTY

Aplica cuando **el contenido de producción se siembra restaurando un `pg_dump` de
la base de desarrollo** (en vez de correr el seed en prod).

**Síntoma:** el contenedor del CMS entra en crash-loop en el primer arranque; el
`CMD` (`payload migrate && next start`) nunca llega a `next start`. En un arranque
interactivo se ve el prompt:

```
? It looks like you've run Payload in dev mode, meaning you've dynamically pushed
  changes to your database. If you'd like to run migrations, data loss will occur.
  Would you like to proceed? › (y/N)
```

**Causa:** una base que ha corrido en **dev con `push`** tiene, además de las
migraciones reales, una fila especial **`dev`** (batch `-1`) en `payload_migrations`.
`payload migrate` la detecta y **pregunta por stdin** antes de continuar. Sin TTY
(contenedor) se queda esperando y, al cerrarse stdin, sale con **exit 1** → el `&&`
corta → el CMS no arranca. (Es distinto del cuelgue de `payload run <script>` en
getPayload; este es el prompt de detección de dev-push.)

**Solución:** el dump de producción debe llevar `payload_migrations` **solo con las
migraciones reales, sin la fila `dev`**. Al generar el paquete: restaurar el dump en
una BD scratch → `DELETE FROM payload_migrations WHERE name='dev';` → **re-dump**.
Con la fila fuera, `payload migrate` es no-op (`Reading migration files… Done.`,
exit 0) y `next start` arranca. Verificado restaurando el dump limpio y corriendo el
`CMD` real del contenedor.

### 6.16 El type-check de `next build` (prod) es MÁS ESTRICTO que `next dev`

**Síntoma:** el primer `next build` de producción del CMS muere en el type-check con
errores que **en local con `next dev` nunca aparecieron**, p. ej.:

```
Type error: Argument of type 'PayloadRequest' is not assignable to parameter of type 'Request'.
  Types of property 'cache' are incompatible.
```

**Causa:** `next dev` **no ejecuta el type-check completo del proyecto**; los errores
de tipo quedan **latentes** hasta el primer `next build`. Además `next build` corta
mostrando **un solo error cada vez** (el primero): al arreglarlo aparece el siguiente.

**Solución:** arreglar el **tipado de verdad** (nunca `as any` ni
`typescript.ignoreBuildErrors` — dejar el check activo). Ejemplos reales que salieron:
un helper `getIp(req: Request)` que recibía un `PayloadRequest` → tiparlo por lo que
usa (`{ headers: Headers }`); un mapa de ids de Media como `number | string` cuando en
Postgres `Media.id` es `number`; ids de fila de **array de Payload** tratados como
`number` cuando **son `string`**; y un loop que llamaba `updateGlobal` con `slug: string`
(de `Object.entries`), que degrada el tipo de `data` → tipar las claves como los slugs
literales. **Reproduce el build de producción en local antes del primer deploy:**

```bash
docker build -f apps/cms/Dockerfile .   # corre el next build real (clean install + type-check)
```

### 6.17 La config de Payload NO puede importar estáticamente nada de otro workspace (RUNTIME)

**Síntoma (runtime, NO build):** el build pasa, pero el contenedor del CMS entra en
crash-loop al arrancar:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/app/apps/web/src/content/home'
  (al ejecutar `payload migrate` del CMD → carga payload.config → endpoints/seed.ts)
```

**Causa:** la imagen es **multi-stage** y al runner solo se copia `apps/cms` (no
`apps/web`). Un endpoint (aquí el de seed) importaba **estáticamente** contenido de
`apps/web/src/content/*`. Esos módulos existen en la etapa *builder* pero **no en la
imagen final**. Como `payload migrate` **carga la config vía tsx** y la config importa
los endpoints, resuelve el import estático → falta el módulo → crash-loop. El
`next build` NO lo detecta: en el builder `apps/web` sí está.

**Solución (sin copiar `apps/web` a la imagen):** convertir esos imports en
**dinámicos DENTRO del handler** (`await import('...')`), de modo que solo se resuelvan
al invocar el endpoint (deshabilitado en prod sin su secreto). Los tipos se traen con
`typeof import(...)` (construcción de solo-tipo, se borra en runtime). **Regla general:
la config de Payload y todo lo que ella importa (endpoints, hooks, collections, globals)
no puede importar estáticamente nada de otro workspace.** Comprobar con
`grep -rE "from ['\"].*<otro-workspace>/src" apps/cms/src`.

**Verificación (runtime, no solo build):** reproducir el **arranque real** del contenedor:

```bash
docker build -f apps/cms/Dockerfile -t cms:verify .
docker run -d --name cms-rt -p 3001:3000 \
  -e DATABASE_URI="postgres://user:pass@host.docker.internal:5432/<db-restaurada>" \
  -e PAYLOAD_SECRET="..." -e PORT=3000 cms:verify
docker logs cms-rt   # migrate "Done." + next "Ready", SIN ERR_MODULE_NOT_FOUND
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3001/admin   # → 200
```

### 6.18 Verifica los formularios NAVEGADOR→endpoint, no solo curl→endpoint

**Síntoma:** un formulario devuelve **400** desde el navegador con todos los campos
aparentemente correctos, pero en las pruebas por `curl` **nunca falla**.

**Causa:** el `curl` construye el JSON **a mano y "bien tipado"**, así que nunca
ejercita las coerciones que hace el DOM al serializar un formulario real. El caso que
lo destapó: un **checkbox nativo marcado**, vía `Object.fromEntries(new FormData(form))`,
produce `consent: "on"` (string), no `true`; el endpoint validaba `consent === true` →
`"on" !== true` → 400. `curl` mandaba `consent: true` y lo ocultaba. (Otras coerciones
que esconde el curl: números como strings, campos vacíos vs ausentes, y el **nombre del
campo del widget** — p. ej. Turnstile inyecta `cf-turnstile-response` salvo que fijes
`data-response-field-name`.)

**Solución:** normalizar en el cliente antes de enviar (p. ej. sobreescribir
`data.consent = el?.checked === true`) y, sobre todo, **verificar cada formulario con un
envío de NAVEGADOR real** contra el endpoint (con claves de test de Turnstile),
inspeccionando status + body recibido + fila guardada. El `curl→endpoint` vale para el
pipeline del servidor, pero **no sustituye** al `navegador→endpoint` para esta clase de bug.

---

### 6.19 `import.meta.env` sin `PUBLIC_` se hornea en el build — y el optimizador borra el código que dependía de él (y con él, sus dependencias)

**Síntoma:** en producción, tres fallos aparentemente inconexos y **mudos**: el captcha
no bloquea nada (los formularios aceptan envíos sin resolverlo), el aviso por email no
llega **y no deja ni una línea de log**, y el contenedor arranca perfectamente. En local
todo verde. Horas de silencio.

**Causa — tres capas apiladas, cada una escondiendo a la siguiente:**

1. **`import.meta.env.LO_QUE_SEA`, para variables que NO empiezan por `PUBLIC_`, lo
   SUSTITUYE VITE EN EL BUILD** por el valor literal de ese momento. No es una lectura:
   es una constante horneada. Si el secreto vive (bien) solo en el **runtime** del
   contenedor, en el bundle queda `undefined`.
2. **Y como queda constante, el optimizador razona sobre ella y BORRA CÓDIGO.** Lo que
   se compiló de verdad:
   ```js
   var verifyTurnstile = async (token) => { const secret = undefined;
                                            if (!secret) return true; ... }  // → return true
   var sendNotification = async (subject, html) => {};   // ← la función ENTERA, vacía
   ```
   El captcha fallaba ABIERTO y el email ni se intentaba. El `console.error` del `catch`
   tampoco sobrevivió: por eso no había logs — no es que el error se tragara, es que
   **no quedaba código**.
3. **Con el código borrado, sus dependencias desaparecen de la imagen sin que ningún
   build lo detecte.** Nadie importaba `nodemailer`, así que nadie notó que **jamás se
   había declarado** en `dependencies` del workspace. Al arreglar las capas 1 y 2, el
   fallo mutó a `ERR_MODULE_NOT_FOUND: Cannot find package 'nodemailer'`.

**Y una cuarta, de propina:** al declararla, seguía sin resolver **dentro de la imagen**.
El Dockerfile aplanaba el workspace con dos `COPY` al mismo destino:
```dockerfile
COPY --from=builder /app/node_modules           ./node_modules
COPY --from=builder /app/apps/web/node_modules  ./node_modules   # ← rompe los symlinks
```
Los symlinks de pnpm son **relativos**: `apps/web/node_modules/x → ../../../node_modules/.pnpm/…`
resuelve bien desde `/app/apps/web/node_modules/`, pero desde `/app/node_modules/` apunta
a `/node_modules/.pnpm/…`, **fuera del contenedor**. No se notaba porque el bundle resuelve
Astro y compañía por rutas del store; lo único que pasa por el symlink es un
`import('paquete')` **resuelto en RUNTIME** — y el único que había estaba dentro de la
función que el optimizador había borrado.

**Solución — tres reglas, no una:**

1. **Los secretos se leen SIEMPRE en runtime.** Un helper de una línea, y prohibido
   `import.meta.env.NOMBRE` literal para nada que no sea `PUBLIC_`:
   ```js
   export const env = (key) =>
     process.env?.[key] || import.meta.env?.[key] || undefined   // clave DINÁMICA: Vite no la inlinea
   ```
   Y **falla cerrado**: `if (!secret) return true` no es una comodidad de desarrollo, es
   una puerta abierta que además parece cerrada.
2. **Toda función de efecto loguea éxito Y fallo, con causa.** Cada `return` mudo es una
   hora de depuración a ciegas. Si el aviso no sale, en el log tiene que constar por qué:
   qué variable falta, qué campo está vacío, qué dijo el SMTP.
3. **El smoke test de la imagen Docker debe EJERCITAR LOS CAMINOS DE EFECTO, no solo el
   arranque.** «El contenedor levanta y sirve HTML» no prueba nada de esto. Hay que hacer
   el POST real contra la imagen y comprobar que el email **se intenta y se ejecuta** —
   con un SMTP de juguete basta (≈40 líneas sin dependencias, aceptando y volcando el
   mensaje). Verificación mínima, con build SIN secretos y runtime CON ellos:
   ```
   POST sin captcha  → 403 + log de rechazo
   POST con captcha  → 200 + "aviso enviado … messageId=…" + el SMTP lo recibe
   ```

**Quinta capa, y la más traicionera: los fallbacks que fabrican valores INVÁLIDOS.**
Con todo lo anterior arreglado, el aviso seguía sin llegar — ahora el proveedor lo
rechazaba: «the sender you used a6159a001@smtp-brevo.com is not valid». El código hacía
`from: env('BREVO_FROM') || user`, y ese `|| user` —que parecía una red de seguridad—
firmaba el mensaje con el **login SMTP**, que nunca es un remitente dado de alta.
Mientras tanto el log decía «aviso enviado». Un valor por defecto solo es una red de
seguridad si el valor es VÁLIDO; si no, es un fallo disfrazado de éxito, y encima
desplaza la culpa al proveedor. Reglas que deja: **variable obligatoria sin fallback**
cuando no existe un valor por defecto correcto, y **el log de éxito imprime los datos
que determinan si el efecto va a funcionar** (aquí, el `from` usado) — no solo «hecho».
Esta capa costó una iteración entera que no habría existido con el `from` en el log.

**Regla general que deja este caso:** cuando un fallo de producción no deja logs,
sospecha primero de que **el código no exista** en el artefacto desplegado, antes que de
la lógica. Y desconfía de los verdes: aquí hubo tres comprobaciones en verde —el build,
el arranque del contenedor y la suite de formularios— y las tres eran ciegas al fallo.

---

## 7. Checklist rápido para un proyecto nuevo

Para el yo del futuro que ya conoce este manual. Sin explicaciones, solo
el orden.

1. Repo monorepo creado, `apps/cms/` y `apps/web/` funcionando en local.
2. Generar migraciones iniciales:
   ```bash
   cd apps/cms && pnpm payload migrate:create
   ```
3. Verificar que el `.ts` generado tiene `CREATE TABLE` para todas las
   collections + globals + tablas internas de Payload.
3b. **Reproducir en local, antes de tocar Easypanel, el build Y el arranque del
   contenedor del CMS:** `docker build -f apps/cms/Dockerfile .` y `docker run`
   del `CMD` real (`migrate && start`) contra una BD restaurada, hasta servir
   `/admin` (200). Caza §6.15–§6.18 sin gastar ciclos del panel. Si siembras prod
   con un dump de dev, límpialo del marcador `dev` (§6.15).
4. Commit y push.
5. **En Easypanel**, crear proyecto.
6. Crear servicio **Postgres** (tipo "Postgres" del catálogo). Anotar
   credenciales internas.
7. Crear servicio **CMS** tipo "Aplicación":
   - Source: GitHub, repo, rama `main`.
   - Build method: Dockerfile.
   - Build path: `/`.
   - Dockerfile path: `apps/cms/Dockerfile`.
   - Variables de entorno: todas las de §4.2. `DATABASE_URI` apunta al
     hostname interno del servicio Postgres.
   - Dominio: añadir `cms.dominio.com` cuando el DNS esté apuntado.
   - Puerto del proxy: `3000`.
8. Implementar → verde. Acceder al admin, crear primer user admin.
9. Cargar contenido inicial: categorías, primer artículo en estado
   `published`.
10. Crear servicio **Web** tipo "Aplicación":
    - Source: mismo repo, rama `main`.
    - Build method: Dockerfile.
    - Build path: `/`.
    - Dockerfile path: `apps/web/Dockerfile`.
    - Build args: PUBLIC_* + `PAYLOAD_API_URL` **sin `/api`** + opcional
      `PAYLOAD_API_TOKEN`.
    - Dominio: `www.dominio.com` (y redirección de apex si la quieres).
    - Puerto del proxy: `3000`.
10b. En Easypanel → servicio CMS → Almacenamiento → Agregar montaje de
    volumen para `/app/apps/cms/media`. **ANTES de subir imágenes al
    admin.** Ver §6.10.
11. Implementar → verde. Verificar:
    - El sitio responde en la URL temporal.
    - Las páginas del CMS (`/escritos/<slug>`, legales) están generadas.
    - GA4 y Turnstile aparecen activos si correspondía.
12. Apuntar DNS al VPS. Esperar propagación. Easypanel saca certs Let's
    Encrypt automáticamente.
13. Configurar redirecciones 301 si vienes de un sitio anterior.
    **Van dentro del Caddyfile del propio servicio web (`apps/web/Caddyfile`),
    NO en el Caddy de borde de Easypanel** — así viajan con el repo y son
    versionables. Incluye el redirect www → sin-www (o viceversa): ver §6.13
    para el bloque exacto y el orden dentro de `:3000`.
14. (Opcional, recomendado a futuro) Implementar webhook `afterChange`
    en las collections que dispare rebuild del web en Easypanel.

---

## 8. Comandos útiles (apéndice)

Todos asumen SSH al VPS. Sustituye `<servicio>` y `<proyecto>` por los
nombres reales del panel.

**Encontrar el contenedor de un servicio:**
```bash
docker ps | grep <servicio>
```

**Entrar al contenedor:**
```bash
docker exec -it <nombre-contenedor> sh
# o si la imagen tiene bash:
docker exec -it <nombre-contenedor> bash
```

**Forzar rebuild (invalidar cache de Easypanel):**
```bash
docker image rm easypanel/<proyecto>/<servicio>:latest
# después, en el panel: Implementar
```

**Verificar conectividad interna entre dos servicios** (desde web al CMS,
por ejemplo):
```bash
docker exec <contenedor-web> wget -qO- http://<servicio-cms-interno>:3000/api/articles | head -c 200
```
Sustituye `<servicio-cms-interno>` por el hostname interno que Easypanel
da al servicio (normalmente `<proyecto>_<servicio>` o similar — mira el
nombre de la red del proyecto con `docker network inspect`).

**Aplicar migraciones manualmente** (raramente necesario porque el `CMD`
ya las aplica al arrancar):
```bash
docker exec -it <contenedor-cms> sh
cd /app/apps/cms
pnpm payload migrate
```

**Inicialización deliberada de una DB vacía** (¡destructivo, salta
migraciones!):
```bash
docker exec -it <contenedor-cms> sh
cd /app/apps/cms
pnpm payload migrate:fresh
```

**Generar nuevas migraciones (en local, contra Postgres dev):**
```bash
cd apps/cms
pnpm payload migrate:create
```

**Regenerar tipos de Payload tras cambios en collections:**
```bash
cd apps/cms
pnpm cms:types
```

**Regenerar `importMap.js` tras cambiar features de Lexical o añadir
componentes custom al admin:**
```bash
pnpm --filter cms exec payload generate:importmap
```

**Inspeccionar el Postgres del proyecto desde fuera** (útil para backup
manual o debugging):
```bash
docker exec -it <contenedor-postgres> psql -U <usuario> -d <db>
```

**Ver logs del último deploy de un servicio:**
```bash
docker logs --tail=200 -f <contenedor>
```

**Volúmenes (uploads del CMS — ver §6.10):**
```bash
# Listar volúmenes del proyecto
docker volume ls | grep <proyecto>

# Backup de un volumen a tar.gz
docker run --rm -v <nombre-volumen>:/data -v $(pwd):/backup alpine tar czf /backup/<nombre>-$(date +%F).tar.gz -C /data .

# Restaurar desde tar.gz
docker run --rm -v <nombre-volumen>:/data -v $(pwd):/backup alpine tar xzf /backup/<archivo>.tar.gz -C /data
```

---

## 9. Verificaciones post-deploy

Checklist para correr una vez todo está desplegado y con DNS apuntado.
Sustituye `dominio.com` por el dominio real.

- [ ] La home y al menos 3 rutas representativas devuelven 200 en la URL canónica.
- [ ] La versión no canónica redirige 301 a la canónica: `curl -I https://www.dominio.com/`
- [ ] Las redirecciones 301 funcionan: `curl -I https://dominio.com/<url-antigua>`
- [ ] Robots.txt apunta al sitemap canónico: `curl https://dominio.com/robots.txt`
- [ ] Sitemap accesible y con URLs canónicas: `curl https://dominio.com/sitemap-0.xml`
- [ ] Canonical en HTML coincide con dominio canónico: `curl -s https://dominio.com/ | grep canonical`
- [ ] Endpoint del CMS responde con GET: `curl -s -o /dev/null -w "%{http_code}\n" https://cms.dominio.com/api/articles`
- [ ] Formulario de contacto envía email correctamente.
- [ ] Imágenes del CMS se ven en el sitio público.
- [ ] Volumen persistente del CMS montado correctamente: `docker exec <id-cms> mount | grep media`
- [ ] Sitemap enviado y procesado en Search Console (estado "Correcto", no "No se ha podido leer").
- [ ] URLs prioritarias enviadas a indexación.

---

## Crédito

Manual generado a partir del despliegue real de un proyecto de cliente
(monorepo pnpm) en Easypanel sobre un VPS, el **2026-05-13**. Cada
trampa de la §6 corresponde a un bug que costó tiempo durante esa
sesión — el manual existe para que no vuelva a costarlo.

Actualizado el **2026-05-14** con las trampas descubiertas el segundo
día de despliegue: volumen persistente para media del CMS (§6.10),
CORS en producción bloqueando el formulario (§6.11), HEAD vs GET en el
endpoint de archivos (§6.12), redirect www → sin-www en el Caddy interno
(§6.13), robots.txt apuntando al sitemap canónico, y la nueva sección
§9 de verificaciones post-deploy.

Actualizado el **2026-07-07** con el primer despliegue de la combinación
completa **Astro 7 + Payload 3** (laskurain.es), que la deja **validada en
producción** y aporta cuatro trampas nuevas: marcador `dev` colgando
`migrate` al restaurar un dump de dev (§6.15), type-check de `next build`
más estricto que dev (§6.16), la config de Payload importando de otro
workspace y reventando en runtime (§6.17), y la verificación de formularios
por navegador vs curl (§6.18). Lección de proceso: reproducir en local el
`docker build` **y** el `docker run` del arranque real antes del primer deploy.

Autoría: [Luismi Lascurain](https://luismilascurain.com) — consultor digital independiente en Donostia — asistido
por Claude Code.
