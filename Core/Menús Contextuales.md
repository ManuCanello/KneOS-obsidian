---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# Menús Contextuales

⬅️ Volver a [[Frontend Core]]

Tres piezas: un motor genérico (`ContextMenu`) y dos consumidores de dominio (`ContextMenuManager`, `TaskbarContextMenu`).

## `ContextMenu` (`core/ContextMenu.js`) — motor genérico

Building blocks reutilizables por *cualquier* menú contextual del sistema (usado también por [[Window y Taskbar|TaskBarManager]] y [[Folder]]). Único estado propio: `_menusAbiertos`, un `Set` **a nivel de módulo** (no de instancia) con los ids de los menús principales actualmente abiertos — compartido entre *todas* las instancias de `ContextMenu`, cada una de las cuales pertenece a un manager de dominio distinto (`Folder` crea la suya en su constructor, `ContextMenuManager` y `TaskbarContextMenu` la suya propia). Sin ese registro cruzado, dos instancias no se enteran una de la otra: abrir el menú de una `Folder` y, sin cerrarlo, el de un ícono adentro de esa carpeta, dejaba los dos abiertos a la vez (bug real, 2026-07-28).

- **`open(menuId, container, x, y, anchor="top")`**: llama a `ContextMenu.closeAll()` (cierra todo lo que hubiera abierto, propio o de otra instancia) antes de crear `div#menuId.menuClickDerecho` posicionado en `(x,y)` (`anchor="bottom"` para menús que cuelgan hacia arriba, como la taskbar). Registra el id en `_menusAbiertos`. Devuelve el elemento.
- **`close(menuId)`**: remueve el elemento y lo saca de `_menusAbiertos`.
- **`static closeAll()`**: cierra todos los menús principales registrados, sin importar qué instancia los abrió. También la llama `Window.traerAlFrente()` (ver [[Window y Taskbar]]): abrir/enfocar una ventana cierra cualquier menú contextual que hubiera quedado abierto — necesario porque el click en un ícono hace `stopPropagation()`, así que nunca le llega al listener de "click afuera" de `bindOutsideClose()` (bug real, 2026-07-28: abrir un menú y después una app dejaba el menú abierto).
- **`addSubmenu(parentMenu, submenuId, triggerItem?)`**: cierra submenús hermanos abiertos, crea uno nuevo. Si se pasa `triggerItem` (el `div.menuClickDerechoItem` que abrió el submenú, ver abajo) **y no es el primer ítem** (`offsetTop > 0`), lo alinea a esa misma altura (`submenu.style.top = triggerItem.offsetTop + "px"`) en vez del `top: -4px` fijo de la clase CSS `.menuClickDerechoCrearSecundario`. Para el primer ítem se deja el `-4px` de la clase tal cual: ese valor ya está pensado para compensar el borde y alinear con el tope del menú padre — forzar `top: 0px` ahí rompería esa compensación (2026-07-28). Sin `triggerItem` cae al comportamiento viejo (compatibilidad).
- **`addItem(container, label, onClick, {icon, iconSize=48, closeMenuId, selected})`**: crea un ítem clickeable, con ícono opcional (SVG con `fill="currentColor"` pintado vía `mask-image`, ver [[Window y Taskbar]]) y marcador `selected` (usado por `Folder` para el criterio de orden activo). `iconSize` (2026-07-28) es lo único que varía el tamaño del ícono: lo usa `Folder` para que "Íconos grandes"/"Íconos pequeños" compartan el mismo SVG pero se vean distinto (ver [[Folder]]). `onClick` recibe el propio elemento del ítem como argumento (`onClick(item)`) — los handlers que no lo necesitan simplemente lo ignoran (parámetro no declarado); los que abren un submenú lo reenvían a `addSubmenu` como `triggerItem` (ver arriba). Los tres puntos donde esto se usa: `ContextMenuManager._abrirSubMenuCrear`, las tres de `Folder` (`_abrirSubMenuCrear`/`Ver`/`Ordenar`) y `TaskbarContextMenu._openAlignSubmenu`.
- **`addSeparator(container)`**.
- **`bindOutsideClose(menuId)`**: cierra el menú si se clickea afuera.

