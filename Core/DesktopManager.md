---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# DesktopManager

⬅️ Volver a [[Frontend Core]]

`public/KneOS/js/core/DesktopManager.js` — orquesta el escritorio completo: creación/edición/borrado de íconos raíz, aplicación de metadata persistida, delega el grid de espacios a [[DesktopGrid y DesktopFolder|DesktopGrid]] y el menú contextual a [[Menús Contextuales|ContextMenuManager]].

## Constructor

Sin parámetros. Al instanciarse:
- Crea `_IconServices = new IconServices()` y lo expone en `window.iconServices`.
- Expone `window.desktopManager = this` (usado por `ContextMenuManager`, [[Folder]], `seleccionMultiple.js` para pedir borrado por tecla `Delete`).
- Cachea `_escritorio` (`#escritorio`) y `_aplicaciones` (`#aplicaciones`).
- Reutiliza/crea `window.archivosAbiertos`.
- Crea `window.desktopFolder = new DesktopFolder()` — singleton, ver [[DesktopGrid y DesktopFolder]].
- Crea `_grid = new DesktopGrid(aplicaciones, _onIconoSoltado)`.
- Llama `habilitarSeleccion(aplicaciones, {ignorarVentanas: true})` — ver [[Drag and Drop y Selección Múltiple]].
- Crea `_contextMenu = new ContextMenuManager(...)` inyectando callbacks (`crearIcono`, `_editarTextoIcono`) e `IconServices`.
- Llama `_bindEventosTeclado()`.

## Métodos públicos

- **`async iniciar()`**: crea la grilla, pide íconos persistidos vía `IconServices.getIcons()` (o usa `defaultIcons` si vacío), garantiza que exista el ícono "Escritorio", y por cada ítem llama `crearIcono(...)` con `edit=false`.
- **`async crearIcono(iconoCss, divClickeado, tipo, name, edit=true, idExistente=null, srcExistente=null, metaExistente={})`**: crea un ícono en la raíz. Resuelve `Class`/`css` vía `getIconMeta(tipo)` ([[Frontend Model Services Utils#Model]]); si `tipo==='desktop'` reutiliza el singleton `window.desktopFolder`. Si no hay `idExistente`, persiste el nuevo ícono. Decide `espacioId` (el clickeado o `_grid.buscarVacio()`), persiste la posición, y si `edit` habilita edición inmediata del nombre.
- **`crearIconoHijo(iconData, contenedorPadre)`**: crea un ícono ya persistido dentro de una carpeta (usado por `Folder._loadContent`). Devuelve la instancia creada.
- **`async crearIconoEnCarpeta(iconoCss, tipo, name, carpeta)`**: crea un ícono nuevo directo dentro de una carpeta (menú "Nuevo" de `Folder`).
- **`eliminarIconos(ids)`**: delega en `_contextMenu.eliminarIconos(ids)`.

## Métodos privados relevantes

- **`_onIconoSoltado(idIcono, espacioId)`**: callback de `DesktopGrid` al soltar un ícono. Actualiza `desktopPlace`; si el archivo venía de una subcarpeta, lo saca del `files` del padre, propaga el descuento de tamaño (`_propagarTamano`, ver [[File]]), limpia `parentId` y persiste el nuevo padre.
- **`_crearContenedorIcono`**: construye el DOM de un ícono, habilita drag (`dragGhost.bindIconDragStart`) y click para `abrirVentana()`.
- **`_actualizarColumnas` / `_aplicarMeta`**: refrescan columnas visuales y copian metadata de BD al objeto `archivo`.
- **`_finalizarIcono`**: asigna `id`, registra en `window.archivosAbiertos`, registra el menú contextual del ícono.
- **`_editarTextoIcono`**: activa `contentEditable` para renombrar inline, persiste vía `IconServices.changeName`.
- **`_bindEventosTeclado`**: reenvía `keydown`/`keyup` del `window` interno al `window.parent` vía `postMessage` — usado por [[Escena 3D]] para animar el teclado 3D.
