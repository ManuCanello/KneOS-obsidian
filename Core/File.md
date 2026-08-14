---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# File

⬅️ Volver a [[Frontend Core]]

`core/File.js` — clase raíz de todo "archivo" del sistema (no extiende nada). [[Folder]] extiende `File`, y por lo tanto [[DesktopGrid y DesktopFolder|DesktopFolder]] también (indirectamente). Todas las [[Apps|apps de escritorio]] extienden `File`.

## Constructor

`File(nombre, extension, src, icono, tipo, size=0)` — el 5º parámetro se llamaba `direction` y se recibía pero nunca se usaba (dead param); el 2026-07-31 se repurpuseó ese mismo lugar como `tipo` en vez de agregar un parámetro nuevo al final. El 6º (`size`, 2026-07-29) es el tamaño inicial en bytes — ver nota abajo.

- `nombre`, `extension`, `icono` guardados tal cual.
- `desktopPlace = null`, `parentId = null`, `id = null`.
- `src = "Escritorio/" + nombreParaRuta"` (sobreescrito luego si viene de BD/carpeta padre; el segmento raíz era `"KneOS"` hasta 2026-07-29, cambiado a `"Escritorio"` para que la ruta mostrada/copiada — ver "Copiar ruta de acceso" y "Propiedades del archivo" en [[Menús Contextuales]] — coincida con el nombre real de la raíz en el breadcrumb/sidebar de [[Folder]]).
- Metadata inicial: `size=size` (párametro del constructor, ver abajo), `createdAt`, `updatedAt`, `lastOpenedAt=null`, `favorite=false`, `pinned=false` (2026-07-30, sobreescritos con datos reales en `DesktopManager._aplicarMeta`), `fileType=tipo ?? FileType.OTHER` (2026-07-31, ver nota abajo).

