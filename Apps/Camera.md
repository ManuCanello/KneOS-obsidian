---
tags:
  - portfolio/kneos
  - apps
---

# Camera

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Camera.js` — extiende [[File]]. Extensión `"camera"`, ícono propio `sources/appIcon/camera.svg` (pixel-art, `currentColor`, mismo patrón que el resto de `sources/appIcon/`), `src = null`. Agregada 2026-08-19; pasó a `ViewWindow` + guardado real en BD el mismo día (ver más abajo — la primera versión descargaba PNGs y mantenía una galería de sesión en memoria, ninguna de las dos cosas existe ya).

> [!abstract] Qué hace
> Cámara de fotos monocromática: toma la imagen de la webcam (`getUserMedia`), la recorta a un cuadrado de **64×64** y le aplica dither Floyd–Steinberg a 4 niveles — el efecto se ve **en vivo en el visor** (no solo en la foto ya sacada, ver `_loopVivo` más abajo), con los 4 tonos armados en runtime como degradé entre `--primary-background` y `--primary-color` (reemplaza la paleta DMG amarillenta original de la Game Boy — pedido explícito de seguir la paleta del propio proyecto, ver [[Reglas]]). Flujo: **Disparar → Foto → Guardar o Tomar otra**. "Guardar" es local a este KneOS (no descarga nada): crea un ícono nuevo tipo `img` en el escritorio (ver [[ImgFile]]) y persiste el array de **grises** de la foto (2026-08-19: bytes 0-255, ya no el color final — ver "Captura + efecto" más abajo) en la tabla `camera_photos` (ver [[Módulo CameraPhoto]]), para que [[ImgFile]] pueda repintarla con el color del tema vigente al abrirla en vez de quedar congelada en el color de captura.

## Constructor(name)

`super(name, "camera", null, "sources/appIcon/camera.svg", FileType.UTILITY, 40_000)` — tamaño elegido libremente, sin archivo real que medir (a diferencia de Maxwell/Doom/Kfruit).

Reemplaza la `Window` por defecto de `File` por una **`ViewWindow`** (`{ width: 340, height: 460, onClose: () => this._detenerCamara() }`) — diálogo chico de "sacar una foto": sin taskbar, sin resize, un solo botón (Cerrar), mismo criterio que [[Calculator]]. `ViewWindow` no reenviaba `onClose` a `Window` hasta esta app (solo desestructuraba `{width, height, clase}`) — se le sumó el passthrough en `core/ViewWindow.js` (cambio de una línea, retrocompatible: nadie más lo pasaba) porque sin eso cerrar la ventana no apagaría los tracks de `MediaStream` y el navegador dejaría la webcam "en uso" con la ventana ya cerrada.

## Visor en vivo (`_loopVivo`) vs. `<video>` fuente

El `<video>` real (`this._video`) **nunca se muestra** — vive siempre `display:none` (clase `cameraVideoFuente`), solo se usa como fuente para `drawImage()`. Lo que el usuario ve es `this._canvasVivo`, un `<canvas>` de 64×64 real (no escalado por HTML/CSS más que el `image-rendering:pixelated` heredado) que `_loopVivo` — un class field arrow function, para poder pasarlo directo a `requestAnimationFrame` sin perder `this` — redibuja en cada frame llamando a `_procesarCuadro`, el mismo método que usa una captura real. Así el efecto bitmap/dither se ve exactamente igual en vivo que en la foto ya sacada, no es una aproximación con CSS filters.

`_loopVivo` se corta solo (no hace falta `cancelAnimationFrame` en ningún otro lado) chequeando `this._canvasVivo?.isConnected` al principio de cada frame — mismo patrón que `container.isConnected` en [[Maxwell]]. Dentro del frame, si el estado no es `"vivo"` o la ventana está minimizada (`offsetParent === null`), se salta el redibujado (pero se sigue pidiendo el próximo frame) para no gastar CPU ditherizando algo que no se ve.

## Captura + efecto (`_procesarCuadro` / `_aplicarEfectoGameboy`)

- `_procesarCuadro(ctx)` es el método compartido entre el visor en vivo y una captura real (`_capturarFrame` le pasa el contexto de un `<canvas>` descartable en vez del de `this._canvasVivo`): recorta el cuadrado central del feed de video (que casi nunca es 1:1), lo espeja horizontal (`ctx.scale(-1,1)`) para que coincida con el visor tipo selfie, y le aplica el efecto bitmap. Devuelve lo que le devuelve `_aplicarEfectoGameboy` (ver abajo) — `_loopVivo` ignora ese valor, `_capturarFrame` lo necesita.
- **Dither Floyd–Steinberg a 4 niveles** (no blanco/negro puro): la luminancia (`0.299R+0.587G+0.114B`) de los 64×64 píxeles se guarda en un `Float32Array` (no `Uint8ClampedArray`) porque el error difundido a los vecinos puede salirse temporalmente de 0-255; recortarlo antes de tiempo arruina el patrón de dither. Cada píxel se cuantiza al nivel más cercano de `[0, 85, 170, 255]` (`GB_NIVELES`), y el error se reparte a los vecinos con los pesos clásicos de Floyd–Steinberg (7/16, 3/16, 5/16, 1/16).
- El índice de nivel (0-3) mapea directo a uno de los 4 tonos RGB de `paleta` (ver "Paleta" abajo) para pintar el `<canvas>` — eso es lo que se **ve**. Por separado, `GB_NIVELES[idx]` (el byte de gris crudo, no el índice) se vuelca a un segundo `Uint8Array grises` en el mismo recorrido — eso es lo que se **persiste**.
- `_aplicarEfectoGameboy(ctx)` devuelve ese `Uint8Array grises`. `_capturarFrame()` (solo al disparar, nunca en `_loopVivo` — no hace falta recalcularlo 60 veces por segundo) hace `Array.from(this._procesarCuadro(ctx))` — necesita un Array plano, no un `Uint8Array`, porque `JSON.stringify` serializa un TypedArray como objeto (`{"0":1,"1":2,...}`) en vez de array, y eso rompería `savePhoto`. `_pixelsADataUrl(pixels)` reconstruye un `<canvas>` descartable solo para mostrar la vista previa inmediata en `<img class="cameraFoto">` — traduce cada gris a color con `mezclarTono` (ver "Paleta" abajo), no guarda nada por su cuenta.
>
> **Hasta 2026-08-19** existía un método `_extraerColores(ctx)` que releía el `<canvas>` ya coloreado y lo re-codificaba a hex — quedó **eliminado**: como `_aplicarEfectoGameboy` ya calculaba el índice de nivel de cada píxel para el dithering, alcanzaba con volcar `GB_NIVELES[idx]` en el mismo recorrido en vez de pintar y después releer/re-parsear el resultado.

## Paleta (`_construirPaleta`) y repintado en caliente de la preview (`_alCambiarTema`)

`_construirPaleta()` arma los 4 tonos RGB de `GB_NIVELES` vía `mezclarTono(nivel / 255)` — helper compartido con [[ImgFile]] en `model/themeColors.js` (2026-08-19; antes era un método privado `_leerColorCSS`/interpolación inline acá mismo, se subió al módulo de tema para no duplicar la lectura de CSS vars entre las dos apps). `mezclarTono(t)` interpola linealmente entre `--primary-background` y `--primary-color`, leídos en runtime (`getComputedStyle`) en cada llamada, nunca cacheados.

> [!info] Recalculada en cada frame, no cacheada (2026-08-19, ver [[Config]])
> Hasta que existió [[Config]] (color fijo de fábrica todo el proyecto), `_construirPaleta()` se llamaba una sola vez en el constructor y el resultado se guardaba en `this._paleta`. Ahora que el color puede cambiar en caliente durante la sesión, `_aplicarEfectoGameboy(ctx)` la recalcula **al principio de cada llamada** (`const paleta = this._construirPaleta();`, ya no hay `this._paleta`) — así, si el usuario abre Config y cambia de color con Camera todavía abierta y en `"vivo"`, el próximo frame de `_loopVivo` ya ditheriza con el color nuevo, sin hacer falta cerrar y reabrir la app. El costo extra es insignificante frente al resto del dither Floyd–Steinberg sobre 4096 píxeles que ya corre en cada frame.

El estado `"vivo"` se resuelve solo, como arriba — pero el estado `"foto"` (preview de una captura recién sacada, todavía sin guardar) no pasa por `_loopVivo`, así que necesita su propio mecanismo: `_alCambiarTema` (class field arrow function, mismo criterio que `_loopVivo`) está suscripta a `THEME_COLOR_EVENT` (despachado por `applyThemeColor`, ver [[Frontend Model Services Utils#Model|themeColors.js]]) desde `_crearContenido()`, y si `this._estado === "foto"` re-genera `this._fotoImg.src` con `_pixelsADataUrl(this._fotoActual.pixels)` — mismos grises, color nuevo. Se desuscribe en el `onClose` de la `ViewWindow` junto con `_detenerCamara()`.

## Otras apps reactivas al cambio de color

[[ImgFile]] (2026-08-19): una foto ya **guardada** ahora también se repinta con el color vigente, tanto al abrirse como en caliente si el color cambia con la ventana ya abierta — mismo evento `THEME_COLOR_EVENT` y mismo `mezclarTono`, ver [[ImgFile#Dibujado (`_dibujar`) y repintado en caliente (`_alCambiarTema`)|ImgFile]]. Antes de este cambio, una foto quedaba fija en el color con que se sacó para siempre — ver [[Módulo CameraPhoto#Formato cambiado el mismo día: de color final a gris (2026-08-19)]].

## Flujo de decisión tras disparar (`_setEstado`)

Máquina de 4 estados (`"iniciando" | "sin-camara" | "vivo" | "foto"`) que gobierna qué se muestra en `.cameraVisor` (`this._canvasVivo` / `<img>` de la foto recién sacada / overlay de texto) y qué botones quedan habilitados:

- **`"vivo"`**: solo Disparar habilitado.
- **`"foto"`**: Disparar se **deshabilita** (a diferencia de la primera versión, que lo dejaba activo) para forzar la decisión explícita entre **Tomar otra** (mismo handler que antes tenía el botón "En vivo": `_setEstado("vivo")`, descarta `this._fotoActual` implícitamente) y **Guardar** (`_guardarFoto`, ver abajo) — pedido explícito: "se debe poder volver a sacar o guardar".

Sin cámara disponible (permiso denegado, sin dispositivo, etc.) no hay ningún mensaje de error "alarmante" — solo el texto neutro "CÁMARA NO DISPONIBLE" en el mismo verde monocromo de siempre, con un botón "REINTENTAR" que vuelve a pedir `getUserMedia` (mismo espíritu que la regla general del proyecto de no meter indicadores de error fuera de paleta, ver [[Reglas]]).

`_iniciarCamara()` es `async` y chequea `this._video?.isConnected` después del `await` de `getUserMedia` — si la ventana se cerró (y quizás se reabrió, generando un `<video>` nuevo) mientras esperaba el permiso del navegador, apaga los tracks del stream recién llegado en vez de dejarlo huérfano. Mismo patrón defensivo que `container.isConnected` en [[Maxwell]].

## Guardar (`_guardarFoto`) — calco de `Kmd.touch`

```js
const espacioId = window.desktopManager?.buscarEspacioVacio();
const espacio = espacioId ? document.getElementById(espacioId) : null;
if (!espacio) return; // sin espacio en el escritorio: se queda como está, sin mensaje de error

