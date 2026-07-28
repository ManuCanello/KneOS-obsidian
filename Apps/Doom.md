---
tags:
  - portfolio/kneos
  - apps
---

# Doom

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Doom.js` — extiende [[File]]. Extensión `"exe"`, `src = "/apps/doom/doom.html"` (único caso con `src` no nulo entre las apps), ícono `sources/icon/doom.png`.

> [!abstract] Qué hace
> Ventana que ejecuta el videojuego DOOM emulado en el navegador vía DOSBox (WebAssembly).

## Constructor(name)

Solo llama `super()` con los parámetros fijos; no agrega estado propio.

## Funciones

- **`_crearContenido()`**: crea `div.doomContainer` con `id="dosbox"`; en un `setTimeout(…, 0)` (asegura que el elemento ya esté en el DOM) instancia `new Dosbox({ id: "dosbox", onload, onrun })`. En `onload`, llama `dosbox.run("https://js-dos.com/cdn/upload/DOOM-@evilution.zip", "./DOOM/DOOM.EXE")` — descarga el bundle ZIP con el juego y el emulador DOS, y ejecuta el `.EXE`. `onrun` vacío.

## Librería externa

**js-dos** (`js-dos-api.js`, cargado globalmente vía `<script>` en `public/KneOS/index.html`), expone la clase global `Dosbox` — emulador DOS en el navegador (dosbox compilado a WASM/JS).

Sin servicio de persistencia (no interactúa con ningún módulo de [[Frontend Model Services Utils]]).
