---
tags:
  - portfolio/kneos
  - apps
---

# KPaint

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/KPaint.js` — extiende [[File]]. Extensión `"kp"`, ícono propio `sources/appIcon/kpaint.svg` (glyph "palette" de Material Icons, `currentColor`), `src = null`. Agregada 2026-08-20.

> [!abstract] Qué hace
> Editor de dibujo en pixel art: un `<canvas>` de 64×64 (misma resolución `PHOTO_SIZE`/`KPAINT_SIZE` que [[Camera]]/[[ImgFile]]) pintado a mano con el mouse, con una paleta fija de 8 colores en vez de un selector libre — negro (índice 0) + 7 tonos de un degradé hacia `--primary-color` vigente. Diez herramientas: pincel y goma de borrar de tamaño configurable (1-6px), balde de pintura (relleno por inundación), línea, y seis formas geométricas — cuadrado/rectángulo/círculo/elipse/triángulo/estrella — arrastradas con previsualización en vivo y rasterizadas por `utils/pixelShapes.js`. A diferencia de [[Camera]]/[[ImgFile]] (solo lectura, `ViewWindow` de tamaño fijo), es un archivo editable de verdad: usa la `Window` normal (taskbar + resize), mismo criterio que [[TxtFile]], con `Ctrl+Z` para deshacer y guardado manual. ABRIR/NUEVO KP/EXPORTAR (2026-08-20) suman un segundo formato y la posibilidad de bifurcar: se puede cargar y editar una foto `img` existente (cuantizada a la paleta de 8 colores), crear un `kp` nuevo con el dibujo actual sin pisar el que se estaba editando, y exportar el dibujo actual como una `img` nueva — ver "Abrir / Nuevo KP / Exportar" más abajo.

## Constructor(nombre)

`super(nombre, "kp", null, "sources/appIcon/kpaint.svg", FileType.PRODUCTIVITY)`. Arranca con `this._pixels = new Uint8Array(4096)` (todo índice 0, en blanco) en vez de `null` — a diferencia de ImgFile/Camera (que siempre visan una foto ya existente), un `kp` recién creado desde el menú "Nuevo" no tiene ningún contenido guardado todavía y el canvas tiene que verse en blanco de entrada, sin esperar ningún fetch.

> [!success] `this.window` pisado con una `Window` normal, no `ViewWindow`
> A diferencia de Camera/ImgFile, KPaint necesita taskbar + resize reales (es un archivo, no un diálogo de una sola foto). Se pisa igual `this.window = new Window(..., { onTogglePin, tamano: {width:480, height:640}, onClose })` solo para no heredar el tamaño por defecto de `File` (1200×1000 en `window.css`, pensado para texto largo — exagerado para un lienzo de 64×64). `onTogglePin`/`onClose` se preservan explícitamente (mismo callback que pasa `File` en su propio constructor) porque pisar `this.window` entero, a diferencia de heredarlo, pierde esos defaults si no se repiten a mano.

## Paleta (`model/kpaintPalette.js`)

```js
export const KPAINT_SIZE = 64;
export const KPAINT_PALETTE_SIZE = 8;

export function construirPaletaKPaint() {
    return Array.from({ length: KPAINT_PALETTE_SIZE }, (_, i) => mezclarTono(i / (KPAINT_PALETTE_SIZE - 1)));
}
```

Reusa `mezclarTono` de [[Frontend Model Services Utils#Model|themeColors.js]] (la misma interpolación `--primary-background`→`--primary-color` que ImgFile/Camera) en vez de inventar una paleta propia — así los 8 tonos siguen el color elegido en [[Config]] sin ningún cálculo adicional. Cada píxel del dibujo guarda el **índice** de paleta (0-7), no el color final — mismo criterio que `camera_photos`: un dibujo guardado se repinta siempre con el color del tema vigente al abrirlo, no con el que estaba activo al dibujarlo.

> [!info] 8 colores "según el color primario + negro", no monocromo estricto
> La barra de paleta de KPaint usa color real (los 8 tonos, más un borde blanco `#fff` para marcar el swatch seleccionado) en vez de limitarse al verde/tema del resto de KneOS — cubierto por la excepción ya documentada para apps con contenido propio (Kfruit, Doom): acá el "contenido propio" es literalmente la paleta de pintura.

## Dibujado

