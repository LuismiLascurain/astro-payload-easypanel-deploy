# Desplegar Astro + Payload CMS 3 en un VPS con Easypanel

Manual práctico, nacido de despliegues reales, con todas las trampas que me
encontré documentadas: monorepo pnpm, build de Payload 3 sobre Next 15,
Postgres, migraciones, volúmenes para media, CORS, dominios, SSL, el puerto
del proxy, y el rebuild-on-publish de verdad (spoiler: el deploy hook simple
no sirve — hay que usar la API tRPC con `forceRebuild`).

**→ [El manual completo](./DEPLOY-ASTRO-PAYLOAD-EASYPANEL.md)**

## Por qué existe

Astro estático + Payload autoalojado es una combinación excelente para webs
de PYME —rendimiento máximo, coste mínimo, datos en tu servidor— pero está
poco documentada en español y tiene trampas que no salen en ningún tutorial:
cada sección §6 del manual corresponde a un error real que costó horas.
Publicado para que a ti no te las cueste.

## Qué cubre

- Arquitectura: Astro (`output: static`) horneando el CMS en build + Caddy
  sirviendo `dist/` + Payload 3 + Postgres, todo como servicios de Easypanel
- Dockerfiles multi-stage completos (CMS y web) para monorepo pnpm
- Migraciones de Payload 3 en producción (no hay `push` en prod y nadie te avisa)
- Las 14 trampas de la §6, con síntoma → causa → solución
- Checklist de proyecto nuevo y de verificación post-deploy

## Contexto

Mantenido por [Luismi Lascurain](https://luismilascurain.com) — consultor
digital independiente en Donostia (haciendo webs desde 1996). Si este manual
te ahorra una tarde, ya sabes dónde encontrarme.

Sin garantías: es un manual de campo, no documentación oficial. Úsalo con
criterio y adapta a tu caso.

## Licencia

MIT — ver [LICENSE](./LICENSE).
