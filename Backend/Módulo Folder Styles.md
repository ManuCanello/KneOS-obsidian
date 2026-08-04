---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Folder Styles

⬅️ Volver a [[Backend]]

Persiste el "estilo" completo de una carpeta (vista + criterio de orden + dirección), una fila por carpeta, para que sobreviva a cerrar/reabrir la ventana o recargar la página (algo que antes no pasaba, ver la nota de estado en memoria en [[Folder]]).

## Endpoints (`routes/folderStylesRoutes.js`, montado en `/folderStylesRoutes`, `requireAuth` en todo el router desde 2026-07-28)

| Método | Path | Controller |
|---|---|---|
| GET | `/:folder_id` | `getFolderStyle` |
| PATCH | `/` | `editFolderStyle` |

`PATCH` hace de upsert (crea si no existe, actualiza si ya hay una fila para ese `folder_id`). `GET` devuelve la fila (o `null` si la carpeta nunca guardó un estilo, **o si la carpeta no es del `pc_id` del token** — ver más abajo).

> [!warning] Era el único módulo sin ningún concepto de `pc_id` — cerrado 2026-07-28
> Antes de esto, `folder_styles` no manejaba `pc_id` en ningún lado: ni el controller (que además era el único de todo el proyecto sin validación de entrada) ni el modelo. Cualquiera que supiera un `folder_id` podía leer o pisar el estilo guardado de una carpeta ajena, sin necesidad de conocer ni suplantar ninguna sesión. Al implementar el JWT de sesión se agregó ownership derivado del join que ya existía en el schema: `folder_styles.folder_id → files.id_icon` (tabla `files`, ex `icons` — renombrada 2026-07-29, ver [[Módulo Icon]]), y `files.pc_id` es el dueño real. Mismo idiom que ya usaba [[Módulo Txt]] (`getTxtContent`, filtro anidado `icons: {pc_id}` — el campo de relación sigue llamándose `icons` en el schema aunque el modelo es `files`). Ver [[Deuda Técnica]].

## Controller (`controllers/folderStylesController.js`)

- **`getFolderStyle`**: lee `folder_id` de `req.params` + `req.pcId` del token. Valida `folder_id` con `isValidId` (2026-07-28 — antes no validaba nada). Llama `getFolderStyleByFolderId(folder_id, req.pcId)`, devuelve la fila o `null` (carpeta sin estilo guardado **o** carpeta que no pertenece al `pc_id` — mismo `null` para ambos casos; el frontend ya lo trata como "no hay estilo guardado" en cualquiera de los dos).
- **`editFolderStyle`**: recibe `folder_id, folder_view, folder_group_by, folder_group_order` del body + `req.pcId` del token. Valida (2026-07-28) `folder_id`/`folder_view`/`folder_group_by` con `isValidId` y `folder_group_order` contra el set `{"asc","desc"}` — los únicos valores que produce/consume el frontend. Llama `saveFolderStyle(folder_id, req.pcId, ...)`; si el modelo devuelve `null` (la carpeta no es del `pc_id`), `404 { error: "Carpeta no encontrada" }`; si no, devuelve la fila resultante.

## Modelo (`models/folderStylesModel.js`)

- **`getFolderStyleByFolderId(folder_id, pc_id)`** *(2026-07-28: firma cambiada)*: `prisma.folder_styles.findFirst({where:{folder_id, icons: {pc_id}}})` — filtro anidado vía la relación `folder_styles.icons`.
- **`saveFolderStyle(folder_id, pc_id, folder_view, folder_group_by, folder_group_order)`** *(2026-07-28: firma cambiada + reescrito a upsert atómico)*: primero una sonda de ownership, `prisma.files.findFirst({where:{id_icon: folder_id, pc_id}})` (ex `prisma.icons.findFirst`, ver nota de rename en [[Módulo Icon]]) — si no existe, devuelve `null` sin tocar `folder_styles`. Si pasa, `prisma.folder_styles.upsert({where:{folder_id}, update, create})` — ahora sí un `upsert` nativo de Prisma, porque `folder_id` tiene `@unique` en el schema (ver nota de condición de carrera abajo).