`_pintarIndices(indices, valor, mutar=true)` pinta un conjunto de índices de píxel con `ctx.fillRect(x, y, 1, 1)` por celda (canvas nativo 64×64, 1 unidad = 1px, sin escalar) — nunca reconstruye los 4096 píxeles para un trazo suelto. `mutar=false` (usado solo por la previsualización de línea/formas, ver más abajo) pinta encima sin tocar `this._pixels`: así un arrastre que no llega a soltarse no deja ningún cambio a medio camino. `_dibujarTodo()` (redibujado completo vía `putImageData`, mismo patrón que `ImgFile._dibujar`) se usa en los puntos de "estado nuevo completo" — carga inicial, deshacer, borrar todo, cambio de tema, **y en cada `pointermove` de línea/formas** (ver preview más abajo, el costo de un `putImageData` de 4096 píxeles por frame es despreciable).

Interacción con `pointerdown`/`pointermove`/`pointerup` + `setPointerCapture` sobre el propio `<canvas>` (no un grid de 4096 `<div>`) — la celda se calcula proyectando `clientX/clientY` contra `getBoundingClientRect()` del canvas escalado por CSS. `setPointerCapture` sigue recibiendo `pointermove` aunque el cursor salga del `<canvas>` a mitad de trazo, así arrastrar rápido no "pierde" el dibujo en el borde. Dos variantes de conversión coordenada→celda, según la herramienta: `_celdaDesdeEvento` (`null` si el cursor está afuera del canvas — usada por pincel/goma/balde, un trazo libre simplemente se corta afuera) y `_coordsClampeadas` (recorta a `[0, 63]` en vez de devolver `null` — usada por línea/formas mientras se arrastra, para que soltar el mouse afuera del canvas complete la forma hasta el borde en vez de perderla).

## Herramientas (`_herramienta`)

Diez valores: `pincel`, `goma`, `balde`, `linea`, `cuadrado`, `rectangulo`, `circulo`, `elipse`, `triangulo`, `estrella` (`HERRAMIENTA_LABELS` para el `aria-label` de cada botón — ver "No native tooltips" en `kneos-conventions`, ninguno usa `title`). Un solo `Set` (`HERRAMIENTAS_DE_FORMA`) separa las seis formas geométricas + línea (arrastre A→B con preview) del resto:

- **Pincel/goma** (`_bindPintura` → rama `trazarLibre`): pintan bajo el cursor en cada `pointermove`, con dedupe por celda (`ultimaCelda`) para no repintar si el mouse no cambió de celda entre eventos. La goma es el pincel forzando `valor = 0` sin importar el swatch seleccionado — el swatch negro (índice 0) ya hacía lo mismo pintando manualmente, la goma es solo el atajo explícito.
- **Balde** (`_balde(x, y)`): relleno por inundación 4-conexo con una pila explícita (no recursión — trivial en tamaño, 4096 celdas como techo absoluto), reemplaza el color conectado a `(x,y)` por `_colorSeleccionado`. Un solo click (`pointerdown`), sin arrastre; corta temprano si el color de destino ya es el seleccionado. Es la única forma de **rellenar** el interior de una figura — las herramientas de forma solo dibujan el contorno (ver más abajo), así que "cuadrado relleno" es "cuadrado" + "balde" clickeado adentro, no un modo aparte.
- **Línea y las seis formas** (`_contornoForma(x0,y0,x1,y1)`, dispatch por `this._herramienta` hacia `utils/pixelShapes.js`): `pointerdown` fija `inicioForma`; cada `pointermove` recalcula el contorno hasta la celda actual, redibuja el estado real (`_dibujarTodo()`) y pinta el contorno previsualizado encima sin mutar `_pixels` (`_pintarIndices(..., false)`) — dedupeado por celda de destino (`ultimoPreview`) para no redibujar si el mouse se movió pero sigue en la misma celda. `pointerup` recalcula el contorno una última vez y lo commitea de verdad (`_pintarIndices(..., true)` por default) antes de empujar al historial.

> [!info] `utils/pixelShapes.js` — helpers puros de rasterizado, sin `this`
> `celdasLinea` (Bresenham) es el bloque de construcción de casi todo lo demás: `celdasTriangulo`/`celdasEstrella` conectan sus vértices con él, `celdasElipse` conecta 48 puntos muestreados paramétricamente sobre la curva con él (en vez de un punto-medio de Bresenham clásico) — así el contorno de cualquier forma queda siempre conectado sin huecos, con una sola implementación de línea reusada en vez de un algoritmo de rasterizado por forma. `celdasCuadrado`/`celdasCirculo` son `celdasRectangulo`/`celdasElipse` forzando lado/radio igual (el mayor de ancho/alto del arrastre) en vez de dejar salir un rectángulo/elipse libre. Todas devuelven listas de `[x,y]`, nunca índices de píxel — la conversión a índice (y el clamp de límites) vive en `KPaint._indicesTrazo`, no acá.

