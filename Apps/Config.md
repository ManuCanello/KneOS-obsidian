---
tags:
  - portfolio/kneos
  - apps
---

# Config

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Config.js` — extiende [[File]]. Extensión `"config"`, ícono propio `sources/appIcon/config.svg` (engranaje, mismo estilo Material-outline de `calc.svg`/`txt.svg`: un único `<path>` `fill="currentColor"`, viewBox 24×24). Agregada 2026-08-19: primera app del panel de "Configuración" del sistema, hoy con un único ajuste.

> [!abstract] Qué hace
> Deja elegir el color de toda la interfaz de KneOS entre 10 opciones (Verde de fábrica, Rojo, Naranja, Amarillo, Azul, Celeste, Rosado, Morado, Turquesa, Blanco — el último agregado 2026-09-02) — una grilla de swatches, cada uno con una muestra del color real + su nombre. Clickear uno repinta **todo** KneOS al instante (no hay botón "Guardar" aparte) y persiste la elección por sesión — ver [[Módulo Theme]].

## Constructor(name)

`super(name, "config", null, "sources/appIcon/config.svg", FileType.SYSTEM, 30_000)` — tamaño elegido libremente, sin archivo real que medir (mismo criterio que [[Camera]]/Calculator). `FileType.SYSTEM` porque es un panel de configuración del sistema, no una utilidad de uso general — a diferencia de Calculator (`UTILITY`).

Reemplaza la `Window` por defecto de `File` por una **`ViewWindow`** (`{ width: 340, height: 420 }`) — diálogo chico de tamaño fijo, sin taskbar ni resize, mismo patrón que Calculator/[[Camera]].

## Cómo funciona el repintado (`model/themeColors.js`)

Todo el mecanismo de color vive en `public/KneOS/js/model/themeColors.js`, no en `Config.js` — Config solo es la UI que lo dispara:

- **`THEME_COLORS`**: objeto `{clave: {label, color, dim}}` para las 10 opciones. `color` pisa **tanto** `--primary-color` como `--primary-glow` (son el mismo valor también para el verde de fábrica, ver `base.css`); `dim` pisa `--primary-dim` a ~80% del brillo de `color` (mismo ratio que `#00ff41`/`#00cc33` de fábrica). `--primary-background` se queda negro sea cual sea el color — es el fondo CRT, no la paleta.
- **`applyThemeColor(clave)`**: `document.documentElement.style.setProperty(...)` de las tres variables — al ser inline en el `<html>`, gana por especificidad a la declaración de `:root` en `base.css` sin tocar ningún stylesheet, y como *toda* regla `var(--primary-*)` del proyecto cuelga de ese mismo elemento, un solo llamado repinta la interfaz entera (íconos SVG con `mask-image` incluidos, ver `utils/iconoStyle.js`). Termina despachando `window.dispatchEvent(new CustomEvent(THEME_COLOR_EVENT))` (2026-08-19) — el único mecanismo de "avisar" que el color cambió; lo escuchan [[Camera]] (preview sin guardar) e [[ImgFile]] (foto ya guardada) para repintarse en caliente sin que Config necesite saber que existen.
- **`DEFAULT_THEME_COLOR = "verde"`**: fallback si `window.themeColorActual` no está seteado o el backend devuelve una clave que ya no existe en `THEME_COLORS`.

> [!warning] Agregar un color acá NO alcanza
> `THEME_COLORS` está duplicado en el backend como `THEME_COLOR_KEYS` (`utils/validation.js`, ver [[Módulo Theme]]) — al agregar "Blanco" (2026-09-02) el swatch aparecía y aplicaba bien en el momento, pero `setColor` fallaba con `400 Color inválido` porque esa segunda lista no se había tocado. Cualquier clave nueva en `THEME_COLORS` tiene que sumarse también a mano en `THEME_COLOR_KEYS`, o la persistencia se rompe en silencio para esa opción (el error solo se ve en la consola del navegador, no hay cartel en la UI).
- **`leerColorCSS(variable)` / `mezclarTono(t)`** (2026-08-19): helpers de mezcla compartidos, subidos acá desde lo que antes era lógica privada de `Camera._leerColorCSS`/`_construirPaleta` — `mezclarTono(t)` interpola entre `--primary-background` y `--primary-color` según `t` (0-1) y es lo que usan tanto Camera (dither en vivo/preview) como ImgFile (repintar una foto guardada) para traducir un byte de gris a color del tema vigente.

