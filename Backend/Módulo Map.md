---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Map

⬅️ Volver a [[Backend]]

Proxy hacia [Nominatim](https://nominatim.org/) (el geocoder de OpenStreetMap), para el buscador de lugar de [[KneMap]] (ciudad, dirección, provincia, país). Igual que [[Módulo Groq]], es un módulo sin `controllers/`/`models/` dedicados — la lógica vive inline en `routes/mapRoutes.js`, sin tabla propia (nada se persiste).

**Por qué hace falta un proxy acá y no en `MapTileServices.js` (los vector tiles, que sí van directo del navegador a OpenFreeMap sin pasar por acá):** Nominatim exige, en su [política de uso](https://operations.osmfoundation.org/policies/nominatim/), un header `User-Agent` que identifique la aplicación — un `fetch()` del navegador no puede setear ese header (es uno de los ["forbidden header names"](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name) del spec de Fetch), así que la consulta tiene que salir del servidor. OpenFreeMap, en cambio, no pide nada de eso (CORS abierto, sin política de atribución más allá de la estándar de OSM), por eso ahí no hace falta proxy.

## Endpoint (montado en `/mapRoutes`, `requireAuth` + `mapGeocodeLimiter` en todo el router)

| Método | Path | Qué hace |
|---|---|---|
| GET | `/geocode?q=<texto>` | Geocodifica `q` contra Nominatim, **sin restringir por tipo** — devuelve el resultado más relevante según el ranking propio de Nominatim, sea país, provincia, ciudad, dirección o POI |

> [!bug] Arrancó restringido a `featureType=state`, sacado el mismo día (2026-08-21)
> El pedido original era literalmente "buscador de provincia", así que la primera versión filtraba con `featureType=state` — pero eso hacía que cualquier búsqueda de ciudad o dirección devolviera vacío (`Mar del Plata` → `[]`, comprobado contra la API real de Nominatim). El usuario preguntó "¿no busca por ciudad/dirección?" apenas lo probó — se sacó el filtro (ver commit del mismo día) y ahora es un buscador de lugar general, no solo de provincias. `AJUSTE_ZOOM_MARGEN`/`_zoomParaBoundingBox` (frontend, ver [[KneMap]]) ya andaban bien para cualquier tamaño de `boundingbox`, así que no hizo falta tocar nada más.

Devuelve `{ result: null }` (nunca un `4xx`/`5xx`) tanto si no hay coincidencias como si Nominatim falla o `q` es inválido (vacío o más de 200 caracteres) — el frontend (`MapGeocodeServices.buscarLugar`) no necesita distinguir "no hay resultado" de "algo salió mal", en los dos casos el mapa simplemente no se mueve, sin cartel de error (ver [[Reglas]]). El error real (si lo hubo) se loguea en el servidor.

`result`, cuando existe: `{ name, lat, lon, boundingbox: [sur, norte, oeste, este] }` (`boundingbox` tal cual lo devuelve Nominatim, convertido a números) — `KneMap._buscarLugar` usa `lat`/`lon` para centrar y `boundingbox` para calcular a qué zoom encuadrar el resultado.

## Respetando la política de uso de Nominatim

Nominatim es gratuito y pide explícitamente **no superar ~1 request/segundo** y **no hacer geocoding en bulk/sistemático**. Dos capas, cada una cubriendo un caso distinto:

1. **`mapGeocodeLimiter`** (20 req/min por `pc_id`, ver [[Backend#middlewares]]): frena a una sesión individual, mismo mecanismo que `groqLimiter`.
2. **Cola con espaciado, módulo-level** (`encolarNominatim`, dentro de `mapRoutes.js`): TODAS las búsquedas de TODAS las sesiones pasan por una única cadena de promesas con un piso de `1100ms` entre llamadas reales a Nominatim — el limiter de arriba frena a un usuario, pero no evita que 5 sesiones distintas, cada una dentro de su propio límite, sumen más de 1 req/seg contra Nominatim. La cola sí lo evita, sin importar cuántas sesiones pidan a la vez.

Además, un **caché en memoria** (`Map`, tope 200 entradas, sin TTL — nombres/coordenadas de un lugar no cambian) evita repetir la llamada real ante la misma búsqueda, mismo patrón que el caché de tiles de `MapTileServices.js` (frontend, ver [[KneMap]]).

## `USER_AGENT`

`'KneOS-Portfolio/1.0 (https://github.com/ManuCanello/Portfolio)'` — string fijo en `mapRoutes.js`, no una env var: no es un secreto (a diferencia de `GROQ_API_KEY`), es justamente lo que Nominatim pide que sea público/identificable.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|MapGeocodeServices.js]] (`buscarLugar(query)`), usado por [[KneMap]] — buscador de lugar, solo en el escritorio (la terminal ya tiene `map <ciudad>`/`map lat lon`, ver [[Kmd]]).