## `ContextMenuManager` (`core/ContextMenuManager.js`)

Menú del escritorio: fondo vacío → "Nuevo" (submenú Documento/Carpeta); ícono → Abrir/Renombrar/Eliminar (Eliminar condicional, ver abajo), incluyendo borrado en cascada.

**Constructor(escritorio, onCrearIcono, editarNombre, iconServices)** — no conoce [[DesktopManager]] directamente, recibe callbacks.

- **`async eliminarIconos(ids)`**: por cada id, `await this._deleteIcon(...)` secuencial.
- **`_setIconMenu(icon, id)`**: engancha `contextmenu` en el ícono. Siempre agrega Abrir/Renombrar; "Eliminar" solo si la extensión del ícono no está en `iconsUndeletable` (2026-07-29, ver nota abajo) — se omite el ítem en vez de mostrarlo deshabilitado o dejar que falle, mismo criterio que el resto del proyecto de no mostrar feedback de error para algo que el usuario no puede hacer.
- **`_bindEventos()`**: si el click-derecho fue sobre un `.espacioParaShortCut` vacío, abre "Nuevo" (`_abrirSubMenuCrear`), con ícono `more.svg` (mismo que usa `Folder` para su propio "Nuevo").
- **`async _deleteIcon(icon, id)`**: primero chequea `iconsUndeletable.has(archivo?.extension)` y corta en silencio si es undeletable (defensa en profundidad — cubre `eliminarIconos`/borrado múltiple, que no pasa por el menú contextual individual y por lo tanto no tiene ítem que ocultar). Si pasa, borrado **recursivo**: obtiene hijos vía `iconServices.getIconsByParent(id)` y los borra primero, luego el propio (ver [[Módulo Icon]]). Si tiene éxito: cierra su ventana si estaba abierta (`archivo?.cerrarVentana()`, 2026-07-27 — seguro llamarlo aunque ya esté cerrada, `Window.cerrar()` no hace nada sin `_ventanaEl`), remueve el DOM, lo saca de `window.archivosAbiertos` y de `files` del padre, y descuenta su tamaño de toda la cadena ancestro. Por ser recursivo, esto cierra también las ventanas de subcarpetas/archivos hijos abiertos.

> [!info] `iconsUndeletable` (`model/iconsUndeletable.js`, 2026-07-29)
> `Set(["desktop", "exe", "kmd", "kfruit", "ai"])` — las extensiones de los 5 íconos de [[defaultIcons.js|Frontend Model Services Utils#Model]] (Escritorio, Doom, Terminal, KFruit, Kne). Ninguna es recreable desde el menú "Nuevo" (que solo ofrece `txt`/`fld`), así que borrarlas las perdería para siempre — incluye el ícono raíz "Escritorio" (`ext: "desktop"`), cuya pérdida además rompería la navegación de [[Folder]]. `txt`/`fld` quedan afuera a propósito, son recreables. Probado en navegador: KFruit no ofrece "Eliminar" en su menú, una carpeta creada por el usuario sí.

## `TaskbarContextMenu` (`core/TaskbarContextMenu.js`)

Menú click-derecho de la barra de tareas — por ahora solo alinea la sección de íconos.

**Constructor(barraId="barraDeTarea", iconsContainerId="taskbarIcons")**.

- `_openMenu(e)`: abre anclado por abajo (`anchor="bottom"`), un único ítem "Ver" → `_openAlignSubmenu`.
- `_openAlignSubmenu(menu)`: ítems "Izquierda"/"Centro" → `_setAlignment`.
- `_setAlignment(position)`: alterna clases `taskbarIcons--left`/`taskbarIcons--center`.

Instanciado una sola vez en `KNEOS.js`, independiente de `DesktopManager`.