> [!success] Condición de carrera cerrada (2026-07-28)
> Antes de esto, `folder_id` no tenía ninguna constraint `@unique` (solo `folder_style_id`, la PK), así que `saveFolderStyle` hacía el patrón manual `findFirst` + `create`/`update` — dos requests concurrentes sobre la misma carpeta nueva podían colarse ambos por la rama `create` y dejar dos filas para el mismo `folder_id`. Se encontró al tocar este módulo para agregarle ownership (JWT) y se corrigió en el mismo pase, mismo arreglo que ya se había aplicado a `kfruit_keybinds` (ver [[Módulo Kfruit]]): `ALTER TABLE folder_styles ADD CONSTRAINT folder_styles_folder_id_unique UNIQUE (folder_id)` (tabla estaba vacía), reintrospección del schema (`folder_id` pasó a `@unique`) y reescritura a `prisma.folder_styles.upsert(...)` atómico. Probado con 10 llamadas concurrentes a la misma carpeta nueva: una sola fila creada. Ver [[Deuda Técnica]].

## Codificación de columnas (decisiones de la app, no del schema)

- **`folder_view`** (Int, FK): el id real de la fila en `folder_views` — `VISTA_A_ID` en `Folder.js` (derivado invirtiendo `ID_A_VISTA`, ver [[Módulo Folder Views]]).
- **`folder_group_by`** (Int, FK): el id real de la fila en `folder_group_by` (no un índice inventado) — se resuelve buscando en `window.folderGroupByOptions` la opción cuya descripción normalizada coincide con `_ordenCriterio`.
- **`folder_group_order`** (VarChar): literalmente `"asc"` / `"desc"` — igual a `_ordenDireccion`. Ver nota de cambio de columna en [[Backend]].

## Consumido por (`Folder.js`)

**Guardar** — `_persistirEstilo()`: arma el estado completo (vista + criterio + dirección) y llama `window.folderStylesServices.saveStyle(...)` vía `advertirSiFalla` (fire-and-forget, mismo patrón que `_assignParent`). Se invoca desde los tres puntos donde cambia el estilo: `_setVista`, `_ordenarPor`, `_ordenarDireccion`. Si `this.id == null` (la carpeta no tiene ícono real en BD — pasa con `DesktopFolder`, el escritorio raíz) no persiste nada.

**Cargar** — `_cargarEstiloGuardado()`, llamado al principio de `_loadContent()` (o sea, solo la primera vez que se abre la carpeta, antes de que `_crearContenido()` arme el header/grid por primera vez — así nace ya con el estilo correcto, sin parpadeo): pide `window.folderStylesServices.getStyle(this.id)`; si hay fila, aplica `folder_view` vía `_aplicarVista(vista)` (la versión de `_setVista` que **no** persiste), resuelve `folder_group_by` buscando el id en `window.folderGroupByOptions` para setear `_ordenCriterio`, y copia `folder_group_order` directo a `_ordenDireccion`. Si no hay fila guardada, no toca nada (quedan los valores por defecto del constructor).

`FolderStylesServices` ([[Frontend Model Services Utils#Services|ver tabla]]) se expone como `window.folderStylesServices`, instanciado en `DesktopManager` igual que `window.iconServices`.

## Borrado en cascada (2026-07-27)

`folder_styles_icons_fk` es `onDelete: NoAction` — Postgres no borra en cascada. Como `folder_styles` no tiene código propio de borrado, el "cascada" se hace a mano en [[Módulo Icon]]: `deleteIcon(id_icon, pc_id)` borra `tx.folder_styles.deleteMany({where:{folder_id}})` dentro de la misma transacción, antes de borrar el `icon`. Mismo patrón ya usado ahí para `txt`.
