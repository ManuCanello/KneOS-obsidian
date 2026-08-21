---
tags:
  - portfolio/kneos
  - apps
---

# KneMap

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/KneMap.js` — extiende [[File]]. Extensión `"map"`, ícono `sources/appIcon/map.svg` (pin de mapa, mismo estilo pixel-art en `<rect>` que el resto de `appIcon/`), `src = null`, tamaño declarado `41_000` bytes (no persiste contenido propio en BD, mismo criterio que [[Kmd]]/[[Doom]]).

> [!abstract] Qué hace
> Puerto de [MapSCII](https://github.com/rastapasta/mapscii) (Michael Strassburger, MIT): mapa mundial navegable dibujado como una **grilla de texto ASCII** verde monocromático — flechas/`hjkl` paneán, `a`/`z` zoomean, click centra, rueda zoomea hacia el puntero, arrastrar paneá. Corre igual dentro de una `Window` del escritorio que dentro de la terminal (`map`, ver [[Kmd]]) — a diferencia de los seis juegos portados de Java no hay un "modo consola" con DOM propio: la salida ya es texto plano en los dos casos, es literalmente el mismo `_iniciarMapa()`. En el escritorio además hay un **buscador de lugar** (ciudad, dirección, provincia, país — 2026-08-21, geocoding real vía Nominatim, ver más abajo).

## Por qué es un puerto y no el paquete `mapscii` de npm

MapSCII es un programa de **terminal real** (Node: `keypress`, `term-mouse`, `node-fetch`, escritura directa a `process.stdout` con secuencias ANSI) — no corre en un navegador tal cual. Además, `tiles.mapscii.me` (su tile server por defecto) está **caído** (`NXDOMAIN`, comprobado), así que ni siquiera con un shim de I/O hubiera funcionado.

De sus ~13 dependencias, solo el motor de render es JS puro y portable: `@mapbox/vector-tile` + `pbf` (parseo de vector tiles), `earcut` (triangulación de polígonos), `rbush` (índice espacial), `bresenham` (líneas). Esas 4 (más `simplify-js`, descartada — ver abajo) se vendorizaron y adaptaron a mano en `public/KneOS/js/vendor/mapscii/` en vez de importarse de `node_modules/mapscii`: su `TileSource.js` original hace `require('fs')`/`node-fetch` (rompería el bundle de esbuild), y de todas formas había que reescribir la última etapa del render (Braille → ASCII, ver abajo) y el estilo (esquema de tiles distinto, ver abajo). Queda auditable y no depende de un paquete sin release desde 2020.

## `vendor/mapscii/` — qué cambió respecto al original

| Archivo | Cambio respecto al MapSCII original |
|---|---|
| `BrailleBuffer.js` | Sin caracteres Braille (`0x2800+mask`): W95FA no trae ese bloque Unicode. En su lugar, una rampa de densidad de 9 niveles ASCII (`" .:+#"`, indexada por popcount de los 8 sub-pixeles de la celda) — el buffer de sub-pixeles 2×4 en sí (`pixelBuffer`) no cambia, sigue siendo la misma resolución que Braille, solo cambia el glifo final. Sin color ANSI real (`x256`/`_termColor`): cada celda lleva un **nivel** 0-2 (ver `Tile.js` abajo), y `rows()` (ex `frame()`) devuelve, por fila, tramos `{text, level}` en vez de un string con secuencias de escape — así `KneMap._draw()` arma un `<span>` por tramo (no uno por celda). `Buffer` (Node) → `Uint8Array`. |
| `Canvas.js` | Mismo dibujo de líneas (bresenham)/polígonos (earcut + triángulos rellenos). Se saca `setBackground(x,y,color)` por celda (no la usaba nadie en `_drawFeature`, solo quedaba el color de fondo global) → `setGlobalLevel(level)`. |
| `LabelBuffer.js` | `string-width` (ancho real de un string con CJK/emoji de doble ancho) → `text.length`: los labels salen con `config.language = "es"` (name:es/name_en/name), en la práctica alfabeto latino de ancho simple. |
| `Renderer.js` | Misma lógica de qué tiles son visibles y en qué orden dibujar. `_getFrame()` (ANSI `CLEAR`/`MOVE`) → `_getRows()`, directo `canvas.rows()`. La detección de "esto es una etiqueta, dibujar al final" pasaba por `layerId.match(/label/)` (cierto en el esquema viejo: `place_label`, `poi_label`...) → `feature.style.type === "symbol"`, no depende del nombre de la capa. Sin `simplify-js` (config.simplifyPolylines ya no existe, era `false` por defecto de todas formas). |
| `Tile.js` | `Pbf`/`zlib` (Node) → `PbfReader` de `pbf@5` (`@mapbox/vector-tile@3` ya lo pide así) y `DecompressionStream` nativo del navegador para el gzip-de-contenido defensivo (OpenFreeMap no lo necesita, comprobado con un tile real). `x256(hex2rgb(color))` → nivel de intensidad 0-2 por **nombre de capa** (`classifyIntensity`, ver tabla abajo) — ya no hay color real que extraer de `line-color`/`fill-color`/`text-color`. |
| `Styler.js` | Sin cambios de lógica, solo CJS→ESM. **Ojo:** `_compileFilter` para `"all"`/`"none"` replica tal cual una inversión que ya tenía el original (herencia de su transpilación CoffeeScript→JS, nunca corregida upstream) — se dejó a propósito sin "corregir" para que el archivo siga siendo un puerto fiel; `style.json` (ver abajo) no usa `"all"`/`"none"` en ningún filtro, así que hoy no tiene efecto práctico. |
| `utils.js`, `config.js` | Proyección (`ll2tile`/`tile2ll`/`baseZoom`/etc.) sin cambios, puro. `config.js` perdió los campos que solo tenían sentido con una terminal real de Node (`input`/`output`, `headless`, `delimeter` ANSI, `persistDownloadedTiles` a disco, `useBraille`) y el `source`/`styleFile` originales (van en `MapTileServices.js`/`style.json` respectivamente). |
| `style.json` | **No es el `dark.json` del original.** MapSCII apunta al esquema viejo de mapscii.me (osm2vectortiles ~2016: `road`, `place_label`, `admin`...); OpenFreeMap sirve el esquema **OpenMapTiles** actual (`transportation`, `place`, `boundary`... comprobado bajando un tile real). Traducir 1560 líneas de filtros contra un esquema distinto era más riesgoso que escribir un estilo chico de cero — que además no necesita `paint` (colores): la intensidad se decide por nombre de capa, no por color real. 17 capas (agua/vías/edificios/etiquetas/POIs/etc.), sin geocoding de ningún tipo. |

`config.poiMarker = "*"` (el `◉` Unicode del original tampoco es ASCII puro).

### `Tile.js` — niveles de intensidad por capa

```
DIM_LAYERS (0, --primary-dim)    = water, waterway
BRIGHT_LAYERS (2, --primary-color, opaco) = transportation, transportation_name,
    boundary, place, poi, aerodrome_label, housenumber, water_name, mountain_peak
