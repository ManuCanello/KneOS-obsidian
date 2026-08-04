---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# Drag and Drop y Selección Múltiple

⬅️ Volver a [[Frontend Core]]

Par de módulos (funciones, no clases) que implementan "seleccionar y arrastrar varios íconos a la vez".

## `dragGhost.js` (`core/dragGhost.js`)

Centraliza (a) la imagen "ghost" transparente usada para reemplazar la vista previa fea que dibuja el navegador al arrastrar, y (b) el cableado estándar de `dragstart` para íconos, integrando la selección múltiple.

- **`getGhostImage()`**: crea (si no existe) un `Image` 1×1 transparente y lo cachea (lazy-singleton). Usado también por [[Window y Taskbar|Window]] (drag de la barra de título).
- **`bindIconDragStart(container, getDomId)`**: engancha `dragstart` sobre `container`. Llama `idsToDrag(container, getDomId())` para resolver qué ids van en el drag (uno solo o toda la selección), los serializa como JSON en `dataTransfer.setData`, fija `effectAllowed="move"` y usa `setDragImage(getGhostImage(), 0, 0)`.

Usado por [[DesktopManager]]`._crearContenedorIcono`, [[DesktopGrid y DesktopFolder|DesktopFolder]]`._renderIconoEscritorio` y [[Folder]] al crear íconos hijos.

## `multiSelect.js` (`core/multiSelect.js`, renombrado desde `seleccionMultiple.js` el 2026-07-29)

Implementa selección múltiple (ctrl/cmd+click) compartida entre el escritorio real y cualquier ventana de carpeta abierta, con la regla de que **solo un contenedor tiene selección activa a la vez**. También maneja el borrado por tecla `Delete`.

> [!note] Traducido al inglés (2026-07-29)
> Identificadores y comentarios de este módulo se tradujeron a inglés (convención del proyecto: código nuevo/refactorizado en inglés aunque el resto de KneOS sea en español — ver `feedback_naming_english`). **No** se tocaron los contratos cruzados con otros archivos: la clase CSS `"icono--seleccionado"` (definida en `icon.css`), la key `dataset.iconoId` (contrato con [[DesktopGrid y DesktopFolder|DesktopFolder]]), el prefijo DOM `"icono"+id` (contrato con [[DesktopManager]]) y la llamada `window.desktopManager?.eliminarIconos(ids)` siguen igual — son superficie de *otros* módulos, no de este.

**Estado de módulo (global, no por instancia):** `SELECTED_CLASS = "icono--seleccionado"`, `activeContainer`, `selected` (Set).

- **`idOf(el)`** *(interna, ex `idDe`)*: `el.dataset.iconoId || el.id` — soporta clones del escritorio ([[DesktopGrid y DesktopFolder|DesktopFolder]]) que no tienen `id` propio.
- **Listener global `keydown`**: si es `Delete` (y el foco no está en un input) y hay seleccionados, llama `window.desktopManager?.eliminarIconos(ids)`.
- **`enableMultiSelect(container, {ignoreWindows=false})`** *(ex `habilitarSeleccion`, opción ex `ignorarVentanas`)*: engancha `click` en fase de captura. Si `ignoreWindows` y el click ocurrió dentro de `.ventana`, no hace nada (deja que el contenedor propio de esa ventana lo resuelva — relevante para el escritorio, que contiene las ventanas de carpetas como descendientes DOM). Con ctrl/cmd: alterna el ícono dentro/fuera de `selected`, cambiando `activeContainer` si era otro (limpiando la selección previa).
- **`idsToDrag(container, ownId)`** *(ex `idsParaArrastrar`)*: si `container` está en la selección activa y hay más de un seleccionado, devuelve todos los ids seleccionados; si no, devuelve `[ownId]`. Consumida por `dragGhost.bindIconDragStart`.
- **`currentSelectionIds(element)`** *(ex `idsSeleccionActual`, 2026-07-29)*: mismo criterio que `idsToDrag` pero devuelve ids **numéricos** (no strings `"iconoN"`), y `null` en vez de `[ownId]` cuando `element` no pertenece a una selección de más de uno — para que el llamador pueda distinguir "no hay selección de grupo" de "esta es la selección". Consumida por `ContextMenuManager._setIconMenu` (ver [[Menús Contextuales]]): el click derecho > Eliminar sobre un ícono que forma parte de la selección activa borra todo el grupo, no solo el clickeado — antes ignoraba la selección por completo.
- **`clearSelection()`** *(ex `limpiarSeleccion`)*: alias público de limpiar — usado por [[DesktopGrid y DesktopFolder|DesktopGrid]], [[Folder]] y [[Menús Contextuales|ContextMenuManager]] tras completar un drop o un borrado múltiple desde el menú contextual.

Consumido por [[DesktopManager]] (escritorio raíz), [[Folder]] (contenedor de cada carpeta), [[DesktopGrid y DesktopFolder|DesktopGrid]], [[Menús Contextuales|ContextMenuManager]] y `dragGhost.js`.
