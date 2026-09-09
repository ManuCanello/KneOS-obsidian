---
tags:
  - portfolio/kneos
  - apps
---

# Kpool

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Kpool.js` (clase `Kpool`) — extiende [[File]]. Extensión `"kpool"`, ícono propio `sources/appIcon/pool.svg`, `src = null`. Se crea como hijo de la carpeta "Juegos" en el primer inicio (`defaultGameFiles`, igual que Truco/Tetris/etc.), no como ícono raíz del escritorio.

> [!abstract] Qué hace
> Pool 8-ball clásico, **1 contra 1 en la misma PC, por turnos** (sin modo online, a diferencia de [[Truco]]). Menú principal en el mismo estilo ASCII que el resto de las apps de juego (logo "KPOOL", una sola opción "Iniciar Juego"). Reglas clásicas: al primer bochazo legal y prolijo se asignan lisas/rayadas a cada jugador; hay que limpiar el grupo propio y embocar la 8 al final para ganar — embocarla antes de tiempo, o junto con la blanca, pierde la partida. Sin backend: no hay keybinds que configurar (juego 100% mouse) ni leaderboard que persistir.

## Constructor(name)

`super()` con `FileType.GAME` y un `size` inicial de `3_250_000` bytes (~3.1 MB, mismo comentario que [[Kfruit]] — incluye el motor físico planck). `body = null`, sin ningún services propio (a diferencia de Kfruit/Tetris, que sí llaman a un backend para keybinds/scores).

## Bolas (`BALLS`, modelo `KpoolBall`)

Mismo patrón que `KfruitFruit` (ver [[Frontend Model Services Utils#Model|KfruitFruit]]), pero campos propios: `number`, `name`, `type` (`"cue"|"solid"|"stripe"|"eight"`), `final` (solo `true` en la 8, señala "bola que termina la partida" — el mismo rol que `final` en la Sandía de Kfruit), `src`. 15 bolas numeradas + `CUE_BALL` (blanca, `number: 0`) aparte.
>
> **Sin campo `color`** (2026-09-09): a diferencia de `KfruitFruit`, no guarda un color por bola — el pedido explícito fue que las bolas sean **monocromáticas** como el resto de KneOS (ver `kneos-conventions`), así que el modelo se achicó en vez de dejar 15 colores hardcodeados sin usar. Todo se dibuja en el color de sistema vigente (ver "Render" más abajo); lisas vs. rayadas se distingue por una línea horizontal sobre las rayadas, no por color. La blanca **mide exactamente lo mismo que el resto** (mismo `RADIUS`, sin relleno especial) — la única marca que la distingue es no tener número, igual que en una mesa real.

## Funciones de UI (menú)

- `_getLogoLines()`: a diferencia del ASCII art dibujado a mano de Kfruit, usa el patrón más simple de `Truco._getLogoLines()` — un diccionario `FONT` de glifos 5×5 por letra (`K`/`P`/`O`/`L`) mapeado sobre `"KPOOL"`.
- `_mostrarMenu()`: logo + una sola opción, "INICIAR JUEGO (1 VS 1)".
- `_crearPanelJugador(numero)`: HUD lateral por jugador (título, tipo asignado o "SIN DEFINIR", lista de números propios ya embocados).
- `async _initGame()`: arma `.kpool-juego` (panel jugador 1 + centro con turno/canvas/ayuda/volver + panel jugador 2), llama `_iniciarJuego` en un `setTimeout(...,0)` como Kfruit.

## Motor del juego — `_iniciarJuego(canvas, turnoEl, paneles)`

Mismo motor **planck** que [[Kfruit]] (`World`, `Circle`, `Chain`), pero mesa top-down en vez de caída por gravedad:

- **Mundo físico**: `World({gravity:{x:0,y:0}})` — sin gravedad. Bolas con `RADIUS=1.6` (bajado de 2 el 2026-09-09, más chicas — todas iguales, blanca incluida), `density: 0.35` (`BALL_DENSITY`, livianas) y `restitution: 0.99` entre sí (casi perfectamente elástico, para que reboten de verdad). `POCKET_RADIUS` se deriva de `RADIUS` (`RADIUS*2.2`) para no desalinearse si el tamaño de bola vuelve a cambiar.
  > [!info] Frenado: fricción de rodadura constante, no `linearDamping` (2026-09-09)
  > Primera versión usaba `linearDamping`/`angularDamping` de planck — un freno *exponencial* (proporcional a la velocidad actual): pierde la mitad de la velocidad cada cierto tiempo fijo, lo que se siente como un frenazo fuerte apenas se tira y después una cola larga arrastrándose casi sin moverse. Reemplazado por un freno manual en el loop (`ROLLING_FRICTION`, unidades/s² constantes, restadas a la rapidez de cada bola en cada `world.step`) — el mismo modelo que el roce real de rodadura sobre el paño (fuerza ~constante, no proporcional a la velocidad). Con esto una bola tirada fuerte viaja más uniforme y se detiene del todo en un tiempo finito y predecible (`velocidad/ROLLING_FRICTION`) en vez de decaer para siempre sin llegar nunca a cero exacto. `ROLLING_FRICTION` ajustado el mismo día: 70 → 24 → **35** (valor final). 70 (el valor inicial, aunque ya no exponencial) seguía frenando demasiado rápido — "de golpe"; 24 quedó del otro lado (demasiado lento/resbaladizo). 35 es el punto intermedio pedido. Un tiro fuerte igual resuelve el turno en pocos segundos porque la energía se reparte entre las 16 bolas y las bandas restan algo en cada rebote, no solo por este freno.
- **Mesa**: un único cuerpo estático con un `Chain` cerrado (rectángulo, `restitution: 0.94`, subida desde 0.75 → 0.88 → 0.94) como banda. Las 6 buchacas (4 esquinas + 2 laterales) **no son fixtures físicas**: son puntos lógicos (`POCKETS`) contra los que cada bola chequea distancia en cada frame del loop — más simple que sensores + contactos, y suficiente porque no hace falta que la bola "caiga" físicamente.
- **Captura de bola embocada**: si `distancia(bola, buchaca) < POCKET_RADIUS`, la bola se "aparca" (`parked`, un `Set<Body>`) — se le pone velocidad 0 y se la manda a `{x:9999,y:9999}` en vez de destruir el `Body`. Evita la complejidad de destruir/recrear cuerpos a mitad del `world.step()` (a diferencia de la fusión de frutas en Kfruit). La blanca es la única que puede volver a entrar en juego: si fue la embocada, `reposicionarBocha()` la reactiva (`parked.delete`) y la reposiciona en el punto de salida, buscando un lugar libre si hay una bola encima.
- **Contacto** (`world.on("begin-contact")`): un único caso, mucho más simple que Kfruit — marca `cueTocoBocha = true` si la blanca chocó con cualquier otra bola. No hay detección de "le pegó primero a la bola equivocada" (foul clásico del 8-ball real) — simplificación consciente, ver más abajo.
- **Tiro con mouse**: `mousedown` cerca de la blanca (radio de tolerancia `RADIUS*10`) arranca el arrastre; la dirección/potencia del tiro salen de `posición_blanca − posición_mouse` en el momento de soltar (`mouseup`), no de dónde se hizo el click inicial — el efecto visual es "estirar un elástico" desde la blanca. `IMPULSE_SCALE=42` (subido desde 25 el 2026-09-09 — el valor original se sentía débil/pesado incluso a arrastre máximo) calibrado para que un arrastre moderado ya cruce buena parte de la mesa.
- **Fin de tiro**: el loop chequea cada frame si todas las bolas (aparcadas o con `getLinearVelocity().length() < STOP_EPS`) están quietas; recién ahí llama a `resolverTiro()`.
- **`resolverTiro()`** — reglas de 8-ball:
  1. Si se embocó la 8: gana el jugador actual solo si ya tenía su grupo completo (`pocketedByType[tipo].size === 7`) y no embocó la blanca en el mismo tiro; si no, pierde y gana el rival (embocarla antes de tiempo, o junto con la blanca, es derrota instantánea).
  2. Si ningún jugador tiene grupo asignado todavía y este tiro embocó una o más bolas propias del mismo tipo, se asigna lisas/rayadas ahí (mesa "abierta" hasta entonces).
  3. Repite turno si embocó alguna bola de su propio tipo (o cualquiera, si la mesa seguía abierta); si no, o si embocó la blanca, o si no tocó ninguna bola, pasa el turno.
- **Render**: sin reusar el `drawShape`/`worldToScreen` genérico de Kfruit (innecesario acá, solo hay rectángulo + círculos) — dibuja la mesa con `strokeRect`, las buchacas con `arc`+`fill`+`stroke`, y cada bola con `arc` + número centrado (`fillText`). Color: **el mismo que el usuario eligió en [[Config]]**, no un verde hardcodeado — `colorSistema()` llama a `leerColorCSS("--primary-color"/"--primary-dim")` (mismo helper de `model/themeColors.js` que ya usan Camera/ImgFile) en cada frame, nunca cachea, así que un cambio de color hecho a mitad de partida se refleja al toque. Solidas/rayadas se distinguen por una línea horizontal sobre las rayadas, no por color.
- **Línea de apuntado**: mientras se arrastra, dibuja una línea punteada desde la blanca en la dirección del tiro (con la potencia acotada a `MAX_PULL`).

> [!info] Simplificaciones deliberadas frente a un 8-ball reglamentario
> No hay "ball in hand" real (la blanca reposicionada tras un scratch va a un punto fijo, no a donde el jugador elija), no hay falta por "pegarle primero a la bola equivocada" (solo se chequea que la blanca haya tocado *alguna* bola), no hay bocha llamada de antemano para la 8, y el armado del triángulo solo respeta la posición reglamentaria de la 8 (centro de la 3ª fila) — el resto de las lisas/rayadas se reparte al azar. Decisión consciente para la primera versión (1 vs 1 local); ver [[Deuda Técnica]] si se pide profundizar las reglas.

## Persistencia

Ninguna — a diferencia de Kfruit/Tetris/Truco, no hay `KpoolServices` ni rutas de backend: no hay keybinds que configurar (todo el control es con mouse) ni un concepto de leaderboard con sentido para una partida 1 vs 1 local.

## Librería externa

**planck** (mismo motor que [[Kfruit]] — ver esa nota para más contexto de la librería en sí).