## Tamaño de pincel (`_tamanioPincel`, 1-6px)

Un solo control (`<input type="range">`, `accent-color` para el tinte en vez de reconstruir el thumb a mano) engrosa **tanto** el trazo libre **como** el contorno de línea/formas — no son dos configuraciones separadas. `_indicesTrazo(celdas)` recibe la lista `[x,y]` "delgada" (un punto suelto del pincel, o el contorno entero de una forma) y la expande a un bloque de `_tamanioPincel`×`_tamanioPincel` alrededor de cada celda (ancla arriba-izquierda en tamaños pares, sin centro exacto), devolviendo un `Set` de índices de píxel ya dedupeado — un trazo largo o un contorno con esquinas pisa la misma celda muchas veces. Efecto secundario buscado: una figura con pincel grande sale con contorno grueso, no un contorno de 1px separado de un "grosor de forma" — un solo concepto de grosor para todo lo que se dibuja arrastrando.

## Historial / deshacer (`Ctrl+Z`)

Pila de snapshots (`Array.from(this._pixels)`, tope `HISTORIAL_MAX = 20`) empujada en `_registrarHistorial()` al soltar el mouse (`pointerup`/`pointercancel`) — no en cada `pointermove`, así un trazo entero de arrastre es un solo paso de undo, no uno por píxel. Deduplicado: si el snapshot actual es idéntico al último (p. ej. click sin arrastrar sobre un píxel ya del mismo color), no se apila nada.

> [!info] `Ctrl+Z` con foco propio, no un listener global en `window`
> El `keydown` de `Ctrl+Z` se registra sobre la raíz de la app (`raiz.tabIndex = -1` + `raiz.focus()` en cada `pointerdown` dentro de la ventana), no en `document`/`window` como el prototipo original que trajo el usuario. Con dos ventanas de KPaint abiertas a la vez, un listener global respondería a `Ctrl+Z` sin importar cuál esté realmente en uso; scoped a la ventana evita eso, y de paso no le pisa el undo nativo a un `contentEditable` de [[TxtFile]] abierto en simultáneo.

## Abrir / Nuevo KP / Exportar (`_origen`, 2026-08-20)

`_origen` es `null` mientras se edita el `kp` propio de este ícono (`this.id`), o `{ tipo: "kp"|"img", id }` si se cargó otro archivo con ABRIR — GUARDAR mira acá para saber a dónde escribir, no siempre a `this.id`.

- **ABRIR** (`_toggleSelectorAbrir`): popup anclado bajo la barra de paleta, listando **todo** `kp`/`img` del sistema vía `window.iconServices.getIcons()` (no solo lo ya cargado en memoria — mismo criterio que [[TaskbarSearch]]). Toggle: un segundo click en ABRIR con el popup ya abierto lo cierra. Elegir un ítem llama a `_cargarArchivo(id, ext, nombre)`:
  - `ext === "kp"` → `KPaintServices.getDrawing(id)`, los índices de paleta se copian tal cual.
  - `ext === "img"` → `CameraPhotoServices.getPhoto(id)` (bytes de gris 0-255) cuantizados a la paleta de 8 colores con `grisAIndice` (`model/kpaintPalette.js`, redondeo directo — sin dithering, alcanza para 64×64). Una foto con más de 8 tonos reales sale posterizada al cargarla: trade-off esperado de editarla con la paleta de 8 colores de KPaint, no un bug.
  - En ambos casos: reemplaza `this._pixels`, resetea el historial de deshacer a ese estado (no hay forma de deshacer "hacia atrás" de un `kp` recién cargado), actualiza `_origen`/`_nombreOrigen` y el indicador del footer (`Editando: <nombre>.<ext>`).
