---
tags:
  - portfolio/kneos
  - apps
---

# Doom

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Doom.js` — extiende [[File]]. Extensión `"exe"`, ícono `sources/appIcon/doom.png`.

> [!abstract] Qué hace
> Ventana que ejecuta el DOOM shareware original dentro de un iframe, corriendo Chocolate Doom compilado a WebAssembly — 100% local, sin CDN externo.

## Migración 2026-09-21: de js-dos a Chocolate Doom WASM en iframe

Reemplazo completo del motor anterior, a pedido explícito del usuario ("no depender de lo que usa actualmente"). El viejo `Doom.js` usaba **js-dos v3** (2015) — un emulador de DOS completo cargado globalmente vía `<script>` desde `https://js-dos.com/cdn/js-dos-api.js`, que además bajaba el juego (`DOOM-@evilution.zip`) de ese mismo CDN en cada partida. Esa librería vieja generaba tres problemas serios, todos documentados en la versión anterior de esta nota:
- No exponía un `stop()`/`exit()` real — cerrar la ventana dejaba el emulador (y su audio) corriendo en memoria para siempre.
- Registraba sus listeners de teclado directo en `document` vía `eval()`, sin sacarlos nunca — bloqueaban el teclado del resto de KneOS aunque la ventana ya estuviera cerrada.
- No exponía su `AudioContext`, había que interceptar el constructor global para poder cerrarlo a mano.

`Doom.js` tenía ~165 líneas, la mitad de ellas parches (`_hookAudioContext`, `_hookInputCapture`/`_unhookInputCapture`, `_stopGame`) para compensar esas tres carencias de la librería.

**Solución adoptada**: [Chocolate Doom](https://www.chocolate-doom.org/) recompilado a WebAssembly con Emscripten, vendorizado de [`OscarRevollo/doom-wasm`](https://github.com/OscarRevollo/doom-wasm) (GPL-2.0-or-later) — trae el engine ya compilado (`chocolate-doom.js`/`.wasm`/`.data`, no hace falta toolchain de Emscripten para tocarlo) más una página mínima (`index.html`/`main.js`/`styles.css`) diseñada explícitamente para embeberse en un iframe. Se reemplazó el `freedoom2.wad` que trae por default por el **`doom1.wad`** shareware original (v1.9-like, 4.207.819 bytes, header `IWAD` + lumps `E1M1`-`E1M9` verificados) — libremente distribuible desde 1995, bajado de [`Doom-Utils/shareware-collection`](https://github.com/Doom-Utils/shareware-collection) y escrito en el FS virtual de Emscripten en runtime (`FS.writeFile('/iwads/doom1.wad', …)` dentro de `preRun`, con un `mkdir('/iwads')` defensivo porque el orden contra el preload de `freedoom2.wad` no está garantizado) en vez de reconstruir el `.data` — evita necesitar el toolchain de Emscripten solo para cambiar el IWAD.

Todo el bundle (~30MB: `chocolate-doom.data` 28MB + `.wasm` 1.6MB + `doom1.wad` 4.2MB) vive versionado en el repo bajo `public/KneOS/apps/doom/` (ver ARCHITECTURE.md), decisión explícita del usuario sobre mantenerlo commiteado en vez de en una carpeta ignorada por git — prioriza cero dependencia externa y carga instantánea sin red por sobre el tamaño del repo.

**Por qué un iframe resuelve los tres problemas de raíz, sin hooks**: un iframe tiene su propio `document`/`window`/`AudioContext` — un browsing context aparte. `Window.cerrar()` (ver [[Window]]) ya hacía `v.remove()` sobre el contenedor de la ventana; con un `<div>` normal eso solo sacaba el DOM visual y el JS de adentro seguía vivo en el mismo realm que el resto de KneOS, pero con un `<iframe>` esa misma llamada destruye el browsing context completo -- listeners de teclado y `AudioContext` incluidos -- sin necesitar interceptar nada. Verificado con Playwright: 0 iframes en el DOM inmediatamente después de cerrar la ventana.

## Constructor(name)

Llama `super()` con los parámetros fijos, `size` inicial `4_207_819` (tamaño real de `doom1.wad`). Ya no crea una `Window` propia con `onClose` (ver el punto de arriba) — usa la `Window` por defecto de `File`, que solo cablea `onTogglePin`.

## Funciones

- **`_crearContenido()`**: crea `div.doomContainer` conteniendo un `<iframe src="/KneOS/apps/doom/index.html">` (`allow="autoplay"`, `sandbox="allow-scripts allow-same-origin"`, `aria-label` en vez de `title` — ver Reglas). Todo el arranque del juego (fetch de `doom1.wad`, creación del módulo Emscripten, `callMain`) vive en `public/KneOS/apps/doom/main.js`, corriendo dentro del iframe, no en `Doom.js`.

## Verificación (2026-09-21)

Probado primero fuera del repo (clon de `OscarRevollo/doom-wasm` en un scratchpad, con el WAD real swapeado) antes de tocar el proyecto real. Ya integrado, corrido con Playwright contra el server local: abre "Juegos" → "Doom" en el escritorio real, la pantalla de título de DOOM (no Freedoom) carga sin errores de consola, y cerrar la ventana saca el iframe del DOM al instante.

## Assets

`public/KneOS/apps/doom/` (fuera de `js/apps/`, servido directo por `express.static`, no bundleado por esbuild):
- `index.html`, `main.js`, `styles.css` — la página del iframe.
- `doom1.wad` — el IWAD shareware.
- `engine/chocolate-doom.{js,wasm,data}` — el motor prebuilt de `doom-wasm`.

Sin servicio de persistencia (no interactúa con ningún módulo de [[Frontend Model Services Utils]]).