todo el resto (1, --primary-color, semitransparente) = landcover, landuse, park,
    building, aeroway, ...
```

Agua tenue, vías/límites/etiquetas fuertes (son las que orientan), el resto — uso de suelo, edificios, parques — medio. Todo el mismo verde de sistema (`--primary-color`), solo cambia la opacidad vía `.kneMapLvl0/1/2` en `knemap.css` — sigue la regla de paleta monocromática de [[Reglas]].

## Tile source — `services/MapTileServices.js`

No pasa por `apiFetch`/el backend propio, a diferencia del resto de `services/*.js` (excepción compartida con `chatSocket.js`, que tampoco es HTTP): habla **directo** con [OpenFreeMap](https://openfreemap.org/) desde el navegador —

```
https://tiles.openfreemap.org/planet/<version>/{z}/{x}/{y}.pbf
```

200, `access-control-allow-origin: *`, sin API key (comprobado bajando un tile real de Buenos Aires). El segmento de versión sale de `GET https://tiles.openfreemap.org/planet` (TileJSON), con `latest` como plantilla de respaldo si esa llamada falla. Caché LRU en memoria (`Map`, tope 16 tiles, mismo tope que el original) — nada de `fs`, nada en BD.

**Sin proxy en Express, a propósito**: el CORS wildcard ya está confirmado, y un proxy server-side para una URL fija de terceros sería el mismo riesgo de SSRF innecesario que ya evita `Kmd._cmdCurl` (ver [[Kmd]]). Un tile que falla (red caída, 404) se descarta en silencio — la grilla se dibuja con lo que haya, sin cartel de error, según [[Reglas]].

## `_iniciarMapa()` — medición de la grilla y bucle de redibujado

`_crearContenido()` arma `.kneMapGrid` (la grilla) + `.kneMapFooter` (coordenadas/zoom) y delega en `_iniciarMapa()`, que:

- Mide el ancho/alto real de un caracter con un `<span>` invisible (`.kneMapProbe`, 20 `#` de `white-space:pre`) en vez de asumir un tamaño de fuente — si el fallback de fuente cambia, la grilla se sigue alineando sola. `.kneMapGrid` usa un stack monoespaciado propio (`ui-monospace, Consolas, "Cascadia Mono", monospace`), no `W95FA`: la fuente del sistema no es de paso fijo para todo el rango que usa la grilla (mismo motivo por el que `Kmd._printDirGrid` usa CSS grid en vez de rellenar con espacios).
- `cols`/`rows` = tamaño del contenedor / tamaño de celda (mínimo 20×10) → `renderer.setSize(cols*2, rows*4)` (submuestreo 2×4, el mismo de siempre).
- Bucle `requestAnimationFrame` continuo con flag `_dirty` (mismo patrón que [[Maxwell]]): redibuja solo cuando hace falta, y hace doble función de "detectar que la ventana se cerró" (`!wrapper.isConnected`) — no hay un game loop propio que ya chequee eso como en [[Tetris]]/[[CarRace]] — y "Kmd cortó todo con `_detener()`" (`this._detenido`, Ctrl+C, ver [[File]]) para soltar `ResizeObserver` + el listener de teclado (`_desregistrarTeclado`).

> [!bug] `_crearContenido()` corre con el DOM todavía desconectado (2026-08-21, resuelto)
> `Window.abrir()` recién appendea lo que devuelve `_crearContenido()` **después** de que la función retorna — así que la primera medición síncrona de `.kneMapProbe` (dentro de `_crearContenido()`) corre con `grid` todavía fuera del documento, y `getBoundingClientRect()` de un nodo desconectado da todo cero. Con `charWidth`/`charHeight` en `0`, `resize()` corta temprano (mismo guard que ya usa [[Maxwell]] con su `container`) y `renderer.setSize()` nunca llega a correr — `renderer.canvas` queda `undefined` hasta que el `ResizeObserver` dispara solo apenas `grid` tiene layout real (se lo puede `.observe()` estando desconectado, el callback llega igual en cuanto entra al documento). El bucle de dibujo ya estaba guardado contra esto (`renderer.canvas` truthy antes de dibujar), pero **los listeners de mouse no**: un `pointerup`/`pointermove` que llegara en esa ventana (pasó en la práctica corriendo un test con Playwright, aparentemente por el propio gesto del doble click sobre el ícono) dividía por `this._charWidth` todavía `undefined` → `NaN` → `_setCenter(NaN, NaN)` → `utils.normalize()` no sanea `NaN` → el mapa quedaba roto **para siempre** (toda esquina de la sesión, mapa en blanco, footer "NaN, NaN"). Dos guards de respaldo: `_eventToLatLon` devuelve el centro actual sin tocar nada si `_charWidth`/`_charHeight` todavía no existen, y `_setCenter` descarta cualquier `lat`/`lon` no finito antes de tocar `this._center` — cualquiera de los dos alcanza, se dejaron los dos porque son baratos y cada uno cubre una ruta de entrada distinta.

## Teclado/mouse

- **Teclado** (`_onKey`, vía `this._registrarTeclado`): flechas/`hjkl` panean, `a` zoom in, `z`/`y` zoom out, `q` solo hace algo en modo terminal (`this._salirTerminal?.()`, ver [[Kmd]] — Ctrl+C ya cubre la salida en cualquier momento). **Guard contra escribir en otro lado**: a diferencia de las flechas, `h`/`j`/`k`/`l`/`a`/`z`/`q` son letras comunes — `_registrarTeclado` engancha en `window` sin noción de qué ventana está "activa" (mismo mecanismo que usa [[Tetris]]/etc.), así que tener el mapa abierto de fondo mientras se escribe en [[Kmd]], un `.txt` o el formulario de [[User]] paneiaría/zoomearía el mapa en vez de escribir — `_isTypingElsewhere()` (`document.activeElement` es `INPUT`/`TEXTAREA`/`contentEditable`) corta el handler entero en ese caso. Es un guard nuevo, sin precedente en el resto de KneOS (ningún otro `_registrarTeclado` lo chequea) — se agregó acá porque el resto de las apps con teclado global usan flechas/espacio, letras sueltas casi no colisionan con texto real.
- **Mouse** (pointer events + `setPointerCapture`, mismo patrón que [[KPaint]]): `pointerdown`+`pointerup` sin moverse = click, centra; con movimiento = drag, paneá; `wheel` zoomea hacia el punto bajo el cursor (mismo algoritmo que el `Mapscii._onMouseScroll` original: guarda la posición lat/lon bajo el puntero antes y después de zoomear, corrige el centro para que quede en el mismo lugar).
- `_colrow2ll(col, row)` es un puerto directo de `Mapscii._colrow2ll` del original.

## Buscador de lugar (2026-08-21, solo en el escritorio)

`_crearBarraBusqueda()` agrega un `<input>` (`.kneMapSearchBar`, arriba de la grilla) **solo cuando `!this._modoTerminal`** — dentro de la terminal ya existe `map <ciudad>`/`map lat lon [zoom]` (ver más abajo), y un `<input>` ahí adentro competiría por el foco con el propio input de [[Kmd]] al reenganchar el prompt. Enter dispara `_buscarLugar(input)`:

1. `MapGeocodeServices.buscarLugar(query)` → `GET /mapRoutes/geocode?q=...` (ver [[Módulo Map]]) — Nominatim sin restricción de tipo: ciudad, dirección, provincia o país, el que mejor rankee. **Corrección el mismo día:** arrancó restringido a `featureType=state` (pedido original era específicamente "buscador de provincia"), pero eso dejaba afuera cualquier búsqueda de ciudad o dirección (`Mar del Plata` devolvía vacío, por ejemplo) — se sacó el filtro apenas se detectó, ahora es un buscador de lugar en general.
2. Sin resultado: no pasa nada — ni cartel de error ni cambio en el mapa (ver [[Reglas]]).
3. Con resultado: `_setCenter(lat, lon)` + zoom calculado a partir del `boundingbox` que devuelve Nominatim con `_zoomParaBoundingBox` (fuera de `vendor/mapscii/`, es funcionalidad nueva, no parte del puerto) — el mismo truco de "ajustar zoom a un bounding box" que usa la API JS de Google Maps (`getBoundsZoomLevel`), adaptado a las unidades de render de `vendor/mapscii/utils.js` en vez de las de Google: `WORLD_DIM = 256` porque en ese sistema de proyección el ancho del mundo en unidades de render a un zoom entero es `256 * 2^zoom` (se confirma agregando: `ll2tile` dice cuántos tiles-índice hay a ese zoom, `tilesizeAtZoom` cuántas unidades de render mide cada uno — el producto de los dos es exactamente esa fórmula). Un margen fijo (`AJUSTE_ZOOM_MARGEN = 0.5`) deja algo de aire alrededor en vez de encuadrar el resultado pegado a los bordes; el resultado se clampea entre `_minZoom` (el de la grilla actual) y `config.maxZoom`.

El input tiene su propio `keydown` con `stopPropagation()` — no llega a `_onKey` (que igual lo hubiera ignorado por `_isTypingElsewhere()`, ver más abajo) ni a ningún otro listener global de `window` que pudiera estar escuchando de fondo.

## Kmd — comando `map` (2026-08-21, ver [[Kmd]])

`Kmd._cmdRun` (el que corre `run <juego>` dentro de la terminal) se partió en dos: el bloque de "tapar `.kmdOutput`, montar `_crearContenido()`, escuchar Ctrl+C en fase de captura, reenganchar `_salirTerminal`" pasó a `Kmd._runInline(instancia)`, compartido por `_cmdRun` y el nuevo `_cmdMap` — el mecanismo ya estaba pensado para cualquier app que se apodere del teclado/pantalla, el mapa cae en el mismo caso exacto que los seis juegos.

`map` sin argumentos abre en el centro por defecto de `KneMap` (Buenos Aires, zoom 4). `map <lat> <lon> [zoom]` salta a coordenadas. `map <ciudad>` busca en `MAP_CIUDADES` (tabla fija en `Kmd.js`, un token sin espacios por ciudad — `buenosaires`/`tokio`/`nuevayork`/`londres`/`paris` hoy; mismo criterio que `RUN_GAMES.carrera` → `"carreraauto"`), **sin geocoding real** acá tampoco (mismo motivo: Nominatim exige un `User-Agent` que un `fetch` del cliente no puede setear). El geocoding real (2026-08-21) sí existe, pero solo del lado del escritorio — ver "Buscador de lugar" más arriba; no se sumó también a Kmd porque un `<input>` de búsqueda ahí adentro competiría por el foco con el propio input de la terminal, y `map <ciudad>` ya cubre el caso de uso rápido en texto plano.

## Estilos

`styles/apps/knemap.css` (nuevo) + `@import` agregado a `main.css`. `.kneMapApp` funciona tanto con clase `app` (Window real) como sin ella (dentro de `kmdRunContainer`, modo terminal) — declara su propio `flex:1;min-height:0;width:100%` en vez de depender de heredarlo de `.app` (que solo existe en un caso de los dos). `.kneMapSearchBar`/`.kneMapSearchInput` (2026-08-21) calcan el look de `.taskbarBuscadorAppInputRow` (`core/search.css`, ver [[Window y Taskbar]]) — borde sólido + input transparente, pero heredando la tipografía del sistema (`W95FA`) en vez de la monoespaciada de `.kneMapGrid`, porque es texto libre, no una grilla que necesite alinearse. Sin `text-shadow` (ver [[Reglas]]).
