---
tags:
  - portfolio/kneos
  - apps
---

# ImgFile

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/ImgFile.js` — extiende [[File]]. Extensión `"img"`, ícono propio `sources/appIcon/img.svg` (pixel-art, `currentColor`, marco de foto con "montaña + sol" — mismo estilo `fill-rule="evenodd"` que `camera.svg`), `src = null`. Agregada 2026-08-19, junto con [[Camera]] guardando de verdad en vez de solo descargar un PNG.

> [!abstract] Qué hace
> Visor de una foto sacada con [[Camera]]: lee el array de 4096 bytes de gris (0-255) persistido en `camera_photos` (ver [[Módulo CameraPhoto]]) y lo traduce a un `<canvas>` de 64×64 con `putImageData`, mezclando cada gris con el `--primary-color` **vigente** (`mezclarTono`, ver [[Frontend Model Services Utils#Model|themeColors.js]]). Hasta 2026-08-19 se guardaba el color final ya mezclado (hex) y una foto quedaba congelada para siempre en el color con que se sacó; ahora se repinta con el tema actual tanto al abrirse como en caliente si el color cambia con la ventana ya abierta (ver `_alCambiarTema` más abajo) — ver [[Config#Otras apps reactivas al cambio de color]].

## Constructor(name)

`super(nombre, "img", null, "sources/appIcon/img.svg", FileType.OTHER)` — sin `size` explícito (lo trae `files.size`, actualizado por el backend al guardar el contenido, mismo patrón que [[TxtFile]]).

> [!success] Pisa `this.window` con una `ViewWindow` de tamaño fijo, mismo criterio que Camera (2026-08-19)
> Primera versión dejaba la `Window` completa por defecto de `File` (redimensionable + taskbar), con el razonamiento de que una foto guardada es "un archivo de primera clase" como [[TxtFile]]/[[Maxwell]]. Se revirtió: la imagen nativa es siempre 64×64 (`PHOTO_SIZE`), así que redimensionar la ventana nunca agrega información, solo espacio vacío o un cuadrado más grande — no hay razón real para que sea redimensionable. Ahora `this.window = new ViewWindow(this._generarVentanaId(), this.nombreCompleto, this.icono, () => this._crearContenido(), { width: 420, height: 500, onClose })`, mismo patrón que [[Camera]] (que ya usaba `ViewWindow` desde el principio, ahí sí por ser un diálogo de "sacar una foto" y no un archivo). Efecto secundario esperado de `ViewWindow`: sin entrada en la taskbar y sin botones de minimizar/maximizar, solo Cerrar (ver [[Window y Taskbar#`ViewWindow`]]).
>
> **`onClose` sumado más tarde (2026-08-19, mismo día):** al principio no hacía falta — a diferencia de Camera, no había ningún recurso (stream de cámara) que cerrar. Pasó a necesitarlo con el repintado en caliente de más abajo: `onClose: () => window.removeEventListener(THEME_COLOR_EVENT, this._alCambiarTema)`, para no dejar el listener de cambio de tema colgado en `window` después de cerrar la ventana.

## Carga (`_cargarFoto`) — mismo patrón que TxtFile

```js
async _cargarFoto() {
    this._cargado = true;
    if (this.id == null) return;
    const pixels = await this._cameraPhotoServices.getPhoto(this.id);
    if (pixels?.length !== PHOTO_SIZE * PHOTO_SIZE) return; // sin datos: canvas en blanco, sin mensaje de error
    this._pixels = pixels;
    this._dibujar(pixels);
}
```

> [!bug] Reabrir la ventana dejaba el canvas en blanco (2026-08-19)
> `Window.cerrar()` tira `_ventanaEl` entero del DOM y lo pone en `null` — el próximo `abrir()` corre `_crearContenido()` desde cero, con un `<canvas>` nuevo. `_crearContenido()` solo llamaba a `_cargarFoto()` si `!this._cargado`, y `_cargarFoto()` marca `_cargado = true` en su primera línea — así que cerrar y reabrir la ventana dejaba ese canvas nuevo sin nadie que lo dibuje: `_cargado` seguía en `true` de la primera apertura, así que la carga se saltaba, y como los píxeles nunca se guardaban en ningún lado (solo vivían como variable local `pixels` dentro de `_cargarFoto`), tampoco había nada en memoria para redibujar directo. Contrasta con [[TxtFile]], que sí cachea el contenido cargado en `this._texto` y lo vuelca directo en `_crearContenido()` (`editor.innerHTML = this._texto`) sin depender de que `_cargarContenido()` vuelva a correr.
>
> **Fix:** se agregó `this._pixels = null` en el constructor, y `_cargarFoto()` ahora hace `this._pixels = pixels` antes de dibujar (mismo rol que `_texto` en TxtFile). `_crearContenido()` pasa a: si `this._pixels` ya está en memoria, dibuja directo (`_dibujar(this._pixels)`) sin pedir nada al servidor; si no, y todavía no se cargó ni una vez (`!this._cargado`), recién ahí llama a `_cargarFoto()`. Verificado con Playwright: mismo conteo de píxeles no-negros en el canvas antes y después de cerrar/reabrir la ventana de una foto guardada.

`_crearContenido()` es síncrono (devuelve el `<canvas>` al toque) y dispara `_cargarFoto()` fire-and-forget al final, igual que `TxtFile._crearContenido`/`_cargarContenido` — el flag `_cargado` evita repedir el contenido si la ventana se cierra y se reabre dentro de la misma sesión. Si `pixels` no tiene exactamente 4096 elementos (foto cuyo guardado falló a mitad de camino, o `id` inválido), el canvas queda en negro sólido en vez de mostrar cualquier indicador de error — mismo criterio "no error message, just stay" que el resto de KneOS. Al final también se suscribe al evento de cambio de tema: `window.addEventListener(THEME_COLOR_EVENT, this._alCambiarTema)` — ver más abajo.

## Dibujado (`_dibujar`) y repintado en caliente (`_alCambiarTema`)

```js
_dibujar(pixels) {
    const ctx = this._canvas.getContext("2d");
    const imageData = ctx.createImageData(PHOTO_SIZE, PHOTO_SIZE);
    pixels.forEach((gris, i) => {
        const [r, g, b] = mezclarTono(gris / 255);
        imageData.data[i * 4]     = r;
        imageData.data[i * 4 + 1] = g;
        imageData.data[i * 4 + 2] = b;
        imageData.data[i * 4 + 3] = 255;
    });
    ctx.putImageData(imageData, 0, 0);
}
```

`mezclarTono(t)` (`model/themeColors.js`, 2026-08-19) es la misma interpolación entre `--primary-background` y `--primary-color` que ya usaba `Camera._construirPaleta` para armar su degradé de 4 tonos, subida a un módulo compartido para que ImgFile la reuse sin duplicar la lectura de CSS vars — cada byte de gris `t = gris/255` mapea directo, sin pasar por un array de niveles intermedio.

```js
_alCambiarTema = () => {
    if (this._pixels) this._dibujar(this._pixels);
};
```

Suscripto a `THEME_COLOR_EVENT` (`window`, despachado por `applyThemeColor` — ver [[Frontend Model Services Utils#Model|themeColors.js]]) en `_crearContenido()`, desuscripto en `onClose`. Si [[Config]] cambia el color mientras la ventana de la foto sigue abierta, redibuja directo con `this._pixels` ya en memoria — sin volver a pedirle nada al servidor. Si todavía no cargó nada (`this._pixels` sigue `null`), no hace nada: el propio `_cargarFoto()` va a dibujar ya con el color vigente en cuanto la respuesta llegue.

`PHOTO_SIZE` (=64) viene de `public/KneOS/js/model/cameraPhoto.js`, no de una constante local — así `Camera.js` (al capturar) e `ImgFile.js` (al dibujar) nunca pueden desincronizarse en qué resolución esperan.

## CSS (`img.css`)

`.imgCanvas` usa `width: 100%; max-width: 480px; aspect-ratio: 1/1` — hace falta `width:100%` (no alcanza con `max-width:100%`, que solo limita el achique, nunca agranda el `<canvas>` desde sus 64px nativos) para que la foto se vea escalada dentro del `.imgApp` (`padding:10px`) en vez de diminuta en una esquina. El tope de 480px es vestigial de cuando la ventana era redimensionable (ver arriba) — con la `ViewWindow` fija de 420×500 nunca se llega a ese ancho, pero se dejó tal cual por si el tamaño fijo cambia más adelante.

## Ícono

`sources/appIcon/img.svg`: un `<path>` `fill-rule="evenodd"` sobre grilla de 24×24 — marco rectangular con el interior recortado (hueco = "foto"), y dentro de ese hueco un cuadradito relleno (sol) más tres escalones apilados (silueta de montaña), mismo truco de paridad par/impar que `camera.svg`.

## Registro (no está en el menú "Nuevo")

`img` está en `model/iconSrc.js` (`load: () => import("../apps/ImgFile.js")`) y en `formato.js` (`TIPOS.img = "Imagen"`), pero **no** en `filesUndeletable.js` (una foto guardada se puede borrar como cualquier `txt`/carpeta de usuario, no es un ícono fijo del sistema) ni en el submenú "Nuevo" de `Folder`/`ContextMenuManager` (una imagen en blanco no tiene sentido — un `img` solo nace desde `Camera._guardarFoto`).

## Persistencia

`camera_photos` (1:1 con `files` por `id_icon`) — ver [[Módulo CameraPhoto]]. `ImgFile` es puramente de lectura: nunca escribe, solo `getPhoto`. Lo persistido es gris, no color — el color se resuelve siempre client-side, al dibujar, nunca queda "fijado" en BD.
