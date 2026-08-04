---
tags:
  - portfolio/kneos
  - apps
---

# Folder

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Folder.js` — extiende [[File]]. Extensión `"fld"`, ícono `sources/appIcon/folder.svg` (constructor acepta un `icono` opcional — lo usa `DesktopFolder` para pasar `desktop.svg` en su lugar). Clase base de [[DesktopGrid y DesktopFolder|DesktopFolder]].

> [!abstract] Qué hace
> Ventana de explorador de archivos estilo Windows: sidebar con árbol de carpetas expandible ("Este equipo" > "Escritorio" > subcarpetas), breadcrumb editable como barra de direcciones, buscador que filtra por nombre, grid con drag&drop para mover archivos entre carpetas, menú contextual para crear archivos/carpetas, cambiar vista (grande/pequeño/lista) y ordenar (nombre/tipo/tamaño, asc/desc), mensaje de "carpeta vacía", selección múltiple, y navegación con Backspace hacia la carpeta padre.

## Constructor(nombre)

`super()`; crea `contenedor` (`.folderGrid`), habilita drop (`crearDropVentana`) y selección múltiple ([[Drag and Drop y Selección Múltiple|enableMultiSelect]]); crea `ContextMenu` propio ([[Menús Contextuales]]); `files=[]`, `first_open=false`, orden por defecto nombre/asc, `_vistaActual="grande"` — valores iniciales que `_cargarEstiloGuardado()` sobreescribe en la primera apertura si la carpeta ya tiene un estilo guardado en `folder_styles`. Desde 2026-07-31, `super()` pasa `FileType.SYSTEM` (ver [[File]]) en el lugar del viejo parámetro `direction` — [[DesktopGrid y DesktopFolder|DesktopFolder]] lo hereda de acá sin pasar nada propio, ya que su `super()` pasa por este mismo constructor antes de pisar `extension` a `"desktop"`.

## Funciones clave

- **`async abrirVentana()`** (override): `await this._loadContent()` antes de `super.abrirVentana()`.
- **`async _loadContent()`**: si ya cargó o `id==null`, no hace nada; si no, primero llama `_cargarEstiloGuardado()` (ver más abajo), después pide íconos vía `IconServices.getIconsByParent(id)` y por cada uno (si no está ya en el DOM) los crea con `desktopManager.crearIconoHijo(iconData, contenedor)`.
- **`async _cargarEstiloGuardado()`** (2026-07-27): trae el estilo persistido de `folder_styles` (`window.folderStylesServices.getStyle(this.id)`, ver [[Módulo Folder Styles]]) y lo aplica — vista vía `_aplicarVista` (sin re-persistir), criterio resuelto contra `window.folderGroupByOptions`, dirección copiada directo. Si no hay fila guardada, no hace nada. Corre antes de que se arme el DOM por primera vez, así el header/grid ya nacen con el estilo correcto.
- **`_actualizarEstadoVacio()`**: muestra/oculta "Esta carpeta está vacía".
- **`get nombreParaRuta`**: usa `nombre` (no `nombreCompleto`).
- **`crearDrop(elemento)` / `crearDropVentana(elemento)`**: habilitan drop de íconos (soporta múltiples ids); por cada uno llaman `_moverArchivo(archivo, elementoArrastrado)`.
- **`async _moverArchivo(archivo, elemento)`** (2026-07-30, nuevo — factoriza lo que antes era lógica casi idéntica duplicada en `crearDrop` y `crearDropVentana`): `await archivo.animateRemoval()` (ver [[File]] — no hace nada si `archivo` no está visible ahora mismo, p. ej. viene de una carpeta cerrada) y **recién ahí** `this.contenedor.appendChild(elemento)` + `_assignParent(archivo)`. Con esto, mover un archivo entre dos carpetas abiertas anima su desaparición en el origen antes de reaparecer en el destino, en vez de teletransportarse al instante — antes de este cambio la animación solo pasaba al mover **desde el escritorio**; ahora es simétrica para cualquier origen (escritorio → carpeta, carpeta → carpeta) gracias al guard `isConnected` de `animateRemoval()`.
- **`_assignParent(archivo)`**: remueve el archivo de la carpeta padre anterior, ajusta tamaños propagados, asigna `parentId`, persiste (`changeParent`/`changeDesktopPlace`), y llama `archivo.animateAppearance()` (2026-07-30, pop-in en el destino — ver [[File]]).
- **`_crearContenido()` / `_crearContenidoPrincipal()` / `_crearHeaderLista()`**: arman sidebar + contenido + breadcrumb + header de columnas + grid.
- **`_crearSidebar()`**: dos contenedores hijos, `div.folderSidebarTree` (`_buildSidebarTree()`) y `div.folderSidebarFavourites` (`_buildSidebarFavourites()`, opcional — se omite el contenedor entero si no hay favoritos).
  > [!info] Dos contenedores + nombres en inglés (2026-07-29)
  > `_buildSidebarTree()`/`_buildSidebarFavourites()` son código nuevo y usan nombres en inglés a propósito (el resto de `Folder.js` es español) — convención del proyecto: código nuevo en inglés, código viejo se deja como está. `_buildSidebarTree()` arma "Este equipo" → "Escritorio" → árbol de carpetas (`_crearNodoSidebar`, sigue en español porque ya existía). `_buildSidebarFavourites()` filtra `window.archivosAbiertos` por `archivo.favorite === true` (ver [[File]]) y arma un título ("Favoritos") + un ítem plano por archivo (sin jerarquía) que lo abre al click; devuelve `null` si no hay ninguno, y `_crearSidebar()` directamente no agrega el contenedor — mismo criterio que el árbol, que tampoco muestra ramas vacías.
  > Marcar `favorite` se hace desde el menú contextual del ícono real — ver "Agregar/Quitar de favoritos" en [[Menús Contextuales]]. Desmarcarlo se puede hacer desde ahí **o** directo desde acá: cada ítem de `folderSidebarFavourites` tiene su propio menú contextual (2026-07-29, `_abrirMenuQuitarFavorito`, id `folderSidebarFavoriteMenu`, registrado con `bindOutsideClose` en el constructor junto a `VIEW_MENU_ID`) con una única opción "Quitar de favoritos". A diferencia del resto del sidebar (que solo se reconstruye la próxima vez que se abre la ventana), esto sí actualiza el DOM al toque: saca el `div` del ítem y, si era el último favorito, también el contenedor `folderSidebarFavourites` entero (título incluido) — sin esperar a un refresco completo.
  > [!info] "Este equipo"/"Favoritos" con el alto del breadcrumb
  > `.folderSidebarItem--equipo` copia el padding (`10px 16px`)/`font-size` (`20px`) de `.folderBreadcrumb` (ver más abajo) para que la fila tenga el mismo alto que la barra de al lado, más un `border-bottom` que traza la línea divisoria bajo "Este equipo". El título "Favoritos" (dentro de `folderSidebarFavourites`) reusa esa misma clase `--equipo` y le suma `folderSidebarItem--favoritosTitulo` solo para la línea de arriba (`border-top` + `margin-top`) que la separa del árbol.
- **`_crearNodoSidebar(carpeta, nivel, ruta)`**: árbol expandible recursivo, auto-expande el camino hasta la carpeta actual.
- **`_carpetasRaiz()`**: carpetas de `window.archivosAbiertos` sin padre (excluyendo el escritorio).
- **`_crearBreadcrumb()` / `_renderRuta()` / `_activarBarraUrl()` / `_navegarPorUrl(texto)`**: breadcrumb clickeable convertible en barra de direcciones tipo URL.
- **`_crearBuscador()` / `_filtrarArchivos(texto)`**: filtra ítems visibles por nombre — solo entre `this.files` (el contenido directo de **esta** carpeta). No confundir con `TaskbarSearch` (2026-07-30, ver [[Window y Taskbar#`TaskbarSearch` (`core/TaskbarSearch.js`, 2026-07-30, nuevo)|TaskbarSearch]]), la búsqueda global de la taskbar: deliberadamente **no** reusa este método ni depende de `DesktopFolder` — arma su propia lista contra la BD entera.
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