- **GUARDAR** (`_guardar`, reescrito): si `_origen?.tipo === "img"`, convierte `this._pixels` a grises con `indiceAGris` y llama `CameraPhotoServices.savePhoto(id, grises)` — **nunca** crea un `kp`. Si no, sigue el camino de siempre (`KPaintServices.saveDrawing`), pero contra `_origen?.id ?? this.id` — si `_origen` apunta a OTRO `kp` (no el propio de este ícono), tampoco se crea nada nuevo, se edita ese archivo en el lugar. `_actualizarColumnaLista` (tamaño/fecha en la vista de lista) solo corre si se guardó en `this.id` — el tamaño de un archivo ajeno vive en su propia instancia, no en esta.
- **NUEVO KP** (`_crearNuevoKp`): crea un `kp` **nuevo** en el escritorio con el dibujo actual (mismo camino que EXPORTAR, sin conversión de paleta — ya es el mismo formato). Pensado para "bifurcar" un dibujo cargado con ABRIR en un archivo propio, en vez de seguir pisando el original. Como EXPORTAR, **nunca toca `_origen`**.
- **EXPORTAR** (`_exportarComoImagen`): siempre crea un `img` **nuevo** en el escritorio (mismo camino que `Camera._guardarFoto`: `buscarEspacioVacio()` + `crearIcono(..., edit=false)`), con `this._pixels` convertido a grises (`indiceAGris`). **Nunca toca `_origen`**: es una copia en otro formato, no un "guardar como" — los próximos GUARDAR siguen yendo a donde iban antes de exportar.

NUEVO KP y EXPORTAR comparten el nombre por defecto (`_generarNombreCopia()`, `"Dibujo <fecha>"` — mismo formato que `Camera._generarNombreFoto`); la extensión final (`kp` vs `img`) la decide `crearIcono`, no el nombre.

> [!info] `_origen` no sobrevive un cierre/reapertura de la ventana
> Cerrar la ventana no resetea `_origen` en memoria (la instancia de `File` sigue viva, ver `Window.cerrar`), así que reabrirla sigue mostrando/guardando lo último cargado con ABRIR. Para "volver" al `kp` propio de este ícono hay que abrir otra instancia (otro ícono `kp`, o el mismo desde cero) — no hay un botón "cerrar archivo cargado" dedicado; no se pidió y hubiera sumado una tercera dimensión de estado sin necesidad clara.

> [!bug] Reabrir la ventana podía mostrar un dibujo desactualizado (encontrado y corregido 2026-08-20, mismo bug que [[ImgFile]])
> `_cargarDibujo()` (la carga inicial del `kp` propio de este ícono al primer `_crearContenido()`) se gateaba con un flag `_cargado`, igual que el `ImgFile._cargarFoto` de antes de su fix: una vez cargado, cerrar y reabrir esta misma ventana (mismo `File` persistido en `window.archivosAbiertos`) nunca volvía a pedir nada, solo redibujaba `this._pixels` tal cual habían quedado en memoria. Eso era inofensivo mientras nada más pudiera tocar ese `kp` — pero dejó de serlo en cuanto una SEGUNDA ventana de KPaint puede abrir ese mismo archivo con ABRIR y guardarle encima (ver arriba): la ventana "dueña" del ícono, si ya se había abierto una vez, seguía mostrando la versión vieja.
>
> **Fix:** se sacó `_cargado`. `_cargarDibujo()` ahora también resuelve `_origen` (no solo el `kp` propio) y `_crearContenido()` la llama sin condición en cada apertura — mismo tratamiento que `ImgFile._cargarFoto`: redibuja lo que haya en memoria de una para no dejar el canvas en blanco mientras se espera la respuesta, y la pisa apenas el fetch fresco vuelve.

## Guardado (`_guardar`)

Igual que `TxtFile._guardar` en el caso base (sin `_origen`, ver arriba): `KPaintServices.saveDrawing(id, pixels)` → texto de estado en el footer ("Guardado"/"Error al guardar", 2s) vía `_mostrarEstadoGuardado` (ahora en `_footerEstadoEl`, ver CSS) → si tuvo éxito, `_actualizarColumnaLista` recalcula `size` client-side (`Blob([JSON.stringify(pixels)]).size`, mismo cálculo que hace el backend) y propaga el delta a las carpetas ancestro.

