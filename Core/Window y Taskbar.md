---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# Window y Taskbar

⬅️ Volver a [[Frontend Core]]

## `Window` (`core/Window.js`)

Ventana arrastrable/redimensionable/maximizable con integración a la barra de tareas. Crea internamente un único `_taskBar = new TaskBarManager()` a nivel de módulo (compartido por todas las instancias).

**Constructor(id, titulo, icono, crearContenido, opciones={})** — `crearContenido` es invocado perezosamente recién al abrirse por primera vez. `opciones` (2026-07-29, agregado originalmente para el ex `PropertiesApp`, primer consumidor de `Window` fuera de [[File]]; hoy lo usa `ViewWindow`, ver más abajo):
- `opciones.clase`: clase CSS extra agregada a `.ventana` al crearla (antes de `_centrar`, para que el centrado mida el tamaño ya corregido) — así una ventana puede tener su propia variante visual sin tocar la regla base de `.ventana`.
- `opciones.tamano` (`{width, height}`): tamaño fijo en px aplicado como **estilo inline** (antes de `_centrar`) — gana sobre el 1200×1000 fijo de `.ventana` en `window.css` sin necesidad de una clase CSS por cada tamaño posible. La usa `ViewWindow`.

- `opciones.onTogglePin` (2026-07-30): callback reenviado tal cual a `TaskBarManager.agregar()`/`fijar()` en cada llamada (ver `_onTogglePin`, guardado en el constructor) — es lo que le permite al **propio menú contextual de la taskbar** anclar/desanclar sin que `TaskBarManager` necesite conocer a `File`. `File.js` es hoy el único punto que construye un `Window` (`new Window(..., { onTogglePin: () => this.togglePin() })`), así que en la práctica toda ventana con `_usaTaskbar` trae este callback.

Todas las opciones son aditivas — instancias que no las pasan se comportan exactamente igual que antes (`ViewWindow`, que no usa taskbar, simplemente nunca lo necesita).

Además, dos **getters overrideables por subclases** (no opciones de instancia — son parte del comportamiento de la clase, ver `ViewWindow` abajo):
- `get _usaTaskbar()` — `true` por defecto; si una subclase lo pisa a `false`, `abrir()`/`cerrar()`/`minimizar()`/`traerAlFrente()` se saltan por completo sus llamadas a `_taskBar` (la ventana nunca aparece en la barra de tareas).
- `get _permiteResize()` — `true` por defecto; si una subclase lo pisa a `false`, `_hacerMovible` no engancha `interact(...).resizable(...)` ni la detección de zonas de snap (ver Snap a bordes, abajo) — la ventana solo se puede arrastrar, nunca redimensionar ni acomodar contra un borde.

**Métodos públicos:**
- **`abrir()`**: si ya existe la ventana, la muestra y trae al frente. Si no, construye el DOM (barra de título + botones + contenido + overlay de bloqueo durante drag/resize), aplica `opciones.clase`/`opciones.tamano` si vinieron, registra `mousedown → traerAlFrente()`, habilita drag/resize vía **interact.js** (`_hacerMovible`), y si `_usaTaskbar` se registra en la taskbar (`_taskBar.agregar(...)`). Guarda la referencia al botón de maximizar en `this._maximizeButton` para que el snap pueda sincronizar el icono sin pasar por el closure del click (solo si `_crearBotonesControl()` lo generó — ver abajo). En ambos caminos (reutilizar o crear) dispara `_playOpenAnimation()`.
- **`cerrar()`**: no remueve el DOM al instante — usa `_fadeOut(v => v.remove())`. Resetea flags (`_snapZone`, `_minimizada`) y, si `_usaTaskbar`, saca el botón de la taskbar de forma inmediata (antes de que termine el fade), para que la UI responda al toque aunque la ventana siga desvaneciéndose; recién después de disparar el fade pone `this._ventanaEl = null` (el nodo capturado en el closure de `_fadeOut` sigue vivo hasta el `remove()`).
  > [!success] `interact(...).unset()` al cerrar — evita interactables fantasma (2026-08-03)
  > `_hacerMovible` registra `interact(ventana).resizable(...)` e `interact(barra).draggable(...)`, pero interact.js guarda cada uno en un registro propio (por elemento) que **no se limpia solo porque el nodo salga del DOM** — encontrado investigando el lag reportado en Chrome (ver [[Deuda Técnica]]). Abrir y cerrar ventanas repetidas veces sin este fix iba acumulando interactables "fantasma" que interact.js seguía evaluando en cada evento de puntero, aunque la ventana ya no existiera. `cerrar()` ahora llama `interact(this._ventanaEl).unset()` e `interact(this._barraEl).unset()` (nuevo campo `_barraEl`, guardado en `abrir()` junto con `_ventanaEl`) antes de sacar el nodo.
  >
  > **Experimento relacionado, revertido**: se probó además desactivar drag/resize (`interact(...).draggable(false)`) de toda ventana que no estuviera al frente — solo la activa quedaba interactiva, reactivándose en `traerAlFrente()`. Medido con trazas de Performance: sin ganancia de GPU (dentro del ruido), y con un bug de UX real (un solo click+arrastre sobre una ventana tapada no arrancaba el drag, por orden de fases de evento entre el listener reactivador y el propio `mousedown` de interact.js sobre `barra`). Revertido por completo — ver [[Deuda Técnica]] para el detalle.
  > [!bug] `--activo` quedaba pegado en un botón anclado tras cerrar (2026-07-30, arreglado)
  > `cerrar()` nunca llamó `_taskBar.desmarcarActivo()` — antes no hacía falta, porque `eliminar()` siempre remataba destruyendo el nodo del botón entero cuando el grupo quedaba vacío, así que la clase `--activo` desaparecía junto con el elemento. Al agregar el anclado (ver `TaskBarManager` abajo), `eliminar()` empezó a **preservar** el botón de una entrada `pinned` en vez de borrarlo — y ese botón se quedaba con `--activo` para siempre, porque nada volvía a sacarle la clase. Fix: `cerrar()` ahora llama `_taskBar.desmarcarActivo(this.id)` igual que ya hacía `minimizar()`.
