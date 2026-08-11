---
tags:
  - portfolio/kneos
  - apps
---

# BlackJack

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/BlackJack.js` (clase `BlackJack`) — extiende [[File]]. Extensión `"blackjack"`, ícono propio `sources/appIcon/blackjack.svg`, `src = null`. Sin `Window` propia: usa la `Window` completa por defecto de `File` (como Kfruit/Kmd), sin tamaño fijo.

> [!abstract] Qué hace
> Port del BlackJack de consola original en Java (clases `App`/`Carta`/`Dealer`/`Jugador`/`Leer`/`Pantalla`, migradas desde `js/temp/BlackJack-main`, 2026-08-11). Misma máquina de mazo/reparto/pedir-plantarse-doblar-dividir y el mismo pago (blackjack ×4, ganar ×2, empate devuelve la apuesta, dealer roba hasta 17) — pero los menúes de `Scanner` pasaron a botones. Las cartas se mantienen tal cual el arte ASCII original (cajas `┌─────────┐` en un `<pre>` monoespaciado, no divs con CSS) y `System.out` pasó a actualizar ese `<pre>` en vez de escribir a consola. Sin persistencia en BD: la plata del jugador vive solo en memoria mientras la ventana está abierta. **Sin pedir nombre** (2026-08-11): el jugador es siempre `"Kne"`, hardcodeado — no hay ningún `<input>` de texto en toda la app.

## Diferencias respecto al Java original

- **Leer.java → botones**: los menúes numéricos (`1)Pedir carta 2)Plantarse...`) pasaron a `_esperarBotones(opciones)`, que renderiza un botón por opción legal y resuelve una Promise al click. Como un botón inválido no puede existir, se eliminó el `default: "opción inválida"` del switch original — no tiene equivalente posible con este medio.
- **Sin `pedirNombre()`**: a diferencia de la primera versión (que sí tenía un `<input>` para el nombre), ahora el jugador arranca siempre como `Jugador("Kne", DINERO_INICIAL)` — pedido explícito de sacar toda entrada de texto libre.
- **Apuesta**: en vez de un monto libre tecleado (`Leer.leerDouble()`), se arma con fichas (`_pedirApuesta`: botones `+$10/+$50/+$100/+$500`, `Todo`, `Limpiar`, `Apostar`). Al clickear "Apostar" se vacía `_elControles` ahí mismo (2026-08-11) — si las fichas quedaban montadas durante el reparto, sumado al tamaño más grande de mesa/cartas, la ventana terminaba con scroll.
- **Pantalla.java → `Pantalla.esperarTecla()`**: reemplazado por un botón "Continuar" explícito en los puntos donde el Java original solo esperaba un keypress (post-BlackJack, post-resultado de mano).
- **Paleta**: los palos ♥/♦ del Java usaban ANSI rojo (`[31m`); acá se mantienen en el mismo verde monocromo del resto de KneOS (ver [[feedback_css_monochrome_green_only]] — regla del proyecto, nunca colores fuera de paleta).
- **Menú principal estilo Kfruit, con carga real del juego** (2026-08-11): igual que `KFruit._crearContenido()`/`_mostrarMenu()`/`_initGame()` (ver [[Kfruit]]), `this.body` es un único contenedor `"mainMenu"` que las pantallas reconstruyen entero (`innerHTML = ""` + `append`), no una ventana con secciones fijas. `_crearContenido()` solo crea el jugador/mazo y llama a `_mostrarMenu()` (logo ASCII en `<pre>` — fuente de bloque 5×5 propia, `_getLogoLines()` — + lista de `<p>` clickeables en `.opciones`, reutilizando esas clases de `Kfruit.css` tal cual). Clickear "JUGAR" saca la clase `mainMenu` y llama a `_initGame()`, que recién ahí arma header/mesa/mensaje/controles (antes vivían armados de entrada en `_crearContenido`) y arranca `_loopPartida()` — el mismo bucle do-while de `juego()` del `App.java` original (elegir "Jugar mano" mientras haya plata, "Volver al menú" corta el loop). Al cortar, vuelve a `_mostrarMenu()`, que repone la clase `mainMenu` y el logo. Sin dinero, el loop muestra "Te quedaste sin dinero" + "Volver al menú"; ya en el menú, esa misma condición cambia "JUGAR" por "REINICIAR" (con el mismo texto "Te quedaste sin dinero" arriba). Con plata, el menú **no** muestra el monto — solo el logo y "JUGAR" (2026-08-11); la plata recién se ve al cargar la mesa (`_initGame` → header).

## Modelo (`js/model/`)

`Carta` (numero/valor/tipo/direccion), `Dealer` (nombre + `cartasEnMano`), `Jugador extends Dealer` (+ `plata`/`apuesta`/`dividir`) — mismos nombres de campo y métodos que las clases Java equivalentes, ver [[Frontend Model Services Utils#Model]].

## Motor del juego (métodos privados de `BlackJack`)

- `_crearMazo`/`_mezclar`/`_restablecerMazo`: **48 cartas** (4 palos × A,2-9,J,Q,K — el loop original va de `i=1` a `12` y nunca genera un numero "10" explícito, así que el mazo no tiene el rango numérico 10; bug preexistente del Java, portado tal cual). Fisher-Yates; `_restablecerMazo` se dispara si quedan <10 cartas, conserva las ya repartidas (misma lógica — con la misma comparación rara del Java original, `numero === tipo`, portada tal cual).
- `_repartirCartas`: reparte en el orden clásico jugador→mesa→jugador→mesa, con una pequeña pausa entre cada carta. La primera carta del dealer se muestra boca arriba; la segunda (la "hole card") se reparte **ya tapada** desde el primer render, no se muestra un instante y se tapa después (2026-08-11, corrección de UX sobre la primera versión — que por fidelidad al `secuenciaRepartir()` del Java mostraba las dos cartas del dealer destapadas durante el reparto).
- `_asciiCartaAtras`/render con `ocultarDealer`: recién al entrar al turno del jugador se tapa la mano del dealer — y se tapa la **segunda** carta repartida (`cartasEnMano[1]`, la "hole card"), la primera (`[0]`) queda visible con su valor mostrado como suma parcial. Calca `imprimirCartaAtras()` del Java, que también mostraba `[0]` y dibujaba un cajón `░░░░░░░░░` fijo para la segunda.
- `_turnoJugador(t)`: `t=0` mano principal, `t=1` segunda mano tras dividir. Detecta blackjack natural (21 con 2 cartas), si no entra al loop pedir/plantarse/doblar/dividir — doblar y dividir solo aparecen como botón si la condición legal se cumple (plata suficiente, 2 cartas, par para dividir).
- `_jugarDividir`: separa el par en dos manos, dobla la apuesta, juega ambas (`_turnoJugador(0)` y `(1)`) y liquida las dos por separado.
- `_turnoDealer`: revela la carta oculta, roba hasta sumar 17+.
- `_mostrarGanador(cartas)`: mismo árbol de pagos que `ganador()` del Java (blackjack ×4 / ambos se pasan devuelve apuesta / gana ×2 / empate devuelve apuesta).

## Persistencia

Ninguna — a diferencia de Kfruit (`KfruitServices`), la plata y el mazo no se guardan en BD; cerrar la ventana y reabrirla reinicia la partida.