> [!info] GUARDAR limpia el canvas tras un guardado exitoso (agregado 2026-08-20)
> A pedido del usuario: `_guardar()` llama a `_borrarTodo()` (mismo método que el botón BORRAR — `fill(0)` + redibujar + empujar al historial) inmediatamente después de un `result` truthy, tanto en la rama `_origen?.tipo === "img"` como en la de `kp`. Deja el lienzo en blanco para arrancar el próximo dibujo sin cerrar/reabrir la ventana. **No toca `_origen`/`_nombreOrigen`**: si se estaba editando un archivo ajeno (cargado con ABRIR), GUARDAR sigue apuntando ahí — un segundo click en GUARDAR sin haber dibujado nada nuevo sobreescribe ese mismo archivo con un lienzo vacío, a propósito (mismo criterio que un editor de texto que vacía el campo al enviar: el usuario pidió limpiar, no "limpiar salvo que eso pise algo").

## CSS (`kpaint.css`)

Cuatro franjas verticales: `.kpaintToolbar` (herramientas + tamaño de pincel) → `.kpaintPaletteBar` (swatches + ABRIR/NUEVO KP/EXPORTAR/DESHACER/BORRAR/GUARDAR) → `.kpaintCanvasWrap` (canvas centrado) → `.kpaintFooter` (dos secciones: `.kpaintFooterOrigen` a la izquierda con ellipsis si el nombre es largo, `.kpaintFooterEstado` a la derecha). `.kpaintCanvas` usa `width: 100%; max-width: 420px; aspect-ratio: 1/1` (mismo criterio que `.imgCanvas`, ver [[ImgFile]]) + `touch-action: none` para que arrastrar sobre el canvas no dispare scroll/zoom táctil. `.kpaintAbrirPopup` (lista de ABRIR) ancla igual que `.txtAiPopup` (ver txt.css): `position: absolute; top: 100%` sobre `.kpaintPaletteBar` (`position: relative`).

Los botones de herramienta (`.kpaintTool`) siguen el verde monocromático del resto de la UI (a diferencia de `.kpaintSwatch`, que sí sale de la paleta — ver arriba): son controles, no colores. Pincel/goma/balde/línea van en texto plano (LAPIZ/GOMA/BALDE/LINEA, `.cameraBoton` como referencia — no hay íconos de esas acciones en `sources/accions/`); las seis formas geométricas se identifican con un ícono dibujado en **CSS puro** (`::before`, sin ningún asset SVG nuevo) en vez de texto — el ícono literalmente ES la forma que van a dibujar: un `<div>` cuadrado/rectangular (`width`/`height`), un círculo/elipse (`border-radius: 50%`), un triángulo (el truco clásico de `border` transparente + un lado sólido) y una estrella de 5 puntas vía `clip-path: polygon(...)`. `.kpaintTool--activo` reusa el mismo invert que `:hover` (`background: var(--primary-color); color: var(--primary-background)`) pero sostenido, para marcar la herramienta seleccionada sin depender de que el mouse siga encima. ABRIR/NUEVO KP/EXPORTAR/DESHACER/BORRAR/GUARDAR (`.kpaintIcon`) son todos texto plano — mismo motivo que las cuatro primeras herramientas: no hay íconos de open/export/undo/trash/save que calcen con `sources/accions/`.

## Registro (sí está en el menú "Nuevo")

`kp` está en `model/iconSrc.js`, `formato.js` (`TIPOS.kp = "Dibujo"`), y — a diferencia de `img` — sí en el submenú "Nuevo" de `Folder._abrirSubMenuCrear` y `ContextMenuManager._abrirSubMenuCrear` (un lienzo en blanco tiene sentido, a diferencia de una foto en blanco): mismo patrón que `txt`/`fld`. No está en `filesUndeletable.js` (se puede borrar como cualquier archivo de usuario).

> [!info] También sembrado en `defaultFiles.js` (2026-08-20) — pero sigue siendo un archivo de usuario común
> A diferencia del resto de entradas de `defaultFiles.js` (Doom/Kmd/Kfruit/KneAI/Maxwell/RecycleBin/Calculator/User/BlackJack/Hangman/FlipCoin/Kdle/CarRace/Tetris/KneChat/Config — todas "fijas": `filesUndeletable` + fuera del menú "Nuevo"), `KPaint` se agregó a `defaultFiles` (`espacio65`, nombre "KPaint") solo para que un escritorio nuevo arranque con un lienzo a mano, sin volverse `filesUndeletable` ni sacarse del menú "Nuevo" — se puede borrar el ícono default y crear otros `kp` libremente, es un lienzo en blanco precargado, no una app-herramienta singleton.

## Persistencia

`kpaint_files` (1:1 con `files` por `id_icon`) — ver [[Módulo KPaint]].
