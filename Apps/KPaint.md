---
tags:
  - portfolio/kneos
  - apps
---

# KPaint

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/KPaint.js` — extiende [[File]]. Extensión `"kp"` por defecto (parametrizable, ver "Editor vs. documentos" más abajo), ícono propio `sources/appIcon/kpaint.svg` (glyph "palette" de Material Icons, `currentColor`), `src = null`. Agregada 2026-08-20.

> [!abstract] Qué hace
> Editor de dibujo en pixel art: un `<canvas>` de 64×64 (misma resolución `PHOTO_SIZE`/`KPAINT_SIZE` que [[Camera]]/[[ImgFile]]) pintado a mano con el mouse, con una paleta fija de 8 colores en vez de un selector libre — negro (índice 0) + 7 tonos de un degradé hacia `--primary-color` vigente. Diez herramientas: pincel y goma de borrar de tamaño configurable (1-6px), balde de pintura (relleno por inundación), línea, y seis formas geométricas — cuadrado/rectángulo/círculo/elipse/triángulo/estrella — arrastradas con previsualización en vivo y rasterizadas por `utils/pixelShapes.js`. A diferencia de [[Camera]]/[[ImgFile]] (solo lectura, `ViewWindow` de tamaño fijo), es un archivo editable de verdad: usa la `Window` normal (taskbar + resize), mismo criterio que [[TxtFile]], con `Ctrl+Z` para deshacer y guardado manual. ABRIR/GUARDAR COMO (2026-08-20, rediseñado 2026-08-21) suman un segundo formato y la posibilidad de bifurcar: se puede cargar y editar una foto `img` existente (cuantizada a la paleta de 8 colores), crear un `kp` nuevo con el dibujo actual sin pisar el que se estaba editando, y exportar el dibujo actual como una `img` nueva — ver "Abrir / Guardar como" más abajo.

## Constructor(nombre, extension = "kp")

`super(nombre, extension, null, "sources/appIcon/kpaint.svg", FileType.PRODUCTIVITY)`. Arranca con `this._pixels = new Uint8Array(4096)` (todo índice 0, en blanco) en vez de `null` — a diferencia de ImgFile/Camera (que siempre visan una foto ya existente), un `kp` recién creado desde el menú "Nuevo" no tiene ningún contenido guardado todavía y el canvas tiene que verse en blanco de entrada, sin esperar ningún fetch.

> [!success] Editor vs. documentos (`apps/KPaintApp.js`, 2026-08-21) — split de identidad
> A pedido explícito del usuario ("kpaint no se debe poder borrar" + "kpaint el editor debe tener otra extensión que los archivos"): antes un solo ícono `kp` era a la vez "la app KPaint" (el ícono de fábrica del escritorio) y "un dibujo real" (con su propia fila en `kpaint_files`), así que no había forma de proteger uno de un borrado accidental sin proteger también los dibujos comunes creados por el usuario — `filesUndeletable.js` (ver [[Menús Contextuales]]) protege por **extensión**, no por ícono puntual.
>
> Solución, mismo criterio que [[Camera]] (`"camera"`, protegida) vs [[ImgFile]] (`"img"`, borrable): se agregó `KPaintApp extends KPaint`, una subclase de ~10 líneas que solo pisa la extensión a `"kpaint"` (`super(nombre, "kpaint")`) — el ícono de fábrica (`model/defaultFiles.js`, `espacio65`) pasó de `ext: "kp"` a `ext: "kpaint"`, y `"kpaint"` se sumó a `filesUndeletable.js`. `"kp"` (los dibujos) **no** está en `filesUndeletable` — siguen siendo borrables como cualquier documento de usuario. `model/iconSrc.js` registra ambas extensiones apuntando a sus respectivas clases (`kp` → `KPaint.js`, `kpaint` → `KPaintApp.js`), mismo ícono (`kpaint.svg`) para las dos.
>
> `KPaint._esLanzador` (`this.extension !== "kp"`) es la única bifurcación de comportamiento entre ambas identidades — la interfaz de edición es 100% compartida, no hay dos UI distintas como en Camera/ImgFile:
> - `_cargarDibujo()`: si es lanzador y no hay `_origen`, no intenta fetchear nada (nunca tiene fila en `kpaint_files`) — arranca en blanco directo.
> - `_guardar()`: si es lanzador y no hay `_origen`, el primer click en GUARDAR crea un `.kp` nuevo (`_crearKpNuevo()`, helper compartido con `_crearNuevoKp`/GUARDAR COMO) y lo **adopta** como `_origen` — los GUARDAR siguientes ya escriben ahí en vez de crear un archivo nuevo en cada click. Verificado con Playwright: dos GUARDAR seguidos desde el lanzador dejan un solo `.kp` en el sistema, no dos.
>
> El menú "Nuevo" (`ContextMenuManager._abrirSubMenuCrear` y su duplicado en `Folder.js`) sigue ofreciendo crear un `.kp` en blanco directo (sin pasar por el lanzador) — se le cambió la etiqueta de "KPaint" a "Dibujo" para no confundirse con el ícono de la app, mismo texto que ya usa `formato.js` (`TIPOS.kp = "Dibujo"`, con `TIPOS.kpaint = "Aplicación"` agregado). El lanzador **no** se agregó al menú "Nuevo" (no se "crean" instancias de una app, mismo criterio que `"camera"`).

> [!success] Un solo editor abierto a la vez (2026-08-21, a pedido explícito: "cuando se abre un archivo kpaint se debe abrir el editor, 1 solo")
> A diferencia del resto de KneOS (cada ícono abre su propia `Window` independiente, pueden convivir varias a la vez), KPaint fuerza un único editor: abrir un `.kp`/`.kpaint` mientras YA hay otro abierto no crea una segunda ventana — carga el archivo clickeado adentro de la misma, pidiendo confirmar el descarte si hay cambios sin guardar (mismo `AlertWindow` que ABRIR/BORRAR). Sin cambios sin guardar, carga directo, sin ningún aviso.
>
> **`static _instanciaActiva`** (campo estático de la clase, compartido por `kp` y `kpaint` por igual): apunta siempre a la ÚNICA instancia dueña de la `Window` real en cada momento, sin importar qué ícono la abrió originalmente ni qué archivo esté mostrando ahora — se limpia sola al cerrar esa ventana (`onClose`).
>
> **Overrides de `File`** — `abrirVentana()` y `toggleVentana()`:
> - `abrirVentana()`: si `_instanciaActiva` sigue viva (abierta o minimizada) y no es la que corresponde a este ícono, delega en `_redirigirAVentanaAbierta(otra)` en vez de abrir una ventana propia.
> - `toggleVentana()` (override necesario, no alcanza con `abrirVentana()` solo): el gate original de `File` (`this.window.estaAbierta()`) asume "mi propia `Window` está abierta" == "lo que se ve ahí soy yo" — bajo este modelo eso deja de valer, porque el `Window` realmente abierto puede pertenecer a OTRA instancia (la primera que se abrió) mostrando el contenido de esta. Sin este override, re-clickear el ícono cuya instancia es la dueña de la ventana compartida (pero que ya no muestra su propio contenido, por haber sido redirigida) solo la minimizaba en vez de cambiar el contenido — bug real encontrado con Playwright, no solo teórico.
>
> **`_otraMuestraEsteContenido(otra)`**: la comparación no es tan simple como `otra === this` (ver bug de arriba). Usa `_mostrandoLanzador` (booleano nuevo, no confundir con `_esLanzador` que es identidad fija de la instancia): true mientras se esté mostrando el lienzo en blanco del lanzador, false en cuanto se carga contenido real (`_cargarDibujo`/`_cargarArchivo`/primer GUARDAR del lanzador), true de nuevo tras `_resetALanzador()`. Sin este flag, `_origen === null` es ambiguo — vale tanto para "mostrando mi propio `.kp` ya guardado" como para "mostrando el lanzador en blanco".
>
> Verificado con Playwright (tres casos): (1) abrir B con A sucio → alerta + al confirmar, B carga en la misma ventana. (2) reabrir A sin cambios sin guardar en B → carga directo sin alerta. (3) re-clickear el ícono dueño de la ventana mientras muestra el contenido de otro ícono → cambia de contenido en vez de solo minimizar.

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

## Abrir / Guardar como (`_origen`, 2026-08-20, rediseñado 2026-08-21)

`_origen` es `null` mientras se edita el `kp` propio de este ícono (`this.id`), o `{ tipo: "kp"|"img", id }` si se cargó otro archivo con ABRIR — GUARDAR mira acá para saber a dónde escribir, no siempre a `this.id`.

- **ABRIR** (`_toggleSelectorAbrir`): popup anclado bajo la barra de paleta, listando **todo** `kp`/`img` del sistema vía `window.iconServices.getIcons()` (no solo lo ya cargado en memoria — mismo criterio que [[TaskbarSearch]]). Toggle: un segundo click en ABRIR con el popup ya abierto lo cierra; si el popup abierto era el de GUARDAR COMO (ver abajo), lo cierra y abre el de ABRIR en su lugar (mutuamente excluyentes, nunca los dos a la vez — `popup._deQue` marca de cuál se trata). Si hay cambios sin guardar en el dibujo actual, pide confirmar antes con un `AlertWindow` (ver "Guardado" más abajo). Elegir un ítem llama a `_cargarArchivo(id, ext, nombre)`:
  - `ext === "kp"` → `KPaintServices.getDrawing(id)`, los índices de paleta se copian tal cual.
  - `ext === "img"` → `CameraPhotoServices.getPhoto(id)` (bytes de gris 0-255) cuantizados a la paleta de 8 colores con `grisAIndice` (`model/kpaintPalette.js`, redondeo directo — sin dithering, alcanza para 64×64). Una foto con más de 8 tonos reales sale posterizada al cargarla: trade-off esperado de editarla con la paleta de 8 colores de KPaint, no un bug.
  - En ambos casos: reemplaza `this._pixels`, resetea el historial de deshacer y el snapshot `_pixelesGuardados` a ese estado, resetea `_avisoConversionAceptado` a `false` (ver "Guardado"), y actualiza `_origen`/`_nombreOrigen`, el footer y el label del botón GUARDAR.
- **GUARDAR COMO ▾** (`_toggleSelectorGuardarComo`, reemplaza los botones sueltos NUEVO KP/EXPORTAR del diseño anterior): popup con dos opciones estáticas (sin fetch, a diferencia de ABRIR) — "Nuevo dibujo (.kp)" y "Como imagen (.img)" — que llaman a `_crearNuevoKp()`/`_exportarComoImagen()` respectivamente. Mismo patrón de toggle/exclusión mutua que ABRIR, reusa sus mismas clases CSS (`.kpaintAbrirPopup`/`.kpaintAbrirItem`) y `_cerrarSelectorAbrir`.
  - **NUEVO KP** (`_crearNuevoKp`): crea un `kp` **nuevo** en el escritorio con el dibujo actual (sin conversión de paleta — ya es el mismo formato). Pensado para "bifurcar" un dibujo cargado con ABRIR en un archivo propio, en vez de seguir pisando el original. **Nunca toca `_origen`**.
  - **EXPORTAR** (`_exportarComoImagen`): siempre crea un `img` **nuevo** en el escritorio (mismo camino que `Camera._guardarFoto`: `buscarEspacioVacio()` + `crearIcono(..., edit=false)`), con `this._pixels` convertido a grises (`indiceAGris`). **Nunca toca `_origen`**: es una copia en otro formato — los próximos GUARDAR siguen yendo a donde iban antes de exportar.
  - Ambas comparten el nombre por defecto (`_generarNombreCopia()`, `"Dibujo <fecha>"` — mismo formato que `Camera._generarNombreFoto`); la extensión final (`kp` vs `img`) la decide `crearIcono`, no el nombre.
- **GUARDAR** (`.kpaintIcon--primario`, acción destacada de la barra — fondo sólido permanente, no solo al `:hover` como el resto): ver "Guardado" más abajo para el comportamiento completo.

> [!info] `_origen` no sobrevive un cierre/reapertura de la ventana
> Cerrar la ventana no resetea `_origen` en memoria (la instancia de `File` sigue viva, ver `Window.cerrar`), así que reabrirla sigue mostrando/guardando lo último cargado con ABRIR. Para "volver" al `kp` propio de este ícono hay que abrir otra instancia (otro ícono `kp`, o el mismo desde cero) — no hay un botón "cerrar archivo cargado" dedicado; no se pidió y hubiera sumado una tercera dimensión de estado sin necesidad clara.

> [!bug] Reabrir la ventana podía mostrar un dibujo desactualizado (encontrado y corregido 2026-08-20, mismo bug que [[ImgFile]])
> `_cargarDibujo()` (la carga inicial del `kp` propio de este ícono al primer `_crearContenido()`) se gateaba con un flag `_cargado`, igual que el `ImgFile._cargarFoto` de antes de su fix: una vez cargado, cerrar y reabrir esta misma ventana (mismo `File` persistido en `window.archivosAbiertos`) nunca volvía a pedir nada, solo redibujaba `this._pixels` tal cual habían quedado en memoria. Eso era inofensivo mientras nada más pudiera tocar ese `kp` — pero dejó de serlo en cuanto una SEGUNDA ventana de KPaint puede abrir ese mismo archivo con ABRIR y guardarle encima (ver arriba): la ventana "dueña" del ícono, si ya se había abierto una vez, seguía mostrando la versión vieja.
>
> **Fix:** se sacó `_cargado`. `_cargarDibujo()` ahora también resuelve `_origen` (no solo el `kp` propio) y `_crearContenido()` la llama sin condición en cada apertura — mismo tratamiento que `ImgFile._cargarFoto`: redibuja lo que haya en memoria de una para no dejar el canvas en blanco mientras se espera la respuesta, y la pisa apenas el fetch fresco vuelve.

## Guardado (`_guardar`, modelo tipo Paint tradicional — reescrito 2026-08-21)

A diferencia del diseño original (GUARDAR vaciaba el canvas al terminar, ver el bug documentado abajo), el modelo actual es el de cualquier editor de imágenes: **GUARDAR nunca borra el canvas**, el dibujo se queda ahí para seguir editándolo. `_pixelesGuardados` (snapshot de `this._pixels` tomado en cada carga/guardado exitoso) es la base de `_hayCambiosSinGuardar()` (compara byte a byte contra `this._pixels`).

- **ABRIR y BORRAR** (las dos acciones que reemplazan/destruyen el contenido actual) pasan por `_confirmarDescartarCambios(mensaje, accionConfirm)` primero: si `_hayCambiosSinGuardar()` es `true`, un `AlertWindow` (mismo componente que usan [[RecycleBin]]/[[KneChat]] para sus confirmaciones) pide confirmar antes de ejecutar `accionConfirm`; si no hay nada sin guardar, ejecuta directo.
- El footer (`_actualizarIndicadorOrigen`) siempre muestra `Editando: <archivo>` + un `*` mientras `_hayCambiosSinGuardar()` sea `true` — ese estado nunca queda implícito. La misma función actualiza también el label del botón GUARDAR (`_actualizarBotonGuardar`): `"GUARDAR"` a secas si se edita el `kp` propio de este ícono, o `"GUARDAR → <archivo>"` si `_origen` apunta a otro — el destino de la escritura queda visible en el botón mismo, no solo en el footer (más chico y lejos del botón que en realidad se clickea).
- **Conversión con pérdida sobre un `img`**: guardar sobre un `_origen` de tipo `img` cuantiza a la paleta de 8 colores (mismo `indiceAGris` que EXPORTAR) — pérdida real de detalle sobre una foto que no se dibujó en KPaint. `_guardar()` chequea `_avisoConversionAceptado` (booleano, `false` por default y en cada `_cargarArchivo` nuevo): si es la primera vez que se guarda sobre ese origen, interpone un `AlertWindow` de confirmación antes de llamar a `_ejecutarGuardado`; una vez aceptado, los siguientes GUARDAR sobre el mismo origen ya no vuelven a preguntar. El guardado real vive en `_ejecutarGuardado(id)`, separado de `_guardar()` justo para que esta advertencia pueda interponerse sin duplicar la lógica de guardado.
- **Cuerpo del guardado** (`_ejecutarGuardado`): si `_origen?.tipo === "img"`, convierte a grises y llama `CameraPhotoServices.savePhoto` — **nunca** crea un `kp`. Si no, `KPaintServices.saveDrawing(id, pixels)` contra `_origen?.id ?? this.id` — si `_origen` apunta a OTRO `kp`, tampoco se crea nada nuevo, se edita ese archivo en el lugar. `_actualizarColumnaLista` (tamaño/fecha en la vista de lista) solo corre si se guardó en `this.id`. En cualquier caso, éxito → texto de estado en el footer ("Guardado"/"Error al guardar", 2s, `_mostrarEstadoGuardado`) + `_marcarComoGuardado()` (mueve `_pixelesGuardados` al estado actual, refresca footer/label, y dispara `_destellarBotonGuardar`).
- **Destello del botón** (`_destellarBotonGuardar`, `.kpaintIcon--destello` en CSS): al confirmarse un guardado exitoso, un breve pulso de brillo/escala (`@keyframes`, nunca `text-shadow`) en el propio botón GUARDAR — refuerza la señal donde está la atención del usuario (el botón que acaba de clickear), no solo en el texto transitorio del footer que puede quedar fuera de foco. Se fuerza un reflow (`void el.offsetWidth`) antes de reagregar la clase, para poder reiniciar la animación si se guarda dos veces seguido antes de que termine la anterior.

> [!bug] Diseño original: GUARDAR vaciaba el canvas tras un guardado exitoso (agregado 2026-08-20, revertido 2026-08-21)
> El primer diseño de GUARDAR llamaba a `_borrarTodo()` inmediatamente después de un guardado exitoso, para "arrancar el próximo dibujo en blanco sin cerrar/reabrir la ventana". A pedido del usuario, ese comportamiento se sacó por completo: además de no calzar con el modelo mental de un editor de imágenes (guardar no debería vaciar el lienzo), combinado con GUARDAR apuntando a `_origen` significaba que un segundo click en GUARDAR sin dibujar nada nuevo sobreescribía silenciosamente ese archivo ajeno con un lienzo vacío. El modelo actual (arriba) resuelve ambos problemas a la vez.

## CSS (`kpaint.css`)

Cuatro franjas verticales: `.kpaintToolbar` (herramientas + tamaño de pincel) → `.kpaintPaletteBar` (swatches + ABRIR/DESHACER/BORRAR/GUARDAR COMO/GUARDAR) → `.kpaintCanvasWrap` (canvas centrado) → `.kpaintFooter` (dos secciones: `.kpaintFooterOrigen` a la izquierda con ellipsis si el nombre es largo, `.kpaintFooterEstado` a la derecha). `.kpaintCanvas` usa `width: 100%; max-width: 420px; aspect-ratio: 1/1` (mismo criterio que `.imgCanvas`, ver [[ImgFile]]) + `touch-action: none` para que arrastrar sobre el canvas no dispare scroll/zoom táctil. `.kpaintAbrirPopup`/`.kpaintAbrirItem` (compartidas por ABRIR y GUARDAR COMO, ver arriba) anclan igual que `.txtAiPopup` (ver txt.css): `position: absolute; top: 100%` sobre `.kpaintPaletteBar` (`position: relative`).

Los botones de herramienta (`.kpaintTool`) siguen el verde monocromático del resto de la UI (a diferencia de `.kpaintSwatch`, que sí sale de la paleta — ver arriba): son controles, no colores. Pincel/goma/balde/línea van en texto plano (LAPIZ/GOMA/BALDE/LINEA, `.cameraBoton` como referencia — no hay íconos de esas acciones en `sources/accions/`); las seis formas geométricas se identifican con un ícono dibujado en **CSS puro** (`::before`, sin ningún asset SVG nuevo) en vez de texto — el ícono literalmente ES la forma que van a dibujar: un `<div>` cuadrado/rectangular (`width`/`height`), un círculo/elipse (`border-radius: 50%`), un triángulo (el truco clásico de `border` transparente + un lado sólido) y una estrella de 5 puntas vía `clip-path: polygon(...)`. `.kpaintTool--activo` reusa el mismo invert que `:hover` (`background: var(--primary-color); color: var(--primary-background)`) pero sostenido, para marcar la herramienta seleccionada sin depender de que el mouse siga encima. ABRIR/DESHACER/BORRAR/GUARDAR COMO/GUARDAR (`.kpaintIcon`) son todos texto plano — mismo motivo que las cuatro primeras herramientas: no hay íconos de open/export/undo/trash/save que calcen con `sources/accions/`.

> [!info] `.kpaintIcon--primario`/`.kpaintIcon--destello` (2026-08-21) — GUARDAR como acción destacada
> Rediseño del flujo de guardado, a pedido del usuario ("no me convence la UX del flujo de guardado"): antes GUARDAR/NUEVO KP/EXPORTAR eran tres `.kpaintIcon` del mismo peso visual, sin ninguna jerarquía entre "sobrescribir" y "crear una copia", y GUARDAR se confundía visualmente con BORRAR (acción destructiva). `.kpaintIcon--primario` le da a GUARDAR fondo sólido permanente (no solo al `:hover`) + `max-width`/ellipsis para el label con destino (`GUARDAR → archivo.ext`). `.kpaintIcon--destello` es la animación de pulso (`@keyframes kpaintDestelloGuardar`, brillo/escala vía `filter`/`transform`, nunca `text-shadow`) que se dispara en `_destellarBotonGuardar` tras cada guardado exitoso.

## Registro (sí está en el menú "Nuevo")

`kp` está en `model/iconSrc.js`, `formato.js` (`TIPOS.kp = "Dibujo"`), y — a diferencia de `img` — sí en el submenú "Nuevo" de `Folder._abrirSubMenuCrear` y `ContextMenuManager._abrirSubMenuCrear` (un lienzo en blanco tiene sentido, a diferencia de una foto en blanco): mismo patrón que `txt`/`fld`. No está en `filesUndeletable.js` (se puede borrar como cualquier archivo de usuario).

> [!info] También sembrado en `defaultFiles.js` (2026-08-20) — pero sigue siendo un archivo de usuario común
> A diferencia del resto de entradas de `defaultFiles.js` (Doom/Kmd/Kfruit/KneAI/Maxwell/RecycleBin/Calculator/User/BlackJack/Hangman/FlipCoin/Kdle/CarRace/Tetris/KneChat/Config — todas "fijas": `filesUndeletable` + fuera del menú "Nuevo"), `KPaint` se agregó a `defaultFiles` (`espacio65`, nombre "KPaint") solo para que un escritorio nuevo arranque con un lienzo a mano, sin volverse `filesUndeletable` ni sacarse del menú "Nuevo" — se puede borrar el ícono default y crear otros `kp` libremente, es un lienzo en blanco precargado, no una app-herramienta singleton.

## Persistencia

`kpaint_files` (1:1 con `files` por `id_icon`) — ver [[Módulo KPaint]].
