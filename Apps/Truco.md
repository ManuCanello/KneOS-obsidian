---
tags:
  - portfolio/kneos
  - apps
---

# Truco

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Truco.js` (clase `Truco`) — extiende [[File]]. Extensión `"truco"`, ícono propio `sources/appIcon/truco.svg`, `src = null`. Sin `Window` propia: usa la `Window` completa por defecto de `File`, igual que [[BlackJack]]/[[Kfruit]]/[[Kmd]].

> [!abstract] Qué hace
> Truco Argentino de 2 jugadores contra la máquina, a 30 puntos (malas 0-15, buenas 16-30). A diferencia de [[BlackJack]]/[[Hangman]]/[[FlipCoin]]/[[Kdle]]/[[CarRace]]/[[Tetris]], **no es un port de un juego de consola en Java** — no había código original que seguir, así que el motor (bazas/pardas/cantos de Truco/Envido/Flor) se escribió desde cero (2026-09-02), a partir de un doc de reglas (`Truco.md`, fuera del repo) que el propio Manuel redactó. La ESTRUCTURA sí calca a BlackJack: mismo patrón de "un botón solo existe si la jugada es legal" (acá `_accionesLegales(quien)`, la única función de la que cuelga toda la máquina de estados — tanto la UI como el bot rival consumen la misma lista, ninguno de los dos puede jugar algo ilegal por construcción), mismo `_esperarAccion*` basado en Promise, mismo doble modo escritorio/consola vía `this._modoTerminal` (`run truco`, ver [[Kmd]]). El rival es un **bot local por reglas** (nivel + personalidad, elegibles en el menú), no un LLM — instantáneo, sin login, sin costo de red, determinístico para poder calibrarlo jugando.

## Modelo

Reusa [[Frontend Model Services Utils#Model|Card]] tal cual (`numero`/`valor`/`tipo`/`direccion` ya encajan) — no hay un modelo propio. `Player`/`Dealer` (de BlackJack) **no** se reusan; el estado de los dos jugadores vive en campos planos de la instancia (`_puntos`, `_cartas`, `_manoOriginal`).

**Jerarquía de las 40 cartas** (1-7,10,11,12 × espada/basto/oro/copa, sin 8/9): en vez de comparaciones encadenadas, cada carta recibe un *nivel* 1-14 vía una tabla de consulta (`NIVEL_BRAVAS`/`NIVEL_POR_NUMERO`) — comparar dos cartas es comparar dos números, y mismo nivel = parda automáticamente (1 de copa y 1 de oro caen los dos en nivel 8, los dos 7 falsos en nivel 4, sin código extra para esos empates).

## Motor del juego

- `_accionesLegales(quien)`: central. Devuelve la lista de acciones legales para quien tiene el turno — jugar una carta, cantar/subir Truco, cantar/subir Envido, responder Quiero/No quiero, o irse al mazo. Un canto pendiente (`_cantoPendiente`) restringe las opciones a solo responderlo.
- `_cerrarBaza()` + `_evaluarMano()`: truth-table de la regla de pardas (ganar 2 bazas / ganar la 1ra y la 2da es parda / la 1ra parda define la 2da / las 2 primeras pardas define la 3ra / las 3 pardas gana el mano), evaluada baza a baza — nunca hace falta una 4ta.
- `_resolverEnvido(querido)`: implementa **una interpretación propia** para el "no quiero" en la cadena Envido→Real Envido→Falta Envido, porque el doc de reglas no la fija (solo da un ejemplo de lo *querido*: "Envido + Real Envido = 5"). Cada escalón guarda su "valor propio" (`envido`=2, `real_envido`=3, `falta_envido`=variable); si se rechaza, el que cantó último gana la suma de los escalones ANTERIORES (mínimo 1 — así "no quiero" a un Envido suelto vale 1 y no 0, como en una mesa real). Verificado jugando: Envido(2)+Real Envido(3) rechazado en el Real Envido devuelve exactamente 2 (el valor del Envido que quedó "aceptado" antes de la subida).
- `_valorFaltaEnvido()`: la Falta Envido tampoco tiene valor fijo en el doc. Interpretación elegida: si el líder está en malas, vale lo que le falta para 15; si ya está en buenas, lo que le falta para 30. Aislada en una función de una sola cuenta para poder cambiar de variante después.
- `_resolverFlorAutomatica()`: la Flor (opcional, toggle "CON FLOR" en el menú, apagada por defecto) **no tiene mecánica de canto/respuesta** en el doc de reglas — a diferencia del Envido, no hay Quiero/No quiero descrito, solo "se compara y gana el mayor". Por eso se resuelve automáticamente apenas se reparte la mano (antes de arrancar el loop de turnos): una sola Flor se la lleva directo, dos Flores se comparan (empate lo gana el mano), y ese mismo mano no juega Envido (Truco.md #15).
- **Simplificación deliberada**: los cantos se resuelven en el orden en que se cantan — si el Truco queda pendiente, el Envido no puede "interrumpirlo" (la regla de mesa real le da prioridad al Envido incluso sobre un Truco ya cantado, pero el doc de reglas no detalla esa prioridad, así que se optó por lo más simple de implementar en vez de inventarla).
- **Corregido en la primera pasada de testing** (2026-09-02, jugando una partida entera con Playwright): un Envido/Flor que cruza el marcador a 30 a mitad de una mano (antes de terminar de jugar la última baza) no cortaba la partida — el chequeo de fin de partida solo corría ENTRE manos. Se agregó `_partidaTerminada()` y se lo suma también a la condición del loop de turnos de `_jugarMano`, así la partida corta ahí mismo, como en una mesa real.

## Turno de cartas vs. turno de respuesta

Tres campos separados evitan un bug de "a quién le toca" cuando hay una cadena de subidas: `_liderBaza` (quién lideró la baza actual — solo lo tocan los cierres de baza, para la regla de "parda mantiene ventaja"), `_turnoCarta` (a quién le toca jugar la próxima carta — el pointer "real"), y `_turno` (quién debe actuar AHORA, que un canto pendiente puede tomar prestado temporalmente). Al resolverse cualquier canto (Quiero, o No quiero de Envido — el de Truco termina la mano), `_turno` vuelve a `_turnoCarta`, nunca a quien cantó último — si no, en una cadena con más de una subida el turno de jugar cartas terminaba en la persona equivocada.

## Rival (bot local)

`_decidirBot(acciones)` consume la MISMA lista que dibuja los botones del jugador — no puede jugar algo ilegal por construcción, y solo lee `this._cartas.rival`/`this._manoOriginal.rival` y lo ya jugado (nunca las cartas del jugador humano, Truco.md #21). Heurística corta a propósito: fuerza de mano (nivel promedio de las cartas que le quedan, 0-1), sus propios puntos de Envido, y dos configuraciones elegibles en el menú — `nivel` (fácil/normal/difícil, controla los umbrales de aceptar/cantar y cuánto ruido aleatorio hay) y `personalidad` (conservador solo canta con mano real; farolero también canta/acepta con mano floja, con cierta probabilidad). Los umbrales quedan en objetos literales locales dentro de `_decidirBot`, fáciles de retocar sin leer el resto del motor.

## Jugable en dos modos (escritorio + consola)

Igual patrón que BlackJack: `_crearContenido()` bifurca por `this._modoTerminal`. El modo consola agrega `truco` a `RUN_GAMES` en [[Kmd]] (`run truco`); cada pantalla tiene su gemela en texto plano (`_mostrarMenuConsola`, y `_iniciarPartida`/`_esperarAccionConsola` construyen la mesa sin clases de `game.css`). En consola las cartas siguen siendo arte ASCII en un `<pre>` (mismo estilo de caja `┌─────────┐` que BlackJack, con padding calculado por carta porque el número puede ser de 1 o 2 caracteres, y el palo es el carácter ♠♣♦♥ tal cual) — una terminal de texto no puede mostrar un ícono. Logo "TRUCO" con la misma fuente de bloque 5×5 por letra que [[FlipCoin]].

## Palo real en el modo escritorio (2026-09-02)

En la mesa gráfica (no en consola) el carácter ♠♣♦♥ de cada carta se reemplaza por el SVG real del palo español -- `sources/core/{espada,basto,oro,copa}.svg`, agregados por Manuel, mismo estilo pixel-art de un solo `<path fill="currentColor">` que el resto de `sources/appIcon/*.svg` (`basto` es un garrote de madera, no el trébol de la baraja francesa). `PALO_ICONO` mapea cada símbolo (`carta.getTipo()`) a su archivo.

Como son SVG **externos** (no paths embebidos en el JS), se colorean con la técnica que el proyecto ya usa para cualquier ícono SVG así -- `mask-image` + `background-color: currentColor` vía `aplicarIconoImagen()` (`utils/iconoStyle.js`, la misma función que pintan los íconos del escritorio y los botones de ventana) -- en vez de un sistema de `<symbol>`/`<use>` aparte.

`_crearCarta(carta)` arma un `<div class="trucoCarta">` con el mismo layout que tenía el arte ASCII (`_bloqueCarta`): rango arriba-izquierda, palo al medio (vía `_crearPalo`), rango invertido 180° abajo-derecha. `_crearDorso()` es un dorso con rayado diagonal (`repeating-linear-gradient`) en vez de un color atenuado. `_pintarFilaCartas`/`_pintarFilaDorsos` reemplazan el `.textContent` de ASCII por `contenedor.replaceChildren(...)` con las cartas reales -- `_render()` bifurca por `this._modoTerminal` para elegir entre esas y las funciones ASCII de siempre.

### Pips por cantidad, con layout tradicional (2026-09-02)

Pedido puntual de Manuel, en dos vueltas:

1. Que la carta muestre tantos íconos de palo como su número (como una baraja española real), en vez de uno solo -- 10/11/12 (figuras, sin conteo de palo) siguen mostrando un solo ícono centrado, sin cambios en ninguna de las dos vueltas.
2. Reemplazar el primer layout (flex-wrap genérico, sin posiciones fijas) por la posición y rotación EXACTA de cada ícono, palo por palo y número por número (1-7), que Manuel dio como una tabla de datos.

`POSICIONES_PIPS` (arriba de la clase, junto al resto de las tablas de consulta) es esa tabla tal cual la dio Manuel -- `{ oro/copa/espada/basto: { "1".."7": [{ position, rotation }] } }` -- sobre una grilla nombrada de 3x3 (`AREA_POR_POSICION`: `topLeft/topCenter/topRight/middleLeft/center/middleRight/bottomLeft/bottomCenter/bottomRight` → `TL/TC/TR/ML/C/MR/BL/BC/BR`). `_crearPalo(carta)` busca `POSICIONES_PIPS[nombrePalo][carta.getNumero()]` -- el nombre del palo sale de `carta.getDireccion().split("-")[0]` (ya lo trae armado `_crearMazo`, `"espada-6"` etc., no hizo falta un mapeo aparte símbolo→nombre) -- y por cada entrada arma un `<span class="trucoPaloIcono trucoPip">` con `style.gridArea` (la posición) y `style.transform = rotate(...)` (la rotación, inline porque la MISMA posición tiene distinta rotación según palo/número -- no se puede resolver con una clase CSS por posición). `.trucoPips` en CSS es el grid 3x3 fijo (`grid-template-areas` con esas 9 áreas + `place-items: center`), ya no flex-wrap.

Los ángulos no son arbitrarios: espada y basto llevan sus pips en diagonal (35°/-35°, en direcciones opuestas entre los dos palos) salvo cuando se cruzan al medio (90°) en los números impares con `center` ocupado -- oro y copa van siempre derechos (rotation 0). Verificado instanciando `Truco` en el navegador y renderizando las 40 cartas del mazo juntas (mismo truco que la vuelta anterior) para comparar las 4×7 combinaciones contra la tabla de una sola mirada -- coinciden pixel a pixel con lo pedido.

De paso (primera vuelta) se agrandó `.trucoCarta` (`clamp(56px→88px)` a `clamp(72px→120px)`) y el font-size del rango, a pedido explícito de "hacer más grande las cartas".

## TRUCO 2.0 -- "Truco estratégico moderno" (2026-09-02)

Modo alternativo pedido por Manuel, agregado DENTRO de la misma app (mismo menú, toggle `MODO: CLÁSICO/TRUCO 2.0`) a partir de un manual de reglas completo que Manuel escribió (`Truco2.md`, fuera del repo, dos variantes -- **mazo propio** y **mazo compartido 20/20**). Se implementó solo la variante de **mazo propio** (elegida por Manuel ante la pregunta directa de cuál de las dos era "Truco 2.0" -- la de mazo compartido 20/20 quedó sin implementar). **Solo existe en modo escritorio**: la pantalla de "elegí 3 cartas entre las que te quedan" no tiene un análogo razonable de consola de texto, así que `run truco` sigue siendo 100% el motor clásico, sin el toggle de modo siquiera.

### Por qué reusa casi todo el motor clásico

La diferencia real entre los dos modos es **de dónde sale la mano de 3 cartas de cada ronda** y **qué pasa alrededor de la ronda** (mazo que se gasta, roles, objetivo secreto, carta de poder, desgaste, presión) -- la mecánica de jugar la ronda en sí (bazas/pardas/Truco/Retruco/Vale cuatro/Envido) es la MISMA. Por eso Truco 2.0 no reimplementa nada de eso: `_accionesLegales`/`_aplicarAccion`/`_evaluarMano`/`_resolverEnvido`/`_calcularEnvido`/`_decidirBot`/`_elegirCarta`/`_render`/`_crearCarta` son los mismos métodos que usa el modo clásico, sin ninguna rama nueva adentro (siguen operando sobre `this._cartas`/`this._jugadas` genéricos). Solo dos métodos clásicos llevan un hook mínimo, gateado por `this._version2` (así que en modo clásico su comportamiento es idéntico, byte a byte, al de antes de este cambio):

- `_cerrarBaza()` ahora compara bazas vía `_compararBaza()`/`_valorParaRonda()` en vez de la comparación inline de siempre -- así la carta de poder "Cambiar jerarquía" puede invertir el orden (`_valorParaRonda` devuelve `-carta.getValor()`) sin tocar `_evaluarMano` ni el resto. La carta de poder "Fuerza" (gana la más baja en la 1ra baza) se resuelve invirtiendo el `resultado` ya calculado, solo si `_indiceBaza === 0` y no fue parda.
- `_resolverEnvido(querido)` llama a `_bonificarEnvidoPoder(ganador)` después de sumar los puntos normales -- no-op en modo clásico, suma el bonus de la carta de poder "Envido cargado" en 2.0.

Truco 2.0 tampoco juega Flor (Truco2.md no la menciona en ningún lado, a diferencia del Envido que sí aparece indirectamente vía la carta de poder) -- `_jugarRonda2()` nunca llama a `_resolverFlorAutomatica()`, y el toggle "CON FLOR" del menú se esconde cuando `MODO` está en 2.0 para no sugerir que hace algo.

### Motor propio de Truco 2.0

- `_jugarPartida2()`: arma un mazo de 40 cartas POR JUGADOR (`_mazoPersonal.jugador`/`.rival`, cada uno una copia independiente de `_crearMazo()` -- nunca se comparten), elige rol (jugador vía pantalla, rival al azar) y objetivo secreto (al azar para ambos), y corre el loop de rondas hasta `_partidaTerminada2()`.
- `_partidaTerminada2()`: reusa `_partidaTerminada()` (el chequeo de puntos de siempre) y le suma la otra condición de Truco 2.0 -- que a alguno de los dos mazos personales le queden menos de 3 cartas (interpretación de "ambos jugadores se quedan sin cartas": los dos mazos se gastan a un ritmo casi idéntico porque cada baza consume una carta de cada lado a la vez, así que alcanza con chequear cualquiera de los dos). El límite de rondas vago que menciona el punto 14 del manual ("se alcanza el límite de rondas establecido") no se implementó -- no fija un número, y el agotamiento del mazo ya cubre ese caso en la práctica.
- `_jugarRonda2()`: análogo a `_jugarMano()` pero la mano sale de `_seleccionarManoV2()` (selección, no reparto al azar) en vez de `_repartir()`, y agrega la carta de poder de la ronda (una al azar de `CARTAS_PODER`, revelada ANTES de elegir mano) y el banner de ronda de presión (`_numeroRonda % 4 === 0`, puntos x2). El loop de turnos interno es el mismo `while` que el modo clásico, con el mismo chequeo de corte a mitad de ronda (`_partidaTerminada()`, no la versión 2 -- ver comentario en el código, es a propósito: la versión 2 mide el mazo restante DESPUÉS de sacar la mano de la ronda, así que evaluarla a mitad de ronda cortaría la partida por error).
- `_seleccionarManoV2()`: reemplaza `_repartir()`. El rival elige con `_elegirManoBot2()` (difícil se queda con sus 3 cartas más fuertes disponibles; fácil/normal, 3 al azar -- mismo criterio deliberadamente simple que `_decidirBot`). El jugador elige en `_pantallaSeleccionCartas2()`, una pantalla propia que pisa `this.body` (reusa `_crearCarta()` para dibujar cada carta del mazo restante como clickeable) hasta juntar 3, con `_armarMesaDesktop()` reconstruyendo la mesa de juego al final.
- `_cerrarRonda2()`: puntaje con hasta tres multiplicadores/bonus (presión x2, carta de poder "Doble apuesta" x2, rol Apostador +1 si ganó con Truco cantado), archiva las cartas JUGADAS en `_usadas` (`{carta, motivo}` -- distingue `"jugada"` de `"desgaste"` porque el rol Conservador solo puede recuperar cartas jugadas, no descartadas por desgaste) y devuelve a `_mazoPersonal` las que no se llegaron a jugar. También lleva la racha de victorias consecutivas (`_racha`) para el objetivo "2 rondas seguidas" y llama a `_evaluarObjetivo2()`.
- `_evaluarObjetivo2(ganador)`: los 4 objetivos secretos del manual, cada uno una sola vez por partida (`obj.cumplido`), +3 puntos la primera vez que se cumple. "Vale cuatro" se resuelve como `_nivelTruco === NIVELES_TRUCO.length` (llegó a cantarse, sea querido o no -- el manual no aclara esa distinción).
- `_desgasteMazo2()`: cada 5 rondas (`RONDA_PRESION_CADA`/`DESGASTE_CADA` son constantes, fáciles de retocar), ambos pierden 1 carta sin jugarla -- el rival descarta su carta más débil, el jugador elige en `_pantallaDesgaste2()`.

### Roles y habilidades de una vez por partida

`ROLES` (5, tal como los da el manual) se eligen en `_elegirRolJugador()` (pantalla propia) para el jugador y al azar para el rival. Calculador (ve 1 carta del mazo rival antes de elegir mano) y Apostador (+1 punto si gana con Truco) son pasivos, sin UI extra. Observador es un botón "Ver historial" en la pantalla de selección que muestra `_historialRondas` (lo que jugó cada uno, ronda por ronda). Conservador (recuperar 1 carta ya jugada) y Estratega (cambiar 1 carta de la mano ya elegida) son las únicas habilidades que gastan un flag de "usado" (`_habilidadUsada`) -- Estratega solo lo gasta si de verdad cambia algo, no por abrir la pantalla y cancelar.

### Verificado

Tres pasadas con Playwright, cero errores de consola en las tres:

1. Partida completa de punta a punta (rol → 12 rondas → fin) hasta que se agotó el mazo -- se vieron pasar las 5 cartas de poder, rondas de presión en 4/8/12, desgaste en 5/10 (sin trabarse), un objetivo secreto cumplido (`carta_baja`, +3) y el mensaje de cierre "Rival ganó la partida 10 a 20 (se agotaron los mazos)".
2. El Conservador: jugada la 1ra ronda, en la 2da apareció la pantalla "recuperar 1 carta ya jugada" con las 3 cartas jugadas en la ronda 1; al recuperar una, el mazo restante pasó de 37 a 38 (40 − 3 jugadas + 1 recuperada) y la pantalla de selección de la ronda 2 la mostró disponible.
3. El Estratega: mano elegida `[1, 2, 3]`, la pantalla de cambio ofreció esas 3, se cambió la primera por un reemplazo elegido a mano, y la carta que terminó en la mesa fue la de reemplazo (`[4, 2, 3]`) -- confirma que el swap pisa la carta correcta antes de que arranque la ronda.

## Testing

Verificado con Playwright automatizando una partida entera de punta a punta (menú → `run truco` → decenas de manos → fin de partida a 30, y de nuevo en modo escritorio abriendo el ícono desde la carpeta Juegos) — cero errores de consola. El cálculo de Envido, la cadena de Truco (incluida la regla de "no se puede subir el propio canto"), la resolución de pardas y el corte a mitad de mano se verificaron leyendo los mensajes reales que produjo el motor durante esas partidas, no solo revisando el código. Para el cambio de palo (2026-09-02) se instanció `Truco` directo en el navegador vía `page.evaluate` para renderizar las 40 cartas del mazo de una sola vez (sin depender de un reparto al azar) y revisarlas todas juntas, además de la partida real en pantalla y en consola (esta última sin cambios, confirmado a propósito).

## Persistencia

Ninguna — igual que [[BlackJack]]/[[Kfruit]] (parcial), el marcador vive solo en memoria mientras la ventana está abierta.