- **`minimizar()`**: marca `_minimizada = true` y, si `_usaTaskbar`, desmarca en la taskbar de inmediato, pero el DOM recién se oculta (`display: none`) cuando termina el fade — vía `_fadeOut(v => v.style.display = "none")`.
- **`restaurar()`**: `display: flex` inmediato + `_playOpenAnimation()` (misma animación de aparición que usa `abrir()`).
- **`toggleMinimizar()`**: alterna según si es la ventana activa.
- **`estaActiva()`**: expone `_esVentanaActiva()` (usado por [[Folder]] para saber si su ventana responde a Backspace).
- **`estaAbierta()`** / **`estaMinimizada()`**: `!!_ventanaEl && !_minimizada` / `!!_ventanaEl && _minimizada`. Las usa `File.toggleVentana()` (ver [[File]]) para decidir si el click sobre un ícono debe cerrar, restaurar o abrir.
- **`toggleMaximizar(btn)`**: agrega `.ventana--snapping` (mismo mecanismo que el snap por arrastre, ver abajo) antes de mutar tamaño/posición, así el click también anima. Si `_snapZone === "full"` restaura el tamaño previo; si no (sin snap, o con snap parcial a mitad/cuarto), guarda estado y aplica el rect `"full"`. Alterna la clase `restaurar` en `btn` vía `_syncMaximizeButton` (ver iconos abajo). `_maximizada` sigue existiendo como getter derivado (`_snapZone === "full"`) por compatibilidad. Nunca se llama en una ventana sin botón de maximizar (`ViewWindow`).
- **`traerAlFrente()`**: `ContextMenu.closeAll()` (ver [[Menús Contextuales]]) + `Clock.closeAll()` (2026-07-29, ver [[Clock]] — mismo motivo: el click que abre/enfoca una ventana hace `stopPropagation()` y nunca llega al listener de "click afuera" del calendario) + máximo z-index + 1; si `_usaTaskbar`, marca activo en la taskbar. `Window.js` importa `Clock.js` para esto.
- **`fijarEnTaskbar(onAbrir)` / `desfijarDeTaskbar()`** (2026-07-30): delegan 1:1 en `_taskBar.fijar(id, titulo, icono, onAbrir)` / `_taskBar.desfijar(id)` (ver `TaskBarManager.fijar/desfijar` abajo), sin efecto si `!_usaTaskbar`. A diferencia de `abrir()`/`cerrar()`, no tocan el DOM de la ventana en sí — solo el botón de la taskbar, que puede existir sin que la ventana esté abierta. Llamados desde [[File#Métodos públicos|File.fijarEnTaskbar/desfijarDeTaskbar]], nunca directo desde fuera de `File`.
- **`refrescarContenido()`** (2026-07-31): reconstruye solo el contenido (`_ventanaEl.querySelector(":scope > .app")` reemplazado por un `_crearContenido()` nuevo), sin tocar la barra de título ni cerrar/reabrir la ventana. No hace nada si `!_ventanaEl` (ventana cerrada — el próximo `abrir()` ya arma contenido fresco solo) ni si el contenido no tiene una raíz con clase `.app` (convención que siguen `Folder`/`RecycleBin`/`FileProperties`/`TaskbarSearch`, no todas las apps).
  > [!bug] `FileProperties` no actualizaba los datos al reabrir (2026-07-31, resuelto)
  > `abrir()` tiene un early return si `_ventanaEl` ya existe (arriba: solo la muestra y la trae al frente, no reconstruye nada) — pensado para el caso normal de restaurar una ventana ya abierta tal cual estaba. El problema: [[Menús Contextuales#`FileProperties`|FileProperties]] cachea una única instancia por archivo (`static _instancias`) y, si se pedía "Propiedades" de nuevo para el mismo archivo **sin haber cerrado la ventana anterior**, `_abrir()` solo actualizaba `titulo`/`icono` y después llamaba `abrir()` — que, al encontrar `_ventanaEl` ya viva, ni se enteraba de que había datos nuevos que mostrar (nombre/tamaño/fechas quedaban congelados en lo que fuera que tenía la primera apertura). Fix: `FileProperties._abrir()` ahora llama `this.window.refrescarContenido()` antes de `this.window.abrir()` — si la ventana estaba cerrada no hace nada (no hay `_ventanaEl` todavía) y `abrir()` arma el contenido fresco como siempre; si ya estaba abierta, reconstruye el contenido en el momento con los datos actuales.

**Privado:**
- `_crearBotonControl(clase, etiqueta, onClick)` — factory de un botón individual de la barra de título; pone `aria-label` (no hay tooltips nativos en KneOS) y el listener con `stopPropagation()`.
- `_crearBotonesControl()` (2026-07-29, **overrideable** — ver `ViewWindow`): arma los botones de la barra de título en orden y guarda el de maximizar en `this._maximizeButton`. Implementación base: `[btnMinimizar, btnMaximizar, btnCerrar]`. `ViewWindow` la pisa para devolver solo `[btnCerrar]`.
- `_applyRect(rect)` / `_savePreviousState()` / `_restorePreviousState()` — mecánica común de tamaño/posición reutilizada tanto por `toggleMaximizar` como por el snap de bordes.
- `_playOpenAnimation()` — saca `.ventana--opening` (por si quedó de una corrida anterior), fuerza reflow, la agrega, y la vuelve a sacar en `animationend`. La usan `abrir()` y `restaurar()`.
- `_fadeOut(onComplete)` — agrega `.ventana--fading`, y en `transitionend` la saca y recién ahí ejecuta `onComplete(v)`; no hace nada si ya está desvaneciéndose (`classList.contains`). La usan `cerrar()` (→ `v.remove()`) y `minimizar()` (→ `v.style.display = "none"`). Sacar la clase es necesario: si quedara puesta (caso `minimizar`, el nodo sigue vivo con `display:none`), la próxima vez que `_playOpenAnimation()` corriera la `@keyframes` de apertura, `.ventana--fading` seguiría ahí de fondo — y apenas la animación termina y se saca `.ventana--opening`, esa clase volvería a pisar `opacity`/`scale`, disparando la transición de `.ventana` de vuelta a oculto justo después de haber terminado de aparecer (bug real: abrir → minimizar → reabrir hacía que la ventana pareciera abrirse y cerrarse/minimizarse sola).
- `_hacerMovible` — configura `interact(ventana).resizable(...)` (bordes, mínimo 300×200, bloquea overlay durante resize — **solo si `_permiteResize`**) e `interact(barra).draggable(...)` siempre, con la lógica de snap descrita abajo (la detección de zona de snap dentro del `move` también se salta si `!_permiteResize`, así `activeZone` nunca se setea y el `end` nunca aplica un rect de snap).

### Animación de apertura/cierre/minimizado

Dos mecanismos distintos, no uno solo, porque tienen requisitos distintos:

- **Apertura/restaurar** (`.ventana--opening`, `@keyframes ventanaAparecer`, `_playOpenAnimation`): es una **animación CSS**, no una transición. Se probó primero con transición (`.ventana--fading` + forzar reflow, después doble `requestAnimationFrame`) y fallaba específicamente al crear una ventana nueva desde `abrir()` (confirmado con Playwright: agregar y sacar la clase pasaba en ~10ms, sin que el navegador llegara a pintar ni un frame del estado "oculto" en el medio — con doble rAF incluido, ya que en ese momento la ventana nunca tuvo un frame previo del que partir). Una `@keyframes` no tiene ese problema: define su propio recorrido completo (`from`/`to`) apenas se aplica la clase, sin depender de ningún estado pintado anteriormente, así que funciona igual de bien en una ventana recién creada que en una que ya existía (restaurar desde minimizado).
- **Cierre/minimizar** (`.ventana--fading`, `_fadeOut`): acá sí una transición funciona bien, porque el elemento siempre existe y ya fue pintado en su estado normal antes de que se dispare el fade — hay un "antes" real del que partir. `.ventana` tiene `transition: opacity, scale` **permanente** en la regla base (no en la clase que se agrega y se saca: si la transición viviera solo ahí, al remover la clase el navegador ya no vería ninguna `transition` declarada en el nuevo estado y el cambio sería instantáneo). `_fadeOut` agrega `.ventana--fading` y espera `transitionend` para recién ahí ejecutar el callback (remover del DOM, o poner `display:none`).

Ambos mecanismos animan `opacity` y la propiedad CSS `scale` independiente (no `transform: scale(...)`), para no pisar el `transform: translate(x,y)` inline que ya usa el drag/snap.

### Snap a bordes (`WindowSnap.js`)

Al arrastrar la barra de título, `_hacerMovible` consulta en cada `move` a `WindowSnap.detectSnapZone(clientX, clientY, contenedor)`, que traduce la posición del puntero dentro de `#aplicaciones` a una zona: `"full"` (franja superior, centro), `"left"`/`"right"` (franjas laterales, mitad), o una de las 4 esquinas (`"topLeft"`, `"topRight"`, `"bottomLeft"`, `"bottomRight"`, cuartos) — las esquinas tienen prioridad sobre la franja superior. Mientras hay una zona activa se muestra un rectángulo fantasma (`showSnapPreview`, clase CSS `.snapPreview`) calculado por `getSnapRect(zone, area)` a partir de `_areaDisponible()`.

Al soltar (`end`), si había una zona activa: guarda el estado previo (si no estaba ya acomodada), agrega la clase `.ventana--snapping` (transición corta de 0.12s en `width/height/transform`) y aplica el rect vía `_applyRect`. `this._snapZone` queda seteado a esa zona.

Solo la zona `"full"` cuenta como "maximizada" para el botón: acomodar a mitad o cuarto deja el ícono en "Maximizar" (expandir), porque la ventana no ocupa el 100% — el botón sigue ofreciendo llevarla a pantalla completa. Únicamente al llegar a `"full"` (por drag contra el borde superior o por el propio botón) el ícono cambia a "Restaurar".

Si se vuelve a arrastrar una ventana ya acomodada, los primeros ~10px de movimiento la despegan: se restaura su tamaño previo y se recalcula la posición para que el punto de agarre bajo el cursor no salte (proporción horizontal + offset vertical fijo dentro de la barra de título).

`WindowSnap.js` no guarda estado por ventana — solo geometría (`detectSnapZone`, `getSnapRect`) y el overlay de preview (`showSnapPreview`/`hideSnapPreview`, un único `<div class="snapPreview">` singleton por módulo, anexado a `#aplicaciones`).

**Iconos de los controles** (`styles/core/window.css`): los botones no llevan texto — un `::before` con `mask-image: var(--icono)` + `background-color: currentColor` pinta el SVG correspondiente (`sources/core/`, ver [[Frontend Core]] — reorganización de `sources/` por categorías, 2026-07-31), heredando el color del botón (incluido el hover, que invierte fondo/color). Mapeo por clase CSS: `btnMinimizar` → `remove.svg`, `btnMaximizar` → `expand.svg` (o `dismiss.svg` cuando tiene la clase `.restaurar`, es decir la ventana ya está maximizada), `btnCerrar` → `cancel.svg`. El botón de cerrar de la lista agrupada de la taskbar (`.taskbarGrupoListaCerrar`, en `context-menu.css`) usa el mismo patrón con `cancel.svg`.

## `TaskBarManager` (`core/Taskbarmanager.js`)

Gestiona los botones de la taskbar. **Agrupa ventanas por extensión** (no una por ventana): si hay más de una del mismo tipo, comparten un botón con badge numérico; el click abre un listado (vía [[Menús Contextuales|ContextMenu]]) con cada título y un botón de cerrar individual.

**Constructor(containerId="taskbarIcons")** — `_grupos: Map<extension, {icono, ventanas: Map<ventanaId, entrada>}>`.

> [!success] Anclar a la taskbar (2026-07-30)
> Cada `entrada` ahora es `{titulo, onToggle, onCerrar, pinned, abierta, reabrir}` en vez de solo `{titulo, onToggle, onCerrar}`. `abierta` distingue si hay una `Window` real detrás (registrada vía `agregar()`) de una entrada puramente anclada (`fijar()`, sin ventana viva); `pinned` es el flag persistido (columna `pin`, ver [[Módulo Icon]]); `reabrir` guarda el callback para reabrir una ventana ya cerrada — se setea únicamente en `fijar()` y **nunca se pisa** en `agregar()`/`eliminar()`, así una entrada anclada siempre sabe cómo reabrirse aunque su `Window` real haya sido destruida por `cerrar()`.
>
> El disparador end-to-end vive en [[Menús Contextuales#`ContextMenuManager` (`core/ContextMenuManager.js`)|ContextMenuManager]] ("Anclar a la barra de tareas" / "Quitar de la barra de tareas", junto al toggle de favoritos) → [[File#Métodos públicos|File.fijarEnTaskbar()/desfijarDeTaskbar()]] → `Window.fijarEnTaskbar()/desfijarDeTaskbar()` (arriba) → acá.

**Métodos públicos:**
- `agregar(ventanaId, titulo, icono, onToggle, onCerrar, onTogglePin)`: agrega la ventana a su grupo (extensión parseada de `titulo`, todo lo que sigue al último `.`) y renderiza. Nunca se llama para una ventana con `_usaTaskbar === false` (ver `Window.abrir()` arriba) — así que ni siquiera hace falta que esta clase sepa de `ViewWindow`. Si ya existía una entrada **no abierta** para ese `ventanaId` (anclada pero cerrada), preserva su `pinned`/`reabrir` y solo actualiza `onToggle`/`onCerrar`/`abierta=true` — si ya estaba `abierta`, no hace nada (guard contra doble-agregado). `onTogglePin` (2026-07-30, opcional) se guarda en la entrada; si no viene, cae al `onTogglePin` que ya tuviera la entrada previa (para no perderlo en un re-`agregar()`).
- `eliminar(ventanaId)`: si la entrada está `pinned`, **no la borra** — la marca `abierta=false`, `onCerrar=null` y revierte `onToggle` a `reabrir` (así el botón sigue clickeable para volver a abrir la ventana). Si no está `pinned`, comportamiento original: la saca del grupo y, si queda vacío, remueve el botón.
- **`fijar(ventanaId, titulo, icono, onAbrir, onTogglePin)`** (2026-07-30): ancla una entrada — crea el grupo/botón si no existía, o marca `pinned=true` sobre una entrada ya `abierta` (conservando su `onToggle`/`onCerrar` reales) o sobre una inexistente (usa `onAbrir` como `onToggle`, sin `onCerrar` porque no hay nada que cerrar). Siempre guarda `reabrir=onAbrir`.
- **`desfijar(ventanaId)`** (2026-07-30): si la entrada sigue `abierta` (ventana viva), solo pone `pinned=false` y re-renderiza — sigue en la taskbar mientras la ventana esté abierta, y al cerrarse caerá por el camino normal de `eliminar()`. Si no está `abierta`, la borra directo (mismo cleanup de grupo vacío que `eliminar()`).
- `marcarActivo(ventanaId)` / `desmarcarActivo(ventanaId)`.

> [!success] Menú contextual propio de la taskbar (2026-07-30)
> Antes, anclar un archivo solo era posible desde el menú contextual de su ícono en el escritorio ([[Menús Contextuales]]). Ahora la taskbar tiene su propio click-derecho para anclar/desanclar, sin tener que ir a buscar el ícono:
> - **Botón de un solo archivo** (`grupo.ventanas.size === 1`, ver `_render`): `btn.oncontextmenu` abre directo el menú de anclado para esa única entrada.
> - **Ítem dentro del listado agrupado** (`_onClick`, cuando el botón representa varios archivos de la misma extensión): cada `item` del listado (`ContextMenu.addItem`) tiene su propio `contextmenu` además del `click` que ya alternaba la ventana — así se puede anclar un archivo puntual del grupo sin afectar a los demás.
> - **`_abrirMenuPin(e, entrada)`** *(privado)*: hace de menú compartido por ambos casos — un único ítem "Anclar"/"Quitar de la barra de tareas" (según `entrada.pinned`) que llama `entrada.onTogglePin()`. No hace nada si la entrada no trae `onTogglePin` (hoy siempre lo trae, ver nota de `Window` arriba — el guard es solo defensivo). Usa un menú propio (`PIN_MENU_ID = "taskbarPinMenu"`, `bindOutsideClose` en el constructor junto a `LISTA_ID`) — abrirlo cierra automáticamente el listado agrupado si estaba abierto (`ContextMenu.open()` llama `closeAll()` internamente, ver [[Menús Contextuales]]).
> - El toggle real (mutar `pinned` + persistir en BD) vive en **`File.togglePin()`**, no acá — ver [[File#Métodos públicos|File]]. `TaskBarManager` solo dispara el callback; no conoce `id_icon` ni `iconServices`.

**Privados:** `_render` (además del click, en cada render fija `btn.oncontextmenu` — solo si el grupo tiene 1 entrada, ver arriba), `_onClick` (si el grupo tiene 1 ventana, toggle directo; si tiene más, abre/cierra listado — cada ítem del listado usa `selected: pinned` para el punto indicador de `ContextMenu.addItem`, y solo agrega el botón de cerrar si `abierta && onCerrar`), `_abrirMenuPin` (ver arriba), `_buscarExtension`, `_extension(titulo)`, `_btnId(extension)`.

Instanciado (singleton por módulo) dentro de `Window.js`.

> [!success] Lupa de búsqueda en la taskbar (2026-07-30)
> `_bindSearchButton()` (llamado desde el constructor) agrega un botón ícono-only (`sources/core/search.svg`, `.taskbarBuscadorBoton`) **fuera** de `#taskbarIcons` (`insertBefore` sobre el `parentElement`, `#barraDeTarea`) — esa sección es solo para botones de ventana agrupados por extensión, la lupa no es una ventana. Al click, `import()` **dinámico** de `TaskbarSearch.js` (no estático arriba del archivo): un `import` estático crearía un ciclo de carga — `Taskbarmanager.js → TaskbarSearch.js → ViewWindow.js → Window.js`, y `Window.js` ya importa `Taskbarmanager.js` en su propia primera línea, así que `Window` todavía no habría terminado de definirse (TDZ) cuando `ViewWindow extends Window` intentara ejecutarse. El `import()` dinámico difiere la carga hasta el primer click, momento en el que el grafo de módulos ya terminó de evaluarse por completo. La instancia se crea una sola vez y se reusa en clicks siguientes (`??=`).
>
> **Singleton en `window.taskbarSearch` (2026-07-31, antes un campo privado `this._search`):** al agregar `Home` (ver más abajo), que necesita abrir esta misma ventana de búsqueda para "entregarle" lo tipeado en su propio buscador, el singleton pasó de vivir en un campo privado de `TaskBarManager` a `window.taskbarSearch` — mismo criterio que `window.desktopManager`/`window.recycleBin`. Dos instancias de `TaskbarSearch` competirían por el mismo id de ventana (`ventana_taskbarBuscador`) apenas se abriera la segunda, así que **tiene** que ser una sola, alcanzable desde cualquier lugar que la necesite.
>
> **Iteración de diseño** (documentado porque pasó por 3 formas distintas en la misma sesión): primero un input de texto siempre visible dentro de la propia taskbar, filtrando en vivo sobre `DesktopFolder._filtrarArchivos` (reabriendo su ventana si hacía falta). Descartado: el usuario pidió que la taskbar solo tenga el ícono, y que **no** dependa de `DesktopFolder` — ver `TaskbarSearch` abajo.

## `TaskbarSearch` (`core/TaskbarSearch.js`, 2026-07-30, nuevo)

Ventana "Buscar" abierta por la lupa de la taskbar — diálogo de sistema sobre `ViewWindow` (tamaño fijo 420×520, sin taskbar), mismo patrón que `FileProperties` (ver [[Menús Contextuales]]). **No depende de `DesktopFolder`**: arma su propia lista de resultados y no abre ni toca su ventana/`_clones`.

- **`constructor()`**: `_items = new Map()` (`id_icon → {row, iconRow}`), `_allIcons = []`, crea el `ViewWindow`.
- **`async open(textoInicial = "")`** (parámetro agregado 2026-07-31): `window.abrir()`, pone `textoInicial` en el input (antes siempre `""`) y el cursor al final (`setSelectionRange`), limpia resultados, y recién ahí trae la lista completa **fresca** vía `window.iconServices.getIcons()` (endpoint `GET /iconRoutes/icons`, ver [[Módulo Icon]] — trae **todos** los archivos del sistema, no solo lo ya cargado en `window.archivosAbiertos`) — re-renderiza y re-filtra con `textoInicial` (o lo que el usuario haya tipeado mientras tanto). El parámetro lo usa `Home` para "entregar" lo tipeado en su propio buscador sin perder ni un caracter (ver `Home._irABuscador` abajo) — el camino normal (lupa de la taskbar) sigue llamando `open()` sin argumentos.
- **`_buildContent()`**: input con ícono (`search.svg`, fila `.taskbarBuscadorAppInputRow`) + contenedor de resultados. `Escape` cierra la ventana.
- **`_renderResults()`**: una fila por cada ícono de la BD salvo `ext === "desktop"` (el Escritorio no es un archivo abrible).
- **`_filter(text)`**: a diferencia de `Folder._filtrarArchivos` (texto vacío = todo visible), acá **texto vacío = nada visible** — la ventana arranca vacía en vez de listar el sistema entero de una.
- **`async _openResult(iconRow)`**: resuelve la instancia real vía `resolveArchivo(iconRow, this._allIcons)` (2026-07-31, extraído a `utils/resolveArchivo.js` — antes era un método privado `_resolveFile` acá mismo; se compartió tal cual porque `Home` necesitaba la misma lógica) y la abre. El resultado puede no estar todavía instanciado como `File` real (su carpeta contenedora nunca se abrió en la sesión — carga perezosa, ver `Folder._loadContent` en [[Folder]]): el helper sube recursivamente por `parent_id` hasta encontrar un ancestro ya en `window.archivosAbiertos` (los archivos raíz siempre lo están) y va bajando de nuevo llamando `_loadContent()` en cada nivel hasta que el archivo buscado quede registrado. Si no se puede resolver, no hace nada (la ventana se queda como está — sin mensaje de error).

> [!info] Limitación conocida: íconos anclados dentro de una carpeta nunca abierta
> `DesktopManager._finalizarIcono` es el único punto que llama `archivo.fijarEnTaskbar()` al cargar un ícono ya anclado desde BD (ver [[DesktopManager]]) — pero los hijos de una carpeta recién se instancian como `File` (vía `crearIconoHijo`) cuando esa carpeta se abre al menos una vez en la sesión (carga perezosa, ver [[Folder]]). Un archivo anclado dentro de una carpeta que el usuario nunca abrió en esa sesión no aparece en la taskbar hasta que la carpeta se abra. No arreglado — requeriría instanciar el árbol completo en el arranque, cambio de arquitectura mayor fuera de alcance de este feature.

## `Home` (`core/Home.js`, 2026-07-31, nuevo — ex `TaskbarHome.js`/clase `TaskbarHome`, renombrado el mismo día)

Ventana "Inicio" abierta por un segundo botón de la taskbar (ícono `menu.svg`, ver `TaskBarManager._bindHomeButton` abajo) — mismo patrón que `TaskbarSearch`: diálogo de sistema sobre `ViewWindow` (720×580, sin taskbar), arma su propia lista vía `IconServices.getIcons()`. La clase/archivo/CSS (`styles/core/home.css`) se llamaban `TaskbarHome`/`taskbarHome.css` al principio; se renombraron a secas porque, a diferencia de `TaskbarSearch`, esta ventana no es solo "lo que abre el botón de la taskbar" — es la pantalla de Inicio del sistema en sí. El botón que la abre en la taskbar (`HOME_BUTTON_ID = "taskbarHomeBoton"`, en `Taskbarmanager.js`) sí conserva el prefijo `taskbar` a propósito: ese nombre describe el botón (vive en la taskbar), no la ventana que abre.

- **`constructor()`**: `_allIcons = []`, `ContextMenu` propio (para el menú de "prender", ver footer abajo), crea el `ViewWindow` (ícono `sources/accions/menu.svg`).
- **`async open()`**: `window.abrir()`, limpia el input de búsqueda, trae `getIcons()` fresco y renderiza las columnas.
- **`_buildContent()`**: `div.app.homeApp` (flex column) con tres partes — buscador arriba, `.homeMain` (flex, `overflow-y:auto`) en el medio, footer siempre visible (`flex-shrink:0`) abajo.

### Buscador que no filtra — entrega a `TaskbarSearch`

`_crearBuscador()` arma un input igual al de `TaskbarSearch` (mismo ícono `search.svg`), pero **no filtra nada localmente**: cada `input` dispara `_irABuscador()`, que:
1. `import()` **dinámico** de `TaskbarSearch.js` — mismo motivo que `TaskBarManager._bindSearchButton` (evita el ciclo `Window.js → TaskBarManager.js → Home.js/TaskbarSearch.js → ViewWindow.js → Window.js`).
2. `window.taskbarSearch ??= new TaskbarSearch()` — **reusa el mismo singleton** que ya usa la lupa de la taskbar (ver la nota de `window.taskbarSearch` más arriba); no crea una instancia propia, porque dos `TaskbarSearch` competirían por el mismo id de ventana.
3. `window.taskbarSearch.open(texto)` — abre con lo ya tipeado, cursor al final (ver el nuevo parámetro de `TaskbarSearch.open` arriba), foco puesto ahí para poder seguir escribiendo sin cortes.
4. `this.window.cerrar()` — Inicio se cierra; es solo la puerta de entrada, no un segundo buscador en paralelo. El orden importa: cerrar *después* de que `TaskbarSearch.open()` ya movió el foco a su propio input, así el cierre de Inicio no le roba el foco.

### Columnas — `_crearColumnaRecientes()` / `_crearColumnaTodo()`

Dos columnas dentro de `.homeMain` (flex row, comparten un solo scroll vertical):

- **"Abierto recientemente"**: `this._allIcons` filtrado a `ext !== "desktop" && last_opened_at` (excluye el Escritorio y lo nunca abierto), ordenado por `last_opened_at` descendente. Sin límite de cantidad — la columna ya es scrolleable. Mensaje `.homeVacio` si no hay ninguno.
- **"Todo"**: todos los archivos (salvo `desktop`) agrupados por `file_type` (`FileType`, ver [[Frontend Model Services Utils#Model|model/fileTypes.js]]) — un `<h4>` de subtítulo (`getFileTypeLabel`) por categoría presente, en el orden fijo `CATEGORIAS_ORDEN` (Juego, Productividad, Utilidades, Sistema, IA, Otros al final). Categorías sin ningún archivo no muestran subtítulo vacío.
- **`_crearItem(iconRow)`**: fila clickeable (ícono real por extensión + nombre) — al click, `_abrirItem` resuelve la instancia con el mismo `resolveArchivo` que usa `TaskbarSearch` y llama `archivo.toggleVentana()`, después cierra Inicio.
- **Menú contextual por ítem — `_abrirMenuItem(e, iconRow, css)`** (2026-07-31): clic derecho ofrece los mismos dos ítems que `ContextMenuManager._setIconMenu` para un ícono del escritorio (ver [[Menús Contextuales]]) — **Copiar ruta de acceso** (`navigator.clipboard.writeText(formatearRuta(iconRow.src))` — igual que `ContextMenuManager`, pasa por `formatearRuta` para copiar espacios en vez de `_`, ver `utils/formato.js` en [[Frontend Model Services Utils]] —, mismo fire-and-forget con `console.warn` si falla) y **Propiedades del archivo** (`FileProperties.abrirPara(iconRow.id_icon, {...iconRow, icono: css})`). El `css` es el mismo ícono ya resuelto por `getIconMeta` para pintar la fila — se lo pasa como snapshot de respaldo (mismo criterio que `RecycleBin._abrirPropiedades`, ver [[RecycleBin]]): si el archivo todavía no está instanciado en `window.archivosAbiertos` (su carpeta nunca se abrió esta sesión), `FileProperties` cae a ese snapshot y muestra una vista de **solo lectura** en vez de fallar. `FileProperties.js` se importa estático arriba del archivo — a diferencia de `TaskbarSearch.js` (que sí necesita import dinámico, ver `_irABuscador`), acá no hay ciclo posible: nada en la cadena de imports de `FileProperties.js` vuelve a `Home.js`, y `Home.js` en sí no es parte del grafo `Window.js → TaskBarManager.js` que obliga a diferir esos otros imports.

> [!success] `.homeContenedor` → `.homeGrupo`: dos rondas de agrupamiento (2026-07-31)
> **Ronda 1**: versión inicial, los `.homeItem` de cada grupo se agregaban como hermanos sueltos del título (`<h3>`/`<h4>`) dentro de `.homeColumna` — título e ítems mezclados al mismo nivel. Se agregó `_crearContenedorItems(iconRows)` — un `div.homeContenedor` (`display:flex; flex-wrap:wrap`) con los `.homeItem` de ese grupo adentro — para que los archivos de cada grupo quedaran debajo de su título, no sueltos.
>
> **Ronda 2** (mismo día): pidieron agrupar también el **título** junto con su `.homeContenedor` en un bloque propio, no solo los ítems entre sí. Se agregó `_crearGrupo(titulo, cuerpo)` — un `div.homeGrupo` (`flex column`) que envuelve el `<h3>`/`<h4>` y su cuerpo (el `.homeContenedor`, o el mensaje `.homeVacio` si el grupo está vacío) — usado por `_crearColumnaRecientes()` (título "Abierto recientemente" + su único contenedor) y por cada categoría de `_crearColumnaTodo()` (subtítulo + contenedor de esa categoría). El `<h3>` "Todo" de la columna queda **afuera** de cualquier `.homeGrupo`: no tiene un único contenedor propio, agrupa varias categorías que ya se agrupan por su cuenta.
>
> El espaciado entre grupos pasó de un `margin-top` por subtítulo (con un `:first-of-type` para no duplicarlo en el primero) al `gap` de `.homeColumna` — con cada título ahora "primero" dentro de su propio `.homeGrupo`, `:first-of-type` hubiera aplicado a todos por igual en vez de solo al primero de la columna, así que ese hack ya no tenía sentido y se sacó.

> [!info] Interpretación de "tres grupos" (2026-07-31)
> El pedido original decía "alineados en izq a der tres grupos Abierto Recientemente, y Todo" pero solo nombraba dos. Se implementaron esos dos ("Abierto recientemente" y "Todo", con "Todo" internamente subdividido por categoría) — si la intención era un tercer grupo con nombre propio (por ejemplo "Favoritos", que ya existe como concepto en el sidebar de [[Folder]]), falta agregarlo.

### Footer — usuario + botón de prender

`_crearFooter()`: `.homeFooter` (`justify-content: space-between`, `border-top`, siempre visible por vivir fuera del área con scroll):
- **Izquierda** — `.homeUsuario`: ícono `sources/appIcon/kneAi.png` (mismo que usa [[KneAI]] — el "usuario" del sistema es la propia Kne) + texto "Kne".
- **Derecha** — `.homePowerBtn` (ícono nuevo `sources/core/power.svg`, círculo+línea estilo stroke — a diferencia del resto de los SVG del proyecto, que son `fill="currentColor"` con paths sólidos; funciona igual porque `aplicarIconoImagen` lo usa como `mask-image`, que no distingue fill de stroke). Al click abre un `ContextMenu` propio (`POWER_MENU_ID`) con **Reiniciar** y **Apagar** — ambos con `onClick: () => {}`, **sin acciones todavía**, a pedido explícito ("por ahora sin acciones").

### `TaskBarManager._bindHomeButton()` — botón en la taskbar

Mismo patrón exacto que `_bindSearchButton` (import dinámico, singleton en `window.home`, `insertBefore` sobre `#taskbarIcons`), reusando las clases CSS `.taskbarBuscadorBoton`/`.taskbarBuscadorBotonImg` (mismo tamaño/estilo que el botón de la lupa, solo cambia el ícono a `menu.svg`) en vez de duplicar CSS. Se llama **antes** que `_bindSearchButton()` en el constructor de `TaskBarManager` — como ambos hacen `insertBefore(btn, this._container)` sobre el mismo `#taskbarIcons`, llamar a Home primero lo deja más a la izquierda: orden final Inicio → Buscador → íconos de ventanas agrupados.

## `ViewWindow` (`core/ViewWindow.js`, 2026-07-29, extiende `Window`)

Ventana "solo de vista": tamaño fijo, un único botón de control (Cerrar — sin minimizar ni maximizar), y no se registra en la taskbar. Pensada para diálogos de inspección puntual que no necesitan comportarse como una app persistente del sistema (usada hoy por [[Menús Contextuales|FileProperties]], ex `PropertiesApp`, y por [[Camera]] desde 2026-08-19 — diálogo de "sacar una foto").

**Constructor(id, titulo, icono, crearContenido, {width, height, clase, onClose}={})** — reenvía a `super(...)` con `{ clase, tamano: {width, height}, onClose }`. El `onClose` se sumó recién con [[Camera]] (2026-08-19): hasta entonces `ViewWindow` no reenviaba nada de `opciones` más allá de `clase`/`tamano`, así que un `onClose` pasado por el caller se perdía en silencio — sin apagar los tracks de `MediaStream` de Camera al cerrar la ventana, el navegador seguiría marcando la webcam como "en uso". Cambio retrocompatible: ningún otro `ViewWindow` existente (`Calculator`, `FileProperties`, `TaskbarSearch`, `Home`) pasaba `onClose`, así que ninguno cambia de comportamiento.

Overrides (los tres ganchos que `Window` expone justamente para esto):
- `get _usaTaskbar()` → `false`.
- `get _permiteResize()` → `false`.
- `_crearBotonesControl()` → `[btnCerrar]` únicamente (reusa `this._crearBotonControl`, heredado de `Window`).

No sobreescribe nada más — `abrir()`/`cerrar()`/`minimizar()`/`restaurar()`/`toggleMaximizar()` son los mismos de `Window`, simplemente no hacen nada dañino porque no hay botón de min/max que los dispare y las llamadas a `_taskBar` quedan saltadas por `_usaTaskbar`.
