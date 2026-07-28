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
- **`bindIconDragStart(container, getDomId)`**: engancha `dragstart` sobre `container`. Llama `idsParaArrastrar(container, getDomId())` para resolver qué ids van en el drag (uno solo o toda la selección), los serializa como JSON en `dataTransfer.setData`, fija `effectAllowed="move"` y usa `setDragImage(getGhostImage(), 0, 0)`.

Usado por [[DesktopManager]]`._crearContenedorIcono`, [[DesktopGrid y DesktopFolder|DesktopFolder]]`._renderIconoEscritorio` y [[Folder]] al crear íconos hijos.

## `seleccionMultiple.js` (`core/seleccionMultiple.js`)

Implementa selección múltiple (ctrl/cmd+click) compartida entre el escritorio real y cualquier ventana de carpeta abierta, con la regla de que **solo un contenedor tiene selección activa a la vez**. También maneja el borrado por tecla `Delete`.

**Estado de módulo (global, no por instancia):** `CLASE = "icono--seleccionado"`, `contenedorActivo`, `seleccionados` (Set).

- **`idDe(el)`** *(interna)*: `el.dataset.iconoId || el.id` — soporta clones del escritorio ([[DesktopGrid y DesktopFolder|DesktopFolder]]) que no tienen `id` propio.
- **Listener global `keydown`**: si es `Delete` (y el foco no está en un input) y hay seleccionados, llama `window.desktopManager?.eliminarIconos(ids)`.
- **`habilitarSeleccion(contenedor, {ignorarVentanas=false})`**: engancha `click` en fase de captura. Si `ignorarVentanas` y el click ocurrió dentro de `.ventana`, no hace nada (deja que el contenedor propio de esa ventana lo resuelva — relevante para el escritorio, que contiene las ventanas de carpetas como descendientes DOM). Con ctrl/cmd: alterna el ícono dentro/fuera de `seleccionados`, cambiando `contenedorActivo` si era otro (limpiando la selección previa).
- **`idsParaArrastrar(container, propioId)`**: si `container` está en la selección activa y hay más de un seleccionado, devuelve todos los ids seleccionados; si no, devuelve `[propioId]`. Consumida por `dragGhost.bindIconDragStart`.
- **`limpiarSeleccion()`**: alias público de limpiar — usado por [[DesktopGrid y DesktopFolder|DesktopGrid]] y [[Folder]] tras completar un drop.

Consumido por [[DesktopManager]] (escritorio raíz), [[Folder]] (contenedor de cada carpeta), [[DesktopGrid y DesktopFolder|DesktopGrid]] y `dragGhost.js`.
