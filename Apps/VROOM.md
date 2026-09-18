---
tags:
  - portfolio/kneos
  - apps
---

# VROOM

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/VROOM.js` (clase `VROOM`, ex `CarRace.js`/`CarRace`, renombrado 2026-09-18 a pedido del usuario — ex `CarreraAuto.js`/`CarreraAuto`, renombrado a inglés 2026-08-13, ver [[Deuda Técnica#Nombres en español traducidos a inglés (2026-08-13)]]) — extiende [[File]]. Extensión `"vroom"` (identificador `ext` persistido en BD, ex `carreraauto`), ícono propio `sources/appIcon/vroom.svg` (ex `carrace.svg`), `src = null`. Sin `Window` propia: usa la `Window` completa por defecto de `File` (como BlackJack/Ahorcado/FlipCoin), sin tamaño fijo.

> [!abstract] Qué hace
> Port del CarreraAuto de consola en Java original (clases `Auto`/`Juego`/`App`, 2026-08-13). 4 autos — **MAX**/**LEWIS**/**FRANCO**/**LANDO** (2026-09-18, ex "Auto 1"–"Auto 4") — avanzan 0-2 posiciones al azar por tick hasta que alguno llega a la meta en 53 — empate en el mismo tick se resuelve a favor del primero en orden de la lista (mismo orden interno rojo→azul→cian→blanco de siempre, solo cambió la etiqueta visible), igual que `nombreGanador()` del original recorre el `ArrayList`. El jugador apuesta a un auto y un monto libre antes de largar; si acierta, cobra `monto × cuota` del auto ganador. El menú del Java tenía 3 opciones (Correr/Ver porcentajes/Salir); acá quedan 2 — "Salir" no se portó, cerrar la ventana cumple esa función (mismo criterio que el resto de las apps de juego).

## Rename CarRace → VROOM (2026-09-18)

> [!info] A pedido explícito del usuario
> Cambio íntegro: archivo (`CarRace.js`→`VROOM.js`), clase, extensión (`carreraauto`→`vroom`), ícono (`carrace.svg`→`vroom.svg`), CSS (`carrace.css`→`vroom.css`, `.carRaceApp`→`.vroomApp`), servicio (`CarRaceServices.js`→`VROOMServices.js`), y todo el stack backend (`carRaceRoutes.js`→`vroomRoutes.js`, `carRaceController.js`→`vroomController.js`, `carRaceModel.js`→`vroomModel.js`, ruta `/carRaceRoutes`→`/vroomRoutes`, `carreraautoLimiter`→`vroomLimiter`) — ver [[Módulo VROOM]]. La tabla de BD también se renombró, `car_race_results`→`vroom_results` (`ALTER TABLE`, preserva los datos existentes, no un drop+create). El logo (banner ASCII fijo, ver más abajo) pasó a deletrear "VROOM" en vez de "CARRERA".
>
> Los nombres de los 4 autos pasaron de "Auto 1"–"Auto 4" a **MAX/LEWIS/FRANCO/LANDO** — el `id` interno (`rojo`/`azul`/`cian`/`blanco`) no cambió, solo el `nombre` mostrado (mismo patrón que el rename de "Rojo/Azul/Cian/Blanco" a "Auto 1"–"Auto 4" del 2026-08-13).

## Diferencias respecto al Java original

- **Persistencia: archivo de texto → tabla en BD**: `grabar()` (que escribía el nombre del ganador a `ganadores.txt`, ruta absoluta hardcodeada) y `calcularPorcentaje()`/`setCuota()` (que releían ese mismo archivo) pasaron a la tabla `vroom_results` (ex `car_race_results`/`carreraauto_resultados`, ver [[Módulo VROOM]]) — igual que `flipcoin_results`, crece con cada carrera corrida, global y sin `pc_id`.
- **Menú numérico → botones**: elegir auto (`apuestas()`, primera mitad) y elegir monto (segunda mitad) pasaron a dos pantallas con botones/input reales — ya no hace falta el reintento en bucle de una entrada inválida, un botón de auto siempre es legal y el de "Apostar" queda deshabilitado hasta que el monto sea > 0.
- **Cuota capturada antes de correr**: la cuota usada para pagar la apuesta se pide al servidor (`GET /vroomRoutes/estadisticas`) justo al confirmar el monto, ANTES de correr la carrera — mismo momento relativo que el original.
- **Cuota con guardas defensivas**: `cuota = total de carreras / victorias de ese auto`. `total === 0` → cuota neutra `x1`; `victorias === 0` con `total > 0` → cuota tapeada en `total` (sustituto finito en vez de `Infinity`).
- **Sin plata persistente por sesión**: la apuesta es un monto libre por carrera sin validar contra ningún balance acumulado (igual que el original).
- **Pista redibujada al final**: el último frame (auto ya cruzando la meta) se renderiza antes del mensaje de ganador, a diferencia del original que cortaba un tick antes.
- **Carriles con etiqueta, sin color ANSI**: cada auto se identifica por una etiqueta de texto (`.caCarrilTitulo`, bold sin color propio desde 2026-09-18 — antes `color: var(--primary-dim)`) sobre su carril en vez de por color de terminal.
- **Pista ocupa el 100% del ancho de la ventana**: la posición se traduce a un porcentaje (`pos/META`) y el auto (`.caAuto`) se posiciona con `left: X%` sobre `.caPista`/`.caPistaLinea`. Pista y auto agrandados 2026-09-18 (`.caPista` de `clamp(10px,1.6cqw,16px)` a `clamp(12px,2cqw,20px)` — `.caAuto` escala con eso porque hereda el font-size del padre).

## Apuestas: máscara de moneda, pago en vivo por auto y "plata ganada" (2026-09-18)

> [!info] Input de apuesta con máscara `$`
> El `<input>` de monto pasó de `type="number"` (con el bug del spinner nativo, ver más abajo) a `type="text"` + `inputMode="numeric"`. Mientras se escribe solo deja dígitos crudos; recién al dejar de escribir (debounce de 600ms tras la última tecla) lo reformatea como `$78.000` (`Intl.NumberFormat("es-AR")`, mismo helper `_formatearPlata` reusado en el header y el mensaje de "Ganaste"). Al volver a enfocar el input se destapa a dígitos crudos para poder seguir editando sin romperse.

> [!info] "Paga: $X" en vivo por carril, durante la carrera
> Antes solo se calculaba la cuota del auto apostado. Ahora `_calcularCuotas(estadisticas)` calcula la cuota de los 4 autos a la vez, y cada carril (`_crearCarril`) muestra "Paga: $X" (monto apostado × cuota de *ese* auto) durante toda la carrera — informativo, no cambia con la posición de los autos (la cuota se fija antes de largar, igual que siempre). Mismo texto agregado al modo consola, junto al nombre de cada auto.

> [!info] "Plata ganada" por auto — persistida, sin importar a quién apostaste
> Columna nueva `vroom_results.premio` (Int, default 0): cada carrera guarda cuánto pagó esa fila **al auto que ganó** (`monto apostado × cuota del ganador`), sin importar si el que corrió esa carrera había apostado a ese auto o a otro — a pedido explícito ("por más que gane el que yo no aposté, se debe igual sumar cuánto dinero ganó"). El "Ganaste $X"/"No ganaste nada" en pantalla sigue dependiendo de si tu apuesta coincidió con el ganador; el `premio` que se manda al backend no. `getEstadisticas()` ahora también devuelve `plataGanada` (`SUM(premio)` agrupado por auto vía `prisma.vroom_results.aggregate`), y la pantalla de porcentajes tiene una 5ª columna "PLATA GANADA".

## Pantalla de porcentajes: de divs a `<table>` (2026-09-18)

`_mostrarPorcentajes()` armaba `.caTabla`/`.caFila` como `<div>`s con `<span>`s adentro. Ahora es un `<table class="caTabla">` real con `<thead>`/`<tbody>` — mismo patrón que la tabla de controles de [[Tetris]]/[[Kfruit]] (`th`/`td` con borde, `border-collapse: collapse`). Columnas: AUTO / VICTORIAS / % / CUOTA / PLATA GANADA. El estado vacío ("Todavía no hay carreras") sigue siendo un `<div>` simple, no una tabla sin filas.

## Contador épico: centrado + fade en vez de temblor (2026-09-18)

> [!info] Iteración en dos pasadas, a pedido del usuario
> Primera pasada: agrandar el "3...2...1.." (`.caCuenta`, `font-size: clamp(64px,14cqw,160px)`, bold) con un pulso continuo (`animation: scale` infinito). Segunda pasada (corrección, mismo día): el usuario pidió centrarlo respecto a **todo lo que queda debajo del header**, no solo el alto chico de `.caMensaje` en su lugar normal del flujo — y reemplazar el pulso por un fade corto que se dispara en cada número, no una animación continua.
>
> Solución: `pistaCont`+`mensaje`+`controles` ahora se envuelven en un `.caArea` (`position: relative`, antes eran 3 hijos directos sueltos de `.vroomApp`). `.caMensaje.caCuenta` pasa a `position: absolute; inset: 0`, centrado con flex — así el número gigante tapa/centra sobre toda esa área en vez de empujar el layout. El pulso (`@keyframes caCuentaPulso`, `transform: scale`) se sacó; en su lugar, `.caCuentaFade` dispara `@keyframes caCuentaFade` (`opacity: 0→1`, 0.35s) — `_secuencia()` saca y vuelve a poner esa clase en cada dígito (`3...`→`2..`→`1..`), con un reflow forzado en el medio (`void mensaje.offsetWidth`) para que el browser note el cambio y reinicie la animación cada vez, en vez de solo la primera. `.caCuenta` (tamaño/peso) se saca al terminar la cuenta, así el mensaje final ("X es el ganador!!!") vuelve al tamaño normal de `.caMensaje`.

## Menú principal

Mismo patrón que BlackJack/Ahorcado/FlipCoin — `this.body` con clases `"app" "game" "vroomApp" "mainMenu"` (2026-09-18: renombrada de `carRaceApp`). A diferencia de FlipCoin, la pantalla de carrera SÍ saca la clase `mainMenu`.

> [!info] Logo: banner ASCII fijo (2026-09-16, reemplaza la fuente de bloque 5×5; texto actualizado 2026-09-18)
> El logo es un banner ASCII grande fijo (pedido explícito del usuario, texto literal pegado), guardado como constante de módulo `LOGO = String.raw\`...\`` y usado directo como `pre.textContent` en `_mostrarMenu()`. Deletreaba "CARRERA" hasta el rename a VROOM (2026-09-18), donde el texto del banner se reemplazó por uno nuevo que deletrea "VROOM" — la constante y el mecanismo (`String.raw`, sin escapar a mano cada `` \` ``) no cambiaron. El modo consola sigue imprimiendo texto plano, ahora "VROOM" en vez de "CARRERA".

## `apps/game.css` (estándar compartido)

No define CSS propio para menú/logo/botones/input — todo eso sale de `.game`/`.mainMenu`/`.game-boton`/`.game-fila-botones`/`.game-input` en `apps/game.css`. `vroom.css` (ex `carrace.css`) solo tiene lo específico de sus pantallas: `.caHeader`, `.caArea` (2026-09-18, wrapper posicionado — ver arriba), `.caPistaCont`/`.caCarril`/`.caCarrilTitulo`/`.caCarrilPago` (2026-09-18)/`.caPista`/`.caPistaLinea`/`.caAuto`, `.caMensaje`/`.caCuenta`/`.caCuentaFade` (2026-09-18), `.caControles` y `.caTabla` (tabla real desde 2026-09-18, ver arriba).

> [!bug] `.game-input` con `type="number"` mostraba el spinner nativo del navegador (2026-09-17)
> El `<input type="number" min="1" placeholder="$">` de `_mostrarMonto()` (monto de la apuesta) era el único `type="number"` de todo KneOS. Fix original en `apps/game.css` (`.game-input`, afecta a **todos** los juegos con un input numérico): `appearance: textfield` + ocultar los botones de spin. Superado 2026-09-18: el input pasó a `type="text"` + `inputMode="numeric"` (ver máscara de moneda arriba), así que el spinner nativo ya ni aplica — el fix de `game.css` queda igual por si algún otro juego usa `type="number"`.

## Modo consola (2026-08-14)

A pedido explícito (jugar desde `run vroom` en [[Kmd]], ex `run carrera` — el alias ahora coincide con el `ext` real, ya no hace falta el caso especial que documentaba la excepción): cada pantalla tiene su par `_mostrarXConsola()` para cuando corre dentro de la terminal (`this._modoTerminal`, ver [[File]]/[[Kmd]]) — texto plano en vez de la UI de `game.css`, navegable con `File._menuTeclado`/`_escapeVuelve`. El bucle de la carrera chequea `this._detenido` para poder cortarse si `Kmd._cmdRun` sale con Ctrl+C a mitad de carrera.

**Mismos autos en consola y desktop:** el arte ASCII del auto (`AUTO_ASCII`) es compartido entre `_crearCarril()` (gráfico) e `_iniciarCarreraConsola()` (texto). Desde 2026-09-18 el modo consola también muestra "Paga: $X" junto al nombre de cada auto y guarda `premio` igual que el modo gráfico — la única asimetría que queda es visual: el modo consola no tiene el contador épico (fade/centrado), sigue con el texto plano "3..."/"2.."/"1.." sin animación, consistente con el resto de las apps portadas en modo terminal.

**Bug histórico: la línea de referencia se quedaba corta contra el auto ganador (2026-08-14):** ver `ANCHO_AUTO` — la línea de guiones pasa a `"-".repeat(META + ANCHO_AUTO)` para que el auto siempre termine dentro de la línea dibujada, aunque esté en `pos = META`.

## Persistencia

Tabla `vroom_results` (ex `car_race_results`/`carreraauto_resultados`, ver [[Módulo VROOM]]) — cada carrera corrida inserta una fila con el auto ganador y el `premio` pagado (fire-and-forget, sin esperar la respuesta antes de mostrar el mensaje final); "VER PORCENTAJES" lee esa tabla vía conteo + suma agregados por auto.

## Consumido por

Servicio frontend `VROOMServices` (ex `CarRaceServices`/`CarreraAutoServices` — ver [[Frontend Model Services Utils#Services]]), usado por [[VROOM]].