const nombre = this._generarNombreFoto(); // "Foto AAAA-MM-DD HH-MM-SS"
const archivo = await window.desktopManager.crearIcono(IMG_ICON_CSS, espacio, "img", nombre, /*edit*/ false);
if (archivo?.id != null) await this._cameraPhotoServices.savePhoto(archivo.id, this._fotoActual.pixels);
```

No hay ningún `import` de `DesktopManager` en `Camera.js` — usa el global `window.desktopManager`, mismo motivo que `Kmd.js` con `TXT_ICON_CSS`/`_cmdTouch`: evitar un ciclo con `model/iconSrc.js`, que ya importa esta clase. Sin feedback de error visible si algo falla (ni espacio libre, ni la persistencia contra el backend): en el `finally` la app vuelve a `"vivo"` igual, el usuario puede reintentar sacando y guardando de nuevo — mismo criterio de "no error message, just stay" del resto de KneOS.

## Ícono

`sources/appIcon/camera.svg` es nuevo (no reutiliza `file.svg` genérico como Maxwell): un `<path>` único con `fill-rule="evenodd"` sobre grilla de 24×24 — cuerpo + visor + un anillo de lente con punto central, todo como agujeros/rellenos del mismo `currentColor` (mismo estilo pixel-art que `knechat.svg`/`flipcoin.svg`).

## Persistencia

Cada foto guardada es un `icon`/`file` real (extensión `img`) más una fila en `camera_photos` — ver [[Módulo CameraPhoto]] y [[ImgFile]]. La app `Camera` en sí no guarda ningún estado propio entre sesiones (no hay galería): `this._fotoActual` es solo la foto recién sacada, pendiente de guardar o descartar.