> [!info] Propiedad `fileType` (2026-07-31)
> Categoría del archivo — enum `FileType` en `model/fileTypes.js` (ver [[Frontend Model Services Utils#Model]]), persistida en `files.file_type` (ver [[Módulo Icon]]). Seis valores: `GAME` (1, Juego), `PRODUCTIVITY` (2, Productividad), `UTILITY` (3, Utilidades), `OTHER` (4, Otros), `SYSTEM` (5, Sistema), `AI` (6, IA) — los ids tienen que coincidir con la tabla `files_type` sembrada en ese mismo orden.
>
> **Cada subclase la pasa explícita en su propio `super(...)`** (ver [[Apps]]: `Doom`/`Kfruit`→`GAME`, `TxtFile`→`PRODUCTIVITY`, `Kmd`→`UTILITY`, `Folder`/`RecycleBin`→`SYSTEM`, `KneAI`→`AI`, `Maxwell`→`OTHER`) — **no** se deriva sola de `extension` dentro de `File`. Una primera versión de este cambio sí la derivaba automáticamente (un mapa extensión→categoría interno a `File`/`fileTypes.js`), pero se descartó a favor de la asignación explícita: así cada app declara su categoría a la vista, en el mismo lugar donde ya declara su extensión/ícono, en vez de tener que ir a revisar un mapa aparte para saber a qué categoría pertenece. Si no se pasa nada (p. ej. una instancia de `File` "pelada", el fallback `default` de `iconSrc.js`), cae en `FileType.OTHER`.
>
> `DesktopManager._aplicarMeta` la pisa con lo persistido en BD si vino (mismo patrón que `fav`/`pin`) — en la práctica siempre coincide con lo que ya pasó el `super()` de la subclase, porque la categoría de cada app no cambia por instancia. `IconServices.newIcon` manda `file.fileType` en el POST (ver [[Módulo Icon]] — `saveIcon`/`newIcon` reciben `file_type` desde el mismo cambio).
>
> **`DesktopFolder` es un caso particular sin bug real:** su cadena de `super()` pasa por `Folder` primero (que pasa `FileType.SYSTEM` fijo en su propio `super()`), y recién *después* de que `File` ya guardó ese `fileType`, `DesktopFolder` pisa `this.extension = "desktop"` en su propio constructor (ver [[DesktopGrid y DesktopFolder]]) — pero nunca toca `fileType`, así que se queda con el `SYSTEM` que le pasó `Folder`. Como conceptualmente el Escritorio también es "Sistema", el resultado es el correcto de cualquier forma; no hay nada que ajustar mientras `DesktopFolder` no necesite una categoría distinta a la de `Folder`.

> [!info] Parámetro `size` (2026-07-29)
> Agregado para que apps sin contenido persistido en BD (Doom, KneAI, KFruit — ver [[Apps]]) puedan declarar un tamaño de archivo plausible en su propio `super(...)`, en vez de mostrar siempre "0 B".
>
> **Limitación cerrada (2026-07-29, mismo día):** al principio `IconServices.newIcon` no mandaba `size` al crear el ícono en BD, así que este valor se perdía apenas `DesktopManager._aplicarMeta` recargaba el ícono desde la BD (columna `size`, siempre 0 porque nunca se persistía). Ahora `newIcon` sí manda `file.size` en el POST (ver [[Módulo Icon]], `saveIcon`/`newIcon` actualizados en el mismo cambio) — el tamaño declarado en el constructor sobrevive al reload para íconos creados desde este cambio en adelante. Sigue sin haber un endpoint `changeSize`/`editSize` para actualizar el tamaño de un ícono ya existente.
- `window = new Window(_generarVentanaId(), nombreCompleto, icono, () => this._crearContenido())`.

## Getters/setters

- `nombreCompleto`: `"${nombre}.${extension}"`.
- `nombreParaRuta`: igual con espacios → `_` (override en `Folder`, que usa solo `nombre`).
- `ventanaId`: delega en `window.id`.

## Métodos públicos

- `abrirVentana()`: `window.abrir()`; si `id != null`, dispara (fire-and-forget vía `advertirSiFalla`) `iconServices.changeLastOpened(id)`.
- `cerrarVentana()`, `minimizar()`, `restaurar()`: delegan 1:1 en `window`.
- `toggleVentana()`: el gesto de click sobre el ícono (usado por `DesktopManager._crearContenedorIcono` y `DesktopFolder._renderIconoEscritorio`, **no** por `abrirVentana()` en sí). Si la ventana ya existe (`estaAbierta()` o `estaMinimizada()`) delega en `window.toggleMinimizar()` — minimiza si estaba activa al frente, o restaura/enfoca si estaba minimizada o tapada por otra ventana; si no existe, la abre. La navegación por breadcrumb/sidebar dentro de [[Folder]] sigue llamando `abrirVentana()` directo (sin toggle): ahí minimizar en vez de enfocar sería sorpresivo. El menú contextual "Abrir" (`ContextMenuManager`) también usa `abrirVentana()` directo por la misma razón.
- `actualizarSrc(srcPadre)`: recalcula `src`, actualiza `updatedAt`, persiste vía `changeSrc` (override en `Folder`, que además propaga el cambio a sus hijos).
- **`fijarEnTaskbar()` / `desfijarDeTaskbar()`** (2026-07-30): setean `this.pinned` y delegan en `window.fijarEnTaskbar(() => this.toggleVentana())` / `window.desfijarDeTaskbar()` (ver [[Window y Taskbar#`TaskBarManager` (`core/Taskbarmanager.js`)|TaskBarManager.fijar/desfijar]]) — el callback de reapertura es siempre `toggleVentana()`, igual que el click normal sobre el ícono. **No persisten en BD por sí solos** — eso es `togglePin()`, abajo.
- **`togglePin()`** (2026-07-30): único punto de verdad para alternar el anclado **con** persistencia — `this.pinned ? desfijarDeTaskbar() : fijarEnTaskbar()` seguido de `iconServices.changePin(id, pinned)` (fire-and-forget, `advertirSiFalla`). Dos consumidores: el menú contextual del ícono ([[Menús Contextuales#`ContextMenuManager` (`core/ContextMenuManager.js`)|ContextMenuManager]]) y el menú contextual de la propia taskbar (`TaskBarManager._abrirMenuPin`, ver [[Window y Taskbar]]) — este último le llega vía `opciones.onTogglePin` pasado al construir `this.window` (`new Window(..., { onTogglePin: () => this.togglePin() })`), reenviado por `Window` a `TaskBarManager.agregar()`/`fijar()` en cada llamada. En la carga inicial no hace falta: `DesktopManager._finalizarIcono` llama `fijarEnTaskbar()` directo (sin persistir) para reflejar en la taskbar un `pinned` que ya vino de BD.
- **`animateAppearance()`** (2026-07-30): anima la aparición del ícono (fade + scale-in, clase `.icono--apareciendo`/`@keyframes iconoAparecer` en `icon.css` — mismo patrón CSS que `.ventana--opening`, ver [[Window y Taskbar]]) sobre `this.iconoElement`. La llaman `DesktopManager.crearIcono` (solo si `idExistente == null`, es decir un archivo recién creado por el usuario, no uno recargado de BD) y `crearIconoEnCarpeta` (siempre nuevo), `Folder._moverArchivo` al aterrizar en la carpeta destino, y `DesktopManager._onIconoSoltado` al aterrizar de vuelta en el escritorio viniendo de una carpeta. Saca `icono--eliminando` por si el mismo ícono venía de `animateRemoval()` — sin eso, al terminar el pop-in la clase de borrado seguiría pisando `opacity`/`scale` y el ícono desaparecería de nuevo justo después de aparecer.
- **`animateRemoval()`** (2026-07-30): anima la desaparición del ícono en su lugar actual (mismo `.icono--eliminando` que usa `ContextMenuManager._animarBorradoIcono` para un borrado real, ver [[Menús Contextuales]]) **sin sacarlo del DOM** — se usa cuando el archivo se va a mover de lugar, no cuando se elimina. Devuelve una promesa que resuelve en `transitionend`, para que quien llama pueda esperar antes de reubicarlo. Si `iconoElement` no está conectado al documento (carpeta que nunca se abrió, o está cerrada) resuelve directo sin animar nada — sin este guard (`el.isConnected`), un `transitionend` que nunca dispara sobre un elemento fuera del render tree dejaría la promesa colgada para siempre y el movimiento nunca terminaría de aplicarse. Consumido por `Folder._moverArchivo` (mueve `elemento` a `this.contenedor` recién después de esperar) y `DesktopManager._onIconoSoltado` (mismo patrón, hacia el escritorio) — ver [[DesktopGrid y DesktopFolder]].
- **`renombrar(nuevoNombre)`** (2026-07-29): único punto de verdad para renombrar un archivo, extraído de lo que antes era lógica inline duplicable en dos lugares (`DesktopManager._editarTextoIcono`, ver [[DesktopManager]], y `FileProperties`, ver [[Menús Contextuales]]). Orden importante:
  1. Calcula `nombreLimpio` (trim) y `distinto` (no vacío y ≠ `this.nombre`).
  2. Si `distinto`, asigna `this.nombre` **antes** de refrescar el label — así el `<p>` del ícono siempre queda consistente, incluso en el camino de "no cambió nada" (p. ej. el usuario retipeó el mismo nombre con la extensión pegada: el label vuelve al valor correcto en vez de quedarse con el texto crudo editado).
  3. `_actualizarLabelIcono()` corre **incondicionalmente**.
  4. Si no `distinto`, corta acá (sin tocar `src`/`ventanaId`/persistencia).
  5. Actualiza `window.titulo` a `nombreCompleto` (antes de este método, la barra de título/taskbar nunca reflejaba un rename — bug preexistente, ahora corregido de paso).
  6. Regenera `ventanaId` (con sufijo `Date.now()`) **solo si** `!window.estaAbierta() && !window.estaMinimizada()` — cambiar el id de una ventana viva deja el botón de la taskbar huérfano para siempre (`TaskBarManager.eliminar` busca el id nuevo, no encuentra el grupo registrado bajo el viejo). Guard agregado en el refactor; el código anterior regeneraba siempre.
  7. `actualizarSrc(srcPadre)` (bump `updatedAt` + persiste `src`; cascada a hijos si `this` es `Folder` — ver nota de bug abajo) y `_refrescarColumnas()`.
  8. Persiste el nombre vía `changeName`, envuelto en `advertirSiFalla`.
  > [!bug] Renombrar una `Folder` nunca abierta no propaga `src` a sus hijos en BD
  > `Folder.actualizarSrc` (override, ver [[Folder]]) cascada iterando `this.contenedor.children`, que está vacío hasta el primer `_loadContent()`. Preexistente a este refactor, pero `FileProperties` lo hace mucho más fácil de gatillar (ya no hace falta abrir la carpeta antes de renombrarla). Documentado, no arreglado — fuera de alcance del pedido original.
- `_actualizarLabelIcono()` (2026-07-29, privado): reescribe el `<p>` del ícono real y, si es raíz, de su clon en `desktopFolder._clones` — mismo patrón que `_refrescarColumnas()` pero para el nombre en vez de fecha/tamaño.

## Métodos usados por subclases y otras clases del core

- `_refrescarColumnas()`: refresca columnas fecha/tamaño en `iconoElement` y en su clon en `desktopFolder._clones` si es un ítem raíz.
- `_propagarTamano(delta)`: recorre la cadena de carpetas ancestro (vía `parentId`) sumando `delta` a `size` de cada una — modela "el tamaño de una carpeta es la suma de su contenido" (espejo de `getTamanosAgregados` en [[Módulo Icon]]).
- `_crearContenido()`: por defecto un `div` vacío; **cada app la sobreescribe** para construir su UI real (ver [[Apps]]).
- `_generarVentanaId()`: `"ventana_" + nombre.replaceAll(" ","_")`.

## Teclado compartido para el modo consola de las apps de juego (2026-08-14)

Agregado para que las seis apps `FileType.GAME` ([[BlackJack]], [[FlipCoin]], [[Hangman]], [[Kdle]], [[CarRace]], [[Tetris]]) fueran jugables 100% por teclado **cuando corren dentro de la terminal** (`run <juego>`, `this._modoTerminal`, ver [[Kmd]]), y para que `Kmd._cmdRun` pudiera cortar cualquiera de ellas con Ctrl+C sin saber nada de la pantalla en la que estaba parada.

**Historia (dos correcciones el mismo día):** la primera pasada le sacó el mouse también a las seis apps abiertas normalmente desde el escritorio (efecto colateral no buscado) — se corrigió sumando click a los mismos helpers de teclado, mouse + teclado conviviendo en el modo desktop. A pedido explícito del usuario, esa convivencia se revirtió horas después: el modo desktop volvió a ser **mouse-only, como era originalmente** (sin `_menuTeclado`/`_escapeVuelve`/`_activarSeleccionRepetible`/`_activarGrillaTeclas` — cada pantalla desktop vuelve a un `<p>`/`<button>` con `addEventListener("click", ...)` directo, igual que antes de que existiera el modo consola). Los cuatro helpers de abajo hoy solo los llama el modo consola (`_mostrarXConsola()` de cada app) — quedan en `File`/`BlackJack.js`/`Tetris.js` por más que ya no tengan caller en modo desktop, documentados acá por si se reintroduce navegación por teclado en el desktop más adelante.

- **`_registrarTeclado(handler)` / `_desregistrarTeclado(handler)`**: wrapper de `window.addEventListener("keydown", ...)` que además guarda cada `handler` en `this._listenersTeclado` (un array propio de la instancia). Todo listener de teclado de una app de juego pasa por acá en vez de `window.addEventListener` directo — incluido el gameplay ya keyboard-only de antes (letras de Hangman/Kdle, movimiento de Tetris, que sigue siendo teclado en AMBOS modos, eso nunca cambió), no solo los menús del modo consola.
- **`_limpiarTeclado()`**: saca TODOS los listeners registrados por la instancia de una sola vez, sin importar cuáles están activos ni en qué pantalla — es lo que le permite a `Kmd._cmdRun` salir de cualquier juego, en cualquier estado, con un solo `instancia._detener?.()` genérico. Cada `_mostrarXConsola()` la llama al principio (antes de reconstruir su DOM); las pantallas del modo desktop ya no la necesitan (no registran nada), pero algunas la siguen llamando sin costo real (no-op sobre un array vacío) porque la función es compartida con el modo consola.
- **`_detener()`**: `this._detenido = true` + `_limpiarTeclado()`. Los bucles de juego propios (`while` de Tetris/CarRace) chequean `this._detenido` igual que ya chequeaban su propia bandera de "partida en curso" — sin esto, un `run tetris`/`run carrera` cortado con Ctrl+C a mitad de partida seguiría corriendo su `setTimeout` en loop contra un tablero ya desmontado.
- **`_menuTeclado(items, marcarItem?)`**: menú de "elegir uno y salir" solo para el modo consola — `items: [{el, accion, deshabilitado?}]`. Arriba/Izquierda mueve la selección al anterior, Abajo/Derecha al siguiente (envolvente), Enter dispara `accion()` del resaltado y saca su propio listener. Su único caller hoy es `_consolaMenu` (siempre con `marcarItem`, el cursor de texto `"> "`) — el branch por default (`classList.toggle("seleccionado")`, sin CSS que lo respalde ya que se sacó de `game.css`) quedó sin uso real.
- **`_escapeVuelve(callback)`**: Escape dispara `callback()` — solo para el modo consola, reemplaza el click en "< VOLVER" en las pantallas de una sola salida (configuración, calificaciones, porcentajes). El modo desktop usa su `<p>` "< VOLVER" clickeable de toda la vida, sin pasar por acá.
- **No cubierto por estos dos helpers, bespoke por app, también solo modo consola**: BlackJack tiene un tercer patrón para su selector de fichas (`_activarSeleccionRepetible`, local a `BlackJack.js`, no en `File` — su único caller es `_pedirApuestaConsola`) — Izquierda/Derecha navega, Enter dispara la acción del ítem resaltado pero **sin salir** (fichas/Todo/Limpiar se pueden apretar varias veces seguidas), salvo el ítem marcado `salir: true` (Apostar). El modo desktop de BlackJack (`_pedirApuesta`) tiene su propio código de click directo, sin este helper. Tetris tenía una navegación 2D análoga (`_activarGrillaTeclas`) para su tabla de reasignación de teclas en modo desktop — se **eliminó** en la segunda corrección (dead code una vez que el desktop volvió a ser mouse-only); el modo consola de Tetris (`_mostrarConfiguracionConsola`) usa el `_menuTeclado` genérico de acá, no necesitó su propia grilla 2D porque una lista de texto no tiene columnas.

## Dependencias

[[Window y Taskbar|Window]] (composición), `utils/avisos.js` (`advertirSiFalla`), `utils/formato.js`, y globales `window.iconServices`, `window.archivosAbiertos`, `window.desktopFolder` — ver [[Frontend Model Services Utils]].
