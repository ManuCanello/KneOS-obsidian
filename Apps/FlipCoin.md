---
tags:
  - portfolio/kneos
  - apps
---

# FlipCoin

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/FlipCoin.js` (clase `FlipCoin`) — extiende [[File]]. Extensión `"flipcoin"`, ícono propio `sources/appIcon/flipcoin.svg` (anillo + círculo relleno, moneda), `src = null`. Sin `Window` propia: usa la `Window` completa por defecto de `File` (como BlackJack/Ahorcado/Kfruit), sin tamaño fijo.

> [!abstract] Qué hace
> Port de la app de consola "Flip-Coin" en Java original (clase `App`, `flip-coin.txt`, 2026-08-12). Mismo sorteo 50/50 (`girarMoneda()`: 0=cara, 1=cruz) y la misma animación de giro (9 frames de `_`/`\`/`|`/`/`, 300ms cada uno — port literal, carácter por carácter, de `animacion()`, incluyendo el rebote vertical del glyph). El menú del Java tenía 4 opciones (Jugar/Ver porcentaje/Ver historial/Salir); acá quedan 3 — "Salir" no se portó, cerrar la ventana cumple esa función (mismo criterio que el resto de las apps de juego).

## Diferencias respecto al Java original

- **Persistencia: archivo de texto → tabla en BD**: `grabar()`/`calcularPorcentaje()`/`historial()` del original leían/escribían un archivo en una ruta absoluta hardcodeada (`E:/Descargas/Programacion/JavaGames/Flip-cpin/respuestas.txt` — ni portable ni aplicable en el navegador). Pasaron a la tabla `flipcoin_resultados` (ver [[Módulo FlipCoin]]) — a diferencia del catálogo fijo de [[Hangman]]/[[Kdle]] (`palabras`, no crece), esta tabla sí crece con cada tirada: mismo criterio que el leaderboard `kfruit_score` de [[Kfruit]], global y sin `pc_id`.
- **Elegir cara/cruz por botones**: `pedirUsuario()` (menú numérico `1)Cara 2)Cruz` por Scanner, con reintento si la opción no era 1 ni 2) pasó a `_mostrarElegir()` — dos `<button class="game-boton">`, sin caso de "opción inválida" posible (mismo criterio que BlackJack/Ahorcado: un botón solo existe si la jugada es legal).
- **Animación**: `animacion()` limpiaba la consola e imprimía cada frame (`a1` 5 frames + `a2` 4 frames, cada string con sus propios `\n\t` líderes — eso es lo que movía el glyph de línea en la terminal: 4,3,2,1,0 saltos en `a1`, 1,2,3,4 en `a2`, o sea rebota de abajo hacia arriba y de vuelta abajo mientras rota `_`→`\`→`|`→`/`). Primer intento del port solo rotaba el símbolo en el mismo lugar (perdía el rebote); ahora `FRAMES_ANIMACION` en `FlipCoin.js` son los mismos 9 strings literales de `a1`+`a2` (con `\n`/`\t` incluidos), volcados sin más a un único `<pre class="fcMoneda">` cada 300ms — el `<pre>` reemplaza el cls()+println() de consola, pero el contenido de cada frame es idéntico al Java. `.fcMoneda` tiene `height: 5em` fijo (reserva las 5 líneas del rebote para que el resto de la pantalla no salte) y `tab-size: 2` (el tab de 8 espacios por default quedaba enorme al tamaño de fuente de la app).
- **Resultado**: tras la animación se muestra el símbolo final (`O` para cara, `X` para cruz, con el mismo prefijo `\n\n\n\n\t` que el último frame para no saltar de posición), el texto "Cara"/"Cruz" y "¡Le pegaste!"/"No le pegaste" (`compararResultado()`), con un botón "Volver al menú" en vez de `esperarTecla()`.
- **Menú principal**: mismo patrón que BlackJack/Ahorcado/Kfruit — `this.body` con clases `"app" "game" "flipcoinApp" "mainMenu"`, logo ASCII "FLIPCOIN" en `<pre>` (fuente de bloque 5×5 propia, `_getLogoLines()`, mismo esquema que BlackJack/Ahorcado). A diferencia de Ahorcado (que saca la clase `mainMenu` durante la partida para un layout de mesa no centrado), FlipCoin mantiene `mainMenu` en todas sus pantallas — es lo bastante simple (moneda + mensaje, o una lista) como para no necesitar un layout propio, mismo criterio que las pantallas de configuración/calificaciones de [[Kfruit]].

## `apps/game.css` (estándar compartido)

No define CSS propio para menú/logo/botones — todo eso sale de `.game`/`.mainMenu`/`.game-boton`/`.game-fila-botones` en `apps/game.css` (ver nota de refactor en [[BlackJack]]). `flipcoin.css` solo tiene lo específico de sus pantallas: `.fcMoneda` (el `<pre>` de la animación/resultado), `.fcMensaje`, `.fcResultado`, `.fcPorcentaje` y `.fcHistorial` (lista scrolleable).

## Jugable por teclado + modo consola (2026-08-14)

A pedido explícito (jugar desde `run flipcoin` en [[Kmd]]), los tres menús (`_mostrarMenu`, `_mostrarElegir` Cara/Cruz, "Volver al menú" post-resultado) pasaron a `File._menuTeclado` (ver [[File]]) — Arriba/Abajo o Izquierda/Derecha navega, Enter confirma. Las tres pantallas de solo lectura (`_mostrarPorcentaje`/`_mostrarHistorial`, más "< VOLVER" en `_mostrarElegir`) usan `File._escapeVuelve` — Escape en "< VOLVER" (ahora con el hint "(ESC)" en el texto). Cada pantalla tiene su par en modo consola (`_mostrarMenuConsola`, etc. — `this._modoTerminal`), con un ítem "CERRAR" extra en el menú principal.

**Dos correcciones el mismo día:** la primera pasada le sacó el click a los `<p>`/`<button>` también en el modo desktop — se corrigió sumando `click` a `_menuTeclado`/`_escapeVuelve` (mouse y teclado conviviendo). A pedido explícito del usuario, horas después esa convivencia se revirtió: el modo desktop volvió a ser **mouse-only, como era originalmente** — `_mostrarMenu()`, `_mostrarElegir()`, `_jugar()`, `_mostrarPorcentaje()` y `_mostrarHistorial()` volvieron a `addEventListener("click", ...)` directo en cada `<p>`/`<button>`, sin `File._menuTeclado`/`_escapeVuelve` ni ESC/flechas/Enter (los `_mostrarXConsola()` del modo consola no se tocaron, siguen usando esos helpers).

## Persistencia

Tabla `flipcoin_resultados` (ver [[Módulo FlipCoin]]) — cada tirada de "JUGAR" inserta una fila; "VER PORCENTAJE"/"VER HISTORIAL" leen esa tabla. A diferencia de [[Hangman]] (solo lectura desde la app), acá sí hay una escritura por partida jugada.
