# Pendientes para el manual

Lecciones descubiertas en proyectos, aún sin incorporar a
DEPLOY-ASTRO-PAYLOAD-EASYPANEL.md. Se vacía cuando se consolidan.

## scloinaz (septiembre 2026)

Ver `~/dev/scloinaz/ESTADO.md` §9 para el detalle con síntoma y remedio.

- turbopack.root en monorepo pnpm: /admin da 500 en bucle con error engañoso
- Normalización NFC/NFD en macOS: comparaciones de nombres fallan en silencio
- DROP COLUMN al activar localización en Payload sobre contenido ya cargado
- FlatCompat de eslint-config-next v16 rompe el lint sin mencionar la config
- En pnpm, lo que se importa hay que declararlo
- PORT en Next 16: sin la variable escucha en el 80 aunque el proxy apunte al 3000
- .dockerignore ausente: los .env con secretos acaban dentro de la imagen
- tar en macOS mete ._ficheros que su propio tar -t esconde (COPYFILE_DISABLE=1)
- El setval de secuencias sobrevive a --exclude-table en pg_dump
- Ficheros que en macOS son uno y en Linux son dos, por la caja de las letras
- Astro emite 404.html en la raíz pero 410 en dist/410/index.html

Cinco de estas tienen la misma raíz: macOS y Linux tratan los ficheros de
forma distinta. Probablemente merecen una sección propia.

Pendiente también: fusionar la receta del botón de publicar
(RECETA-PUBLICACION-ASTRO-PAYLOAD-EASYPANEL.md, hoy en luismilascurain.com)
con el patrón de relay scoped que se implemente en scloinaz.

## luismilascurain (agosto 2026) — §6.14 se quedó en la foto de mayo

La §6.14 cubre el disparo y la trampa de la URL, pero desde entonces se
aprendió bastante más y NADA de esto está en el manual:

- El cinturón del Paso 10. Es lo más grave: un build que falla a medias
  publica un sitio roto CON CÓDIGO 0. Pasó en vivo el 03/08/2026, 17 páginas
  como redirecciones vacías. Causa raíz: Payload no devuelve 404 cuando un
  documento no existe, contesta 200 con docs vacío, así que "no existe" y "no
  he podido hablar con el CMS" caen en el mismo catch. Hoy el manual explica
  cómo montar el botón sin avisar de esto.
- El semáforo de estado: build.json en el front, las dos cantidades que no son
  la misma, el bug del primer día, el estado ⚪ que dice "no lo sé".
- El cuelgue de Corepack al forzar reconstrucción (19/08). Cero menciones.
- La regla de cuándo forzar y cuándo no, con sus tres casos.
- El relay scoped, hoy dos líneas de aviso sin desarrollar.

Todo está en RECETA-PUBLICACION-ASTRO-PAYLOAD-EASYPANEL.md (693 líneas), en el
repo de luismilascurain.com. Al consolidar, fusionarla con lo que salga del
relay de scloinaz.

## Cuándo

Al cerrar scloinaz, no antes. Motivo: el relay scoped que se implementa allí
cambia justo la parte de seguridad de la §6.14, y documentarlo dos veces sale
peor. Sesión propia, no un rato entre tareas.

## Al consolidar, decidir también el formato

Con las 11 de scloinaz y las 4 de luismilascurain, la §6 pasa de 20 trampas a
35. Hoy están ordenadas por orden de descubrimiento, así que para saber si una
te aplica hay que leerlas todas.

Plantearse agruparlas por FASE: build · arranque del contenedor · red y DNS ·
datos y migraciones · sistema de ficheros (macOS vs Linux). Solo esa última
tiene ya cinco casos de scloinaz.
