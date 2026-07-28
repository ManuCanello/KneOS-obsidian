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

**Constructor(id, titulo, icono, crearContenido)** — `crearContenido` es invocado perezosamente recién al abrirse por primera vez.

**Métodos públicos:**
- **`abrir()`**: si ya existe la ventana, la muestra y trae al frente. Si no, construye el DOM (barra de título + botones + contenido + overlay de bloqueo durante drag/resize), registra `mousedown → traerAlFrente()`, habilita drag/resize vía **interact.js** (`_hacerMovible`), se registra en la taskbar (`_taskBar.agregar`). Guarda la referencia al botón de maximizar en `this._maximizeButton` para que el snap pueda sincronizar el icono sin pasar por el closure del click. En ambos caminos (reutilizar o crear) dispara `_playOpenAnimation()`.
- **`cerrar()`**: no remueve el DOM al instante — usa `_fadeOut(v => v.remove())`. Resetea flags (`_snapZone`, `_minimizada`) y saca el botón de la taskbar de forma inmediata (antes de que termine el fade), para que la UI responda al toque aunque la ventana siga desvaneciéndose; recién después de disparar el fade pone `this._ventanaEl = null` (el nodo capturado en el closure de `_fadeOut` sigue vivo hasta el `remove()`).
- **`minimizar()`**: marca `_minimizada = true` y desmarca en la taskbar de inmediato, pero el DOM recién se oculta (`display: none`) cuando termina el fade — vía `_fadeOut(v => v.style.display = "none")`.
- **`restaurar()`**: `display: flex` inmediato + `_playOpenAnimation()` (misma animación de aparición que usa `abrir()`).
- **`toggleMinimizar()`**: alterna según si es la ventana activa.
- **`estaActiva()`**: expone `_esVentanaActiva()` (usado por [[Folder]] para saber si su ventana responde a Backspace).
- **`estaAbierta()`** / **`estaMinimizada()`**: `!!_ventanaEl && !_minimizada` / `!!_ventanaEl && _minimizada`. Las usa `File.toggleVentana()` (ver [[File]]) para decidir si el click sobre un ícono debe cerrar, restaurar o abrir.
- **`toggleMaximizar(btn)`**: agrega `.ventana--snapping` (mismo mecanismo que el snap por arrastre, ver abajo) antes de mutar tamaño/posición, así el click también anima. Si `_snapZone === "full"` restaura el tamaño previo; si no (sin snap, o con snap parcial a mitad/cuarto), guarda estado y aplica el rect `"full"`. Alterna la clase `restaurar` en `btn` vía `_syncMaximizeButton` (ver iconos abajo). `_maximizada` sigue existiendo como getter derivado (`_snapZone === "full"`) por compatibilidad.
- **`traerAlFrente()`**: `ContextMenu.closeAll()` (ver [[Menús Contextuales]]) + máximo z-index + 1; marca activo en la taskbar.

**Privado:**
- `_crearBotonControl(clase, etiqueta, onClick)` — factory de los tres botones de la barra de título (`btnMinimizar`/`btnMaximizar`/`btnCerrar`); pone `aria-label` (no hay tooltips nativos en KneOS) y el listener con `stopPropagation()`.
- `_applyRect(rect)` / `_savePreviousState()` / `_restorePreviousState()` — mecánica común de tamaño/posición reutilizada tanto por `toggleMaximizar` como por el snap de bordes.
- `_playOpenAnimation()` — saca `.ventana--opening` (por si quedó de una corrida anterior), fuerza reflow, la agrega, y la vuelve a sacar en `animationend`. La usan `abrir()` y `restaurar()`.
- `_fadeOut(onComplete)` — agrega `.ventana--fading`, y en `transitionend` la saca y recién ahí ejecuta `onComplete(v)`; no hace nada si ya está desvaneciéndose (`classList.contains`). La usan `cerrar()` (→ `v.remove()`) y `minimizar()` (→ `v.style.display = "none"`). Sacar la clase es necesario: si quedara puesta (caso `minimizar`, el nodo sigue vivo con `display:none`), la próxima vez que `_playOpenAnimation()` corriera la `@keyframes` de apertura, `.ventana--fading` seguiría ahí de fondo — y apenas la animación termina y se saca `.ventana--opening`, esa clase volvería a pisar `opacity`/`scale`, disparando la transición de `.ventana` de vuelta a oculto justo después de haber terminado de aparecer (bug real: abrir → minimizar → reabrir hacía que la ventana pareciera abrirse y cerrarse/minimizarse sola).
- `_hacerMovible` — configura `interact(ventana).resizable(...)` (bordes, mínimo 300×200, bloquea overlay durante resize) e `interact(barra).draggable(...)`, con la lógica de snap descrita abajo.

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

**Iconos de los controles** (`styles/core/window.css`): los botones no llevan texto — un `::before` con `mask-image: var(--icono)` + `background-color: currentColor` pinta el SVG correspondiente (`sources/icon/`), heredando el color del botón (incluido el hover, que invierte fondo/color). Mapeo por clase CSS: `btnMinimizar` → `remove.svg`, `btnMaximizar` → `expand.svg` (o `dismiss.svg` cuando tiene la clase `.restaurar`, es decir la ventana ya está maximizada), `btnCerrar` → `cancel.svg`. El botón de cerrar de la lista agrupada de la taskbar (`.taskbarGrupoListaCerrar`, en `context-menu.css`) usa el mismo patrón con `cancel.svg`.

## `TaskBarManager` (`core/Taskbarmanager.js`)

Gestiona los botones de la taskbar. **Agrupa ventanas por extensión** (no una por ventana): si hay más de una del mismo tipo, comparten un botón con badge numérico; el click abre un listado (vía [[Menús Contextuales|ContextMenu]]) con cada título y un botón de cerrar individual.

**Constructor(containerId="taskbarIcons")** — `_grupos: Map<extension, {icono, ventanas: Map<ventanaId,{titulo,onToggle,onCerrar}>}>`.

**Métodos públicos:**
- `agregar(ventanaId, titulo, icono, onToggle, onCerrar)`: agrega la ventana a su grupo (por extensión) y renderiza.
- `eliminar(ventanaId)`: la saca del grupo; si queda vacío, remueve el botón.
- `marcarActivo(ventanaId)` / `desmarcarActivo(ventanaId)`.

**Privados:** `_render`, `_onClick` (si el grupo tiene 1 ventana, toggle directo; si tiene más, abre/cierra listado), `_buscarExtension`, `_extension(titulo)`, `_btnId(extension)`.

Instanciado (singleton por módulo) dentro de `Window.js`.
