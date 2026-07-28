---
tags:
  - portfolio/kneos
  - apps
---

# Kfruit

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Kfruit.js` (clase `KFruit`) — extiende [[File]]. Extensión `"kfruit"`, ícono `sources/icon/kneAi.png` (reutiliza el de KneAI — placeholder, sin ícono propio todavía), `src = null`.

> [!abstract] Qué hace
> Menú principal en modo "terminal ASCII" (logo dibujado con `<pre>`) con tres opciones: **Iniciar Juego** (caída y fusión de frutas con física real, estilo Suika Game/2048 de frutas — se sueltan frutas que caen y, al chocar dos del mismo nivel, se fusionan en la fruta del nivel siguiente sumando puntos, hasta la Sandía), **Configuración** (rebindeo de teclas: mover izquierda/derecha y soltar, 2 teclas por acción) y **Calificaciones** (leaderboard tipo arcade). Al perder, pantalla de Game Over donde se ingresan 3 iniciales para guardar el puntaje.

## Constructor(name)

`super()`; `body=null` (contenedor raíz, reasignado dinámicamente); `_kfruitServices = new KfruitServices()`; keybinds default: `moveLeft=["ArrowLeft"]`, `moveRight=["ArrowRight"]`, `drop=["ArrowDown","Space"]`.

## Frutas (`FRUITS`, modelo [[Frontend Model Services Utils#Model|KfruitFruit]])

11 frutas con radio (`size`), puntos y color propios:

Cereza(1) → Fresa(2) → Uva(3) → Mandarina(4) → Naranja(5) → Manzana(6) → Pera(7) → Durazno(8) → Piña(9) → Melón(10) → **Sandía(11, `final: true`)**.

## Funciones de UI (menú)

- `_crearContenido()`: crea `.mainMenu`, llama `_mostrarMenu()` y `_cargarKeybinds()`.
- `async _cargarKeybinds()`: pide `KfruitServices.getKeybinds()`, parsea strings `;`-separados a arrays.
- `_getLogoLines()`: ASCII-art "KFRUIT" (reusado en menú y en el canvas del juego).
- `_mostrarMenu()`: logo + 3 opciones (`_initGame`, `_mostrarConfiguracion`, `_mostrarCalificaciones`).
- `_mostrarConfiguracion()` + `_capturarTecla(prop, index, btn)`: rebindeo de teclas, persiste con `KfruitServices.updateKeybinds`.
- `async _mostrarCalificaciones()`: pide `KfruitServices.getScores()`, renderiza `{name, score}`.

## Motor del juego — `_iniciarJuego(canvas, puntajeActualValor)`

Usa **planck** (`World`, `Circle`, `Chain`, importado vía import map como `planck@1.5.0`) como motor de física 2D.

- **Mundo físico**: `World({gravity:{x:0,y:-20}})`.
- **Cuerpos estáticos**: `cShape` (contorno "C" del recipiente), `limit` (sensor límite superior, dispara Game Over).
- **Fruta activa (`body`)**: cuerpo estático (se mueve por teclado), fixture `Circle` marcada `isSensor` hasta soltarse; aleatoria entre las **primeras 4** frutas (Cereza/Fresa/Uva/Mandarina).
- **Colisiones** (`world.on("begin-contact")`), 3 casos: **fusión** (mismo nivel → destruye ambas, suma puntos, crea la fruta siguiente si no es `final`), **aterrizaje** de la fruta soltada, **Game Over** (algo distinto a la fruta activa/soltada toca `limit`).
- **`triggerGameOver()`**: idempotente, llama `_mostrarPantallaGameOver`.
- **Render**: Canvas 2D puro, conversión metros↔píxeles (`PPM=6`), `drawShape` según tipo de fixture, `drawTitulo()` dibuja el logo ASCII de fondo.
- **Loop** (`requestAnimationFrame`, `timeStep=1/60`): step físico, destruye/crea cuerpos pendientes, mueve la fruta activa según `moveDir`, renderiza.
- **Controles**: `keydownListener`/`keyupListener` (mueve/suelta según `moveLeft`/`moveRight`/`drop`), `dropBall()` (convierte el cuerpo a dinámico, activa colisión real).
- **`_createCircle(pos, fruitDef)`**: cuerpo dinámico para la fruta resultante de una fusión (a diferencia de `_createBall`, que crea el estático controlable).

## Otras funciones de UI del juego

- `async _crearPanelPuntajes()`: puntaje en vivo + mejores calificaciones.
- `_crearPanelFrutas()`: leyenda de las 11 frutas.
- `_mostrarPantallaGameOver(puntaje, contenedor)`: captura de 3 iniciales (a-z, Backspace, Enter) → `KfruitServices.insertScore(nombre, puntaje)`.
- `async _initGame()`: arma `.kfruit-juego` (paneles + `<canvas>`), llama `_iniciarJuego` en un `setTimeout(...,0)`.

## Persistencia

[[Frontend Model Services Utils#Services|KfruitServices]] → [[Módulo Kfruit]].

## Librería externa

**planck** (`planck@1.5.0`, Box2D portado a JS) — único uso de motor de física del proyecto.
