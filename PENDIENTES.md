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
