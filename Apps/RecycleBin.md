---
tags:
  - portfolio/kneos
  - apps
---

# RecycleBin

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/RecycleBin.js` — extiende [[File]]. Extensión `"recyclebin"`, ícono propio `sources/appIcon/trash.svg` (SVG con `fill="currentColor"`, pintado por `mask-image` igual que el resto — ver `iconoStyle.js` en [[Frontend Model Services Utils]]), `src = null`, `size = 0`. Está en `filesUndeletable` (ver [[Menús Contextuales]]) y sembrada en `defaultFiles.js` (`espacio61`) — no se puede borrar ni recrear desde "Nuevo".

> [!abstract] Qué hace
> Papelera de reciclaje real: todo lo que antes se borraba de una (menú contextual "Eliminar" o tecla `Delete`) ahora hace un soft delete (`deleted_at` en la fila `files`, ver [[Módulo Icon]]) y queda listado acá hasta que se restaura o se elimina definitivamente.

## Diseño: por qué es una lista propia, no una `Folder`

A diferencia de [[Folder]], los ítems de la papelera **no son instancias `File`/`Folder` vivas** — cuando algo se manda a la papelera, `ContextMenuManager._deleteIcon` ya corrió su borrado en cascada normal (cierra ventana, saca de `window.archivosAbiertos`, anima y remueve del DOM); lo único que cambió es que el backend hace `UPDATE deleted_at` en vez de `DELETE`. `RecycleBin` no reconstruye esas instancias: pide `IconServices.getRecycleBinIcons()` (filas planas con `id_icon/name/ext/src/size/deleted_at`, ver [[Módulo Icon]]) y renderiza una fila liviana por ítem (reutiliza las clases `.icono`/`.iconoColumna` de `folder.css`, forzando `folderGrid--lista` — no tiene sentido una vista de íconos grandes para una lista de metadata).

> [!success] Se refresca sola al mandar algo a la papelera (2026-07-31)
> `window.recycleBin` (singleton, seteado en `DesktopManager._finalizarIcono` igual que `window.desktopFolder`) es el único punto por el que `ContextMenuManager.eliminarIconos` — el único punto de entrada real de "mandar a la papelera": tecla `Delete`, menú "Eliminar" (single o selección múltiple, unificados en una sola llamada desde 2026-07-31) y el drop sobre el ícono de la Papelera — le avisa a esta clase que refresque. `refrescar()` vacía `this.contenedor`/`this._items`, resetea `_cargado = false` y vuelve a llamar `_cargarPapelera()` (repite el `getRecycleBinIcons()` completo en vez de intentar armar a mano el ítem nuevo — más simple y evita duplicar la lógica de "solo raíces" que ya resuelve el backend, ver [[Módulo Icon]]). Se llama **siempre** después de cada borrado, sin importar si la ventana de la Papelera está abierta ahora mismo — antes de este fix, `_cargado` se ponía en `true` la primera vez que se abría y nunca se volvía a pedir la lista, así que ni reabriendo la ventana se veía lo último borrado.

## Constructor(nombre)

`super(nombre, "recyclebin", null, "sources/appIcon/trash.svg", FileType.SYSTEM, 0)` (2026-07-31: el 5º parámetro era `direction`, dead param, ahora es `FileType.SYSTEM` — ver [[File]]). Crea `this.contenedor` (el `div.folderGrid.folderGrid--lista.recycleBinLista`) **una sola vez**, igual patrón que `Folder`/`DesktopFolder`: persiste entre aperturas/cierres de la ventana aunque `Window.cerrar()` destruya el DOM de la ventana entera (ver [[Window y Taskbar]]), así no hace falta re-pedir la lista cada vez que se reabre. `this._items` (paralelo a `Folder.files`) guarda `{item, fila}` por cada fila renderizada.

## `abrirVentana()` / `_cargarPapelera()`

Override que espera `_cargarPapelera()` antes de `super.abrirVentana()` — mismo patrón que `Folder.abrirVentana()`/`_loadContent()`. Guard `_cargado` (equivalente a `Folder.first_open`): solo pide `getRecycleBinIcons()` la primera vez.

> [!bug] Ciclo de imports con `iconSrc.js` (2026-07-31, resuelto el mismo día; el ciclo en sí quedó estructuralmente imposible desde 2026-08-13)
> `iconSrc.js` importaba **estáticamente** la clase `RecycleBin` (para el registro `recyclebin: {css, Class}`, igual que hacía con las otras 7 apps de ese momento). La primera versión de esta clase también importaba `getIconMeta` de `iconSrc.js` arriba de todo (estático, para resolver el ícono de cada fila por extensión) — eso cerraba un ciclo `iconSrc.js → RecycleBin.js → iconSrc.js`. A diferencia de las otras apps (que no le importaban nada a `iconSrc.js`), acá el ciclo rompía en runtime: `Uncaught ReferenceError: Cannot access 'RecycleBin' before initialization` en `iconSrc.js`, porque cuando `iconSrc.js` llegaba a armar `iconsSrc` (línea `recyclebin: {..., Class: RecycleBin}`), todavía estaba en medio de resolver su propio import de `RecycleBin.js`, que a su vez quedaba a mitad de camino esperando el import circular de vuelta a `iconSrc.js` — la clase `RecycleBin` nunca llegaba a inicializarse (TDZ) antes de que `iconSrc.js` intentara leerla. Fix de entonces: sacar el `import { getIconMeta } from "../model/iconSrc.js"` estático y reemplazarlo por un `import()` **dinámico** dentro de `_cargarPapelera()`, resuelto (y pasado como parámetro a `_agregarFila`) recién cuando el usuario abre la ventana — para ese momento el grafo de módulos ya terminó de cargar por completo, así que no había ciclo que romper.
>
> *(2026-08-13)* `iconSrc.js` dejó de importar **cualquier** clase de app de forma estática — cada entrada de `iconsSrc` pasó a `{css, load}` con `load: () => import(...)` (ver [[Deuda Técnica#Apps pesadas (Maxwell/KFruit) ya no se instancian en cada boot de KneOS (2026-08-13, resuelto tras evaluarse como pendiente en el mismo pase de performance de arriba)]]). El ciclo descrito arriba ya no puede volver a cerrarse por este camino específico — no queda ningún `import` estático de `RecycleBin.js` (ni de ninguna otra app) en `iconSrc.js` con el cual cerrarlo. El import dinámico de `getIconMeta` en `_cargarPapelera()` se dejó igual (no hacía falta tocarlo, sigue siendo el patrón correcto para el caso general).

## Contenido: `_crearBarraHerramientas()` + `_crearHeaderLista()` + fila por ítem

- **Toolbar** (`recycleBinToolbar`/`.recycleBinToolbarBtn`, `recyclebin.css`): un único botón "Vaciar papelera" → `_vaciarPapelera()`.
- **Header de lista** (reusa `.folderListaHeader`/`.folderListaHeaderCol` de `folder.css`, siempre visible — no hay otra vista): columnas Nombre / Fecha de eliminación / Tipo / **Ubicación original** (columna nueva, `.iconoColumna--ubicacion`, `recyclebin.css`) / Tamaño.
- **`_agregarFila(item, getIconMeta)`**: arma la fila (`div.icono` + `.iconoImagen` con el ícono real de la extensión) y sus columnas — `_ubicacionOriginal(item.src)` corta el último segmento del `src` guardado (mismo campo que ya usa "Copiar ruta de acceso", ver [[Menús Contextuales]]) para mostrar solo la carpeta contenedora.
  - **Click simple → Propiedades (2026-07-31):** `_abrirPropiedades(item, css)` — ver más abajo.
  - **Ctrl+click → selección múltiple propia (2026-07-31):** `_toggleSeleccion(fila)`, clase `.icono--seleccionado` (la misma de `icon.css` que usa el resto del sistema). **No** reutiliza `multiSelect.js` (el módulo compartido por escritorio/carpetas): ese módulo liga la tecla `Delete` y el drag directo a `desktopManager.eliminarIconos` (soft delete), acción que no tiene sentido sobre un ítem que ya está en la papelera — acá la acción de grupo es Restaurar/Eliminar definitivamente, nunca "volver a borrar". Clickear el fondo vacío de `this.contenedor` limpia la selección (`_limpiarSeleccion`).
  - **Clic derecho → menú de grupo:** `_grupoParaAccion(item, fila)` decide sobre qué actuar — si la fila clickeada es parte de una selección activa de más de una, toda la selección; si no, solo esa fila (mismo criterio que `currentSelectionIds` de `multiSelect.js`, pero implementado acá porque el Set de seleccionados es propio). El menú (`_abrirMenuItem`) siempre ofrece **Restaurar** / **Eliminar definitivamente**, sin importar el tamaño del grupo.

## Propiedades (`_abrirPropiedades`)

`FileProperties.abrirPara(item.id_icon, {...item, icono: css})` (ver [[Menús Contextuales#`FileProperties`|FileProperties]]) — un ítem de la papelera no es una instancia `File` viva (no está en `window.archivosAbiertos`), así que se le pasa una copia plana de sus datos con el ícono ya resuelto (el mismo `css` que ya calculó `_agregarFila` vía `getIconMeta`, para no tener que importar `iconSrc.js` en `FileProperties.js` — ver el bug de ciclo de imports más abajo). `FileProperties` reconoce que no hay instancia viva y arma una vista de **solo lectura** (sin input de nombre editable, footer con un único botón "Cerrar" en vez de Aceptar/Cancelar).

> [!info] Import dinámico de `FileProperties.js` — prevención, no un bug real (2026-07-31)
> A diferencia del ciclo de `iconSrc.js` de arriba (ese sí rompió en runtime), acá nunca hubo un crash que arreglar — `_abrirPropiedades` se escribió con `import()` **dinámico** desde el primer commit de esta función, por las dudas, antes de que existiera ningún bug que lo motivara. La razón de la cautela: mismo patrón de riesgo que el de `iconSrc.js` — si `RecycleBin.js` importara `FileProperties.js` **estáticamente**, y `FileProperties.js` importara `getIconMeta` de `iconSrc.js` también estático, se cerraría `iconSrc.js → RecycleBin.js → FileProperties.js → iconSrc.js`. En los hechos el ciclo ni siquiera llegaría a cerrarse: `FileProperties.js` no necesita `iconSrc.js` en absoluto (recibe el ícono ya resuelto en el snapshot que le pasa `RecycleBin`). El `import()` dinámico quedó igual, como margen de seguridad barato ante quien toque `FileProperties.js` a futuro y le agregue esa dependencia sin pensar en el ciclo.

## Restaurar / Eliminar definitivamente / Vaciar papelera

Las tres operan sobre un **grupo** (`Array<{item, fila}>`, ver `_grupoParaAccion` arriba) — uno solo o varios a la vez:

- **`_restaurar(grupo)`**: por cada `{item, fila}`, `IconServices.restoreIcon(id_icon)` (limpia `deleted_at` de todo el subárbol en BD, ver [[Módulo Icon]]) → si OK, saca esa fila de acá y llama `window.desktopManager.restaurarIcono(item)` (ver [[DesktopManager]]) para reconstruir el ícono real en su ubicación original.
- **`_purgar(grupo)`**: por cada entrada, `IconServices.purgeIcon(id_icon)` (el viejo borrado en cascada real, `deleteIconRecursivo`, ver [[Módulo Icon]]) → junta las que dieron OK y las saca todas juntas de acá (no hay ventana/ícono vivo que limpiar, ya se limpiaron cuando se mandó a la papelera).
- **`vaciarPapelera()`** (pública, ex `_vaciarPapelera` — renombrada 2026-07-31 al dejar de ser de uso exclusivo del botón de la toolbar): `await this._cargarPapelera()` y después `_purgar([...this._items])` — todo lo listado, sin importar la selección. El `await` extra es necesario porque también puede llegar sin haber abierto la ventana antes esta sesión — sin ese `await`, `this._items` estaría vacío (nunca se pidió `getRecycleBinIcons()`) y "vaciar" no purgaría nada aunque la papelera tuviera contenido real en BD. Al estar `_cargarPapelera` guardado por `_cargado`, si la ventana ya se abrió antes esto no repite el fetch. **Ya no es el punto de entrada del usuario** (ver `confirmarVaciarPapelera` abajo) — vacía directo, sin preguntar nada.
- **`confirmarVaciarPapelera(x, y)`** (2026-07-31): el punto de entrada real para el usuario, tanto desde el botón de la toolbar (`e.clientX/clientY` del click) como desde el ítem "Vaciar papelera" del menú contextual del propio ícono (ver [[Menús Contextuales]], posicionado con el `getBoundingClientRect()` de ese ítem). Abre un `ContextMenu` más (`CONFIRMAR_VACIAR_MENU_ID`) con dos opciones — **"Sí, vaciar la papelera"** (llama `vaciarPapelera()`) y **"Cancelar"** (no hace nada) — en vez de un `window.confirm()` nativo: no hay ningún `confirm()`/`alert()` nativo en todo el proyecto, porque rompería la estética retro monocromática (ver [[Reglas]]); un `ContextMenu` más mantiene el mismo idioma visual que ya usa el resto del sistema para cualquier decisión del usuario.
- **`_quitarFilas(entries)`** *(interna)*: saca cada fila del DOM, de `this._items` y de `this._seleccionados`, y refresca el estado vacío una sola vez al final.
- **`_actualizarEstadoVacio()`**: mismo patrón que `Folder._actualizarEstadoVacio` (mensaje `.folderVacio` si `this._items` queda en 0).

## Drop targets — `_habilitarDrop(elemento)` / `crearDropEliminar(elemento)`

Dos lugares aceptan soltar archivos, ambos delegando en el mismo `_habilitarDrop(elemento)` *(interna)* — mismo patrón que `Folder.crearDrop`/`crearDropVentana` (ícono en el escritorio vs. contenido de la ventana ya abierta), pero acá compartiendo una sola implementación porque la acción es idéntica sin importar el elemento sobre el que se soltó:

- **El ícono de la Papelera en el escritorio**: `crearDropEliminar(elemento)`, wireado desde `DesktopManager._finalizarIcono` (`if (archivo instanceof RecycleBin) archivo.crearDropEliminar(container)`, análogo al `if (archivo instanceof Folder) archivo.crearDrop(container)` ya existente).
- **El propio `this.contenedor` (la lista, con la ventana ya abierta)** (2026-07-31): `this._habilitarDrop(this.contenedor)` se llama directo en el constructor — soltar un archivo adentro de la ventana de la Papelera ya abierta manda igual a la papelera, sin tener que apuntar al ícono del escritorio.

Arrastrar uno o varios íconos (selección múltiple incluida, mismo payload `text/plain` JSON que el resto del drag&drop, ver [[Drag and Drop y Selección Múltiple]]) y soltarlos en cualquiera de los dos llama `window.desktopManager.eliminarIconos(ids)` — **reutiliza el mismo camino** que "Eliminar" del menú contextual (cascada de hijos, rescate de indestructibles, cierre de ventana/taskbar, descuento de tamaño en ancestros — ver [[Menús Contextuales]]), sin duplicar nada de esa lógica.

## Persistencia

Ver [[Módulo Icon]]:
- `trashIcon(id_icon, pc_id)` / `restoreIcon(id_icon, pc_id)`: `$executeRaw` con `WITH RECURSIVE` que marca/limpia `deleted_at` en todo el subárbol de una sola query.
- `getDeletedIcons(pc_id)`: solo las **raíces** de cada subárbol borrado (no cada descendiente suelto).
- `purgeIcon(id_icon, pc_id)`: ex `deleteIcon`, el borrado en cascada real (`deleteIconRecursivo`), ahora reservado a ítems que ya están en la papelera.
