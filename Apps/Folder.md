---
tags:
  - portfolio/kneos
  - apps
---

# Folder

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Folder.js` — extiende [[File]]. Extensión `"fld"`, ícono `sources/icon/folder.svg` (constructor acepta un `icono` opcional — lo usa `DesktopFolder` para pasar `desktop.svg` en su lugar). Clase base de [[DesktopGrid y DesktopFolder|DesktopFolder]].

> [!abstract] Qué hace
> Ventana de explorador de archivos estilo Windows: sidebar con árbol de carpetas expandible ("Este equipo" > "Escritorio" > subcarpetas), breadcrumb editable como barra de direcciones, buscador que filtra por nombre, grid con drag&drop para mover archivos entre carpetas, menú contextual para crear archivos/carpetas, cambiar vista (grande/pequeño/lista) y ordenar (nombre/tipo/tamaño, asc/desc), mensaje de "carpeta vacía", selección múltiple, y navegación con Backspace hacia la carpeta padre.

## Constructor(nombre)

`super()`; crea `contenedor` (`.folderGrid`), habilita drop (`crearDropVentana`) y selección múltiple ([[Drag and Drop y Selección Múltiple|habilitarSeleccion]]); crea `ContextMenu` propio ([[Menús Contextuales]]); `files=[]`, `first_open=false`, orden por defecto nombre/asc, `_vistaActual="grande"` — valores iniciales que `_cargarEstiloGuardado()` sobreescribe en la primera apertura si la carpeta ya tiene un estilo guardado en `folder_styles`.

## Funciones clave

- **`async abrirVentana()`** (override): `await this._loadContent()` antes de `super.abrirVentana()`.
- **`async _loadContent()`**: si ya cargó o `id==null`, no hace nada; si no, primero llama `_cargarEstiloGuardado()` (ver más abajo), después pide íconos vía `IconServices.getIconsByParent(id)` y por cada uno (si no está ya en el DOM) los crea con `desktopManager.crearIconoHijo(iconData, contenedor)`.
- **`async _cargarEstiloGuardado()`** (2026-07-27): trae el estilo persistido de `folder_styles` (`window.folderStylesServices.getStyle(this.id)`, ver [[Módulo Folder Styles]]) y lo aplica — vista vía `_aplicarVista` (sin re-persistir), criterio resuelto contra `window.folderGroupByOptions`, dirección copiada directo. Si no hay fila guardada, no hace nada. Corre antes de que se arme el DOM por primera vez, así el header/grid ya nacen con el estilo correcto.
- **`_actualizarEstadoVacio()`**: muestra/oculta "Esta carpeta está vacía".
- **`get nombreParaRuta`**: usa `nombre` (no `nombreCompleto`).
- **`crearDrop(elemento)` / `crearDropVentana(elemento)`**: habilitan drop de íconos (soporta múltiples ids), reasignando padre con `_assignParent`.
- **`_assignParent(archivo)`**: remueve el archivo de la carpeta padre anterior, ajusta tamaños propagados, asigna `parentId`, persiste (`changeParent`/`changeDesktopPlace`).
- **`_crearContenido()` / `_crearContenidoPrincipal()` / `_crearHeaderLista()`**: arman sidebar + contenido + breadcrumb + header de columnas + grid.
- **`_crearSidebar()` / `_crearNodoSidebar(carpeta, nivel, ruta)`**: árbol expandible recursivo, auto-expande el camino hasta la carpeta actual.
- **`_carpetasRaiz()`**: carpetas de `window.archivosAbiertos` sin padre (excluyendo el escritorio).
- **`_crearBreadcrumb()` / `_renderRuta()` / `_activarBarraUrl()` / `_navegarPorUrl(texto)`**: breadcrumb clickeable convertible en barra de direcciones tipo URL.
- **`_crearBuscador()` / `_filtrarArchivos(texto)`**: filtra ítems visibles por nombre.
- **`_rutaCarpetas()`**: cadena de ancestros desde la raíz.
- **`_bindViewMenu()` / `_permiteCrear()` / `_abrirSubMenuCrear` / `_abrirSubMenuVer` / `_abrirSubMenuOrdenar`**: menú contextual "Nuevo" (delega en `desktopManager.crearIconoEnCarpeta`), "Ver", "Ordenar por".
  > [!info] "Ordenar por" y "Ver" son data-driven (2026-07-27)
  > Ninguno de los dos submenús tiene sus opciones hardcodeadas: "Ordenar por" itera `window.folderGroupByOptions` (`{folder_group_by_id, folder_group_by_desc}`, vía [[Frontend Model Services Utils#Services|FolderGroupByServices]] → [[Módulo Folder Group By]]) y "Ver" itera `window.folderViewsOptions` (`{folder_view_id, folder_view_desc}`, vía `FolderViewsServices` → [[Módulo Folder Views]]), ambos cargados una sola vez al arranque en `KNEOS.js`. Si el fetch falla o aún no llegó, cada uno cae a su propio array de respaldo hardcodeado (`ORDENAR_OPCIONES_POR_DEFECTO` / `VISTA_OPCIONES_POR_DEFECTO`).
  >
  > Las dos usan una estrategia distinta para pasar de la descripción de la BD a la clave interna: "Ordenar por" **normaliza el texto** (`_criterioDesdeDescripcion`: "Tamaño" → "tamano", matchea los comparadores de `_aplicarOrden`); "Ver" **mapea por id** (`ID_A_VISTA = {1:"grande", 2:"pequeno", 3:"lista"}`), porque "Íconos grandes" no se puede normalizar mecánicamente a "grande". `VISTA_A_ID` (usado al persistir, ver más abajo) se deriva invirtiendo `ID_A_VISTA`.

  > [!info] Íconos del menú contextual (2026-07-28)
  > Los tres ítems de nivel superior tienen ícono fijo en el código: "Nuevo" → `more.svg`, "Ver" → `see.svg`, "Ordenar por" → `group-by.svg` (mismo trío para el "Nuevo" del escritorio en `ContextMenuManager.js`). Las opciones de **"Ver"** en cambio traen su ícono de la BD (`icon_path` de [[Módulo Folder Views]], con fallback en `VISTA_OPCIONES_POR_DEFECTO`); "grande" y "pequeño" comparten el mismo `file.svg` pero se pintan a distinto tamaño vía `TAMANO_ICONO_VISTA = {grande: 56, pequeno: 28}` (el `iconSize` nuevo de `ContextMenu.addItem`, default 48). "Ordenar por" no tiene íconos por opción, solo en el ítem padre.
- **`_ordenarPor(criterio)` / `_ordenarDireccion(dir)` / `_aplicarOrden()`**: reordena por comparadores.
- **`_elementoDeOrden(archivo)`** (overrideable, distinto en `DesktopFolder`, que usa clones).
- **`_setVista(vista)`** / **`_aplicarVista(vista)`**: `_setVista` = `_aplicarVista` (cambia clases de grid/lista/header) + persistir; `_aplicarVista` sola es la usada por `_cargarEstiloGuardado` para restaurar sin re-guardar.
  > [!info] El estilo se persiste y se carga (actualizado 2026-07-27)
  > `_setVista`, `_ordenarPor` y `_ordenarDireccion` llaman a `_persistirEstilo()` al final, que manda el estado completo (vista + criterio + dirección) a `window.folderStylesServices.saveStyle(...)` → tabla `folder_styles` (ver [[Módulo Folder Styles]]). No hace nada si `this.id == null` (p. ej. `DesktopFolder`, que no tiene ícono propio en BD). Al abrir la carpeta por primera vez, `_cargarEstiloGuardado()` (llamado desde `_loadContent`) trae ese estilo y lo aplica — ya no se resetea siempre a nombre/asc/grande.
- **`actualizarSrc(srcPadre)`** (override): propaga el nuevo `src` a todo el contenido.
- **Listener global `Backspace`**: navega a la carpeta padre de la ventana de carpeta activa.

## Persistencia

[[Frontend Model Services Utils#Services|IconServices]] → [[Módulo Icon]].