`Config._crearContenido()` arma la grilla iterando `Object.entries(THEME_COLORS)`; cada swatch es un `<button>` con un `<span>` de muestra (`background-color` inline, el color real) + un `<span>` de etiqueta — **nunca** el atributo `title` (ver [[Reglas]]), el nombre siempre es texto visible. `.configSwatch--activo` (borde+glow, `var(--primary-glow)`) marca la opción vigente.

> [!info] Única excepción a la paleta monocromática del proyecto
> La regla general de KneOS es que el chrome del sistema nunca usa colores fuera de `--primary-*` (ver [[Reglas]]). La grilla de Config es una excepción deliberada: la muestra de cada swatch necesita mostrar el color real para poder elegirlo — el resto de la ventana (bordes, texto, foco) se queda en `--primary-*` como cualquier otra app.

## Elegir un color (`_elegirColor`)

```js
_elegirColor(clave) {
    if (clave === (window.themeColorActual ?? DEFAULT_THEME_COLOR)) return;

    window.themeColorActual = clave;
    applyThemeColor(clave);
    this._actualizarSeleccion();

    advertirSiFalla(this._themeServices.setColor(clave), `No se pudo guardar el color del sistema (${clave})`);
}
```

`window.themeColorActual` es el cache global en memoria del color vigente (mismo patrón que `window.folderGroupByOptions`/`window.folderViewsOptions`, ver `KNEOS.js`) — se actualiza acá de forma optimista (antes de que la persistencia confirme) para que la UI nunca se sienta lenta; si `setColor` falla, `advertirSiFalla` solo deja un `console.warn`, el color elegido queda aplicado igual (mismo criterio "no error message, just stay" del resto del proyecto).

## Arranque (`KNEOS.js`)

El color persistido se carga y aplica **antes** de crear `DesktopManager`, todavía detrás de `#loadingScreen`:

```js
window.themeColorActual = (await new ThemeServices().getColor()) ?? DEFAULT_THEME_COLOR;
applyThemeColor(window.themeColorActual);
```

Así ningún ícono/ventana llega a pintarse en verde de fábrica para después "saltar" al color guardado — el único otro punto que escribe `window.themeColorActual`/llama `applyThemeColor` es `Config._elegirColor`, en caliente, ya con la sesión andando.

## Ícono de escritorio y borrado

Está en `defaultFiles.js` (`espacio64`, nombre "Config") y en `filesUndeletable` — mismo criterio que el resto de las apps fijas del sistema (Calculator, [[Camera]], KneChat, etc.): no se puede recrear desde el menú "Nuevo" (que solo ofrece `txt`/`fld`), así que borrarla la perdería para siempre.

## Otras apps reactivas al cambio de color

- [[Camera]] (2026-08-19, a pedido explícito del usuario tras el primer pase de Config): su paleta de dither de 4 tonos (`_construirPaleta`, vía `mezclarTono`) se recalcula en cada frame de `_loopVivo` en vez de una sola vez al construirse — si Config cambia el color mientras Camera sigue abierta en `"vivo"`, el próximo frame ya se ve en el color nuevo, sin reabrir la app. La preview de una foto recién sacada sin guardar (estado `"foto"`) se repinta escuchando `THEME_COLOR_EVENT` (`_alCambiarTema`).
- [[ImgFile]] (2026-08-19, segunda vuelta el mismo día): una foto **ya guardada** también se repinta con el color vigente — dejó de guardarse el color final mezclado y pasó a guardarse el gris crudo (ver [[Módulo CameraPhoto]]), así que abrir una foto vieja (o tenerla abierta mientras Config cambia de color) siempre la muestra en el color actual, nunca en el de cuando se sacó.

## Alcance actual

"Por ahora" (pedido explícito del usuario) el único ajuste es el color. Queda **fuera** de esta primera pasada, documentado como deuda conocida en [[Deuda Técnica]]:
- El patrón de puntos del fondo del escritorio (`core/desktop.css`, `background-image` con un SVG `data:` embebido con `#00ff41` hardcodeado) no seguía el tema — no se toca por ahora.
- El wireframe 3D de [[Maxwell]] (`THREE.MeshBasicMaterial({ color: 0x00ff41, ... })`) tampoco — necesitaría leer `--primary-color` en runtime como ya hace [[Camera]] con su paleta.
- Un swatch translúcido hardcodeado en `apps/user.css` (`background-color: #00cc332e`).

## Consumido por / consume

Servicio frontend `ThemeServices` (`public/KneOS/js/services/ThemeServices.js`) — ver [[Módulo Theme]].
