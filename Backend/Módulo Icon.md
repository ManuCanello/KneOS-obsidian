---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Icon

⬅️ Volver a [[Backend]]

CRUD de íconos del escritorio (archivos y carpetas), incluida su jerarquía (`parent_id`) y cálculo de tamaños agregados. Es el modelo de backend más complejo del proyecto.

> [!info] Tabla/modelo Prisma renombrado `icons` → `files` (2026-07-29)
> El usuario renombró la tabla directamente en Postgres. Se corrió `npx prisma db pull` (regenera `schema.prisma` desde la BD) + `npx prisma generate` (regenera el client), y se actualizaron todos los `prisma.icons.*`/`tx.icons.*` a `prisma.files.*`/`tx.files.*` en `models/iconModel.js`, `models/txtModel.js` y `models/folderStylesModel.js` — incluida la consulta SQL cruda de `getTamanosAgregados` (`FROM icons`/`JOIN icons` → `FROM files`/`JOIN files`, ahí no hay Prisma de por medio, es el nombre de tabla literal). **`db pull` no preserva `@updatedAt`** (es una anotación de comportamiento de Prisma, no algo que exista en la BD, así que la introspección no puede reconstruirla) — se reagregó a mano en `updated_at` del modelo `files`, si no `updateAt` dejaba de auto-actualizarse en cada `update()` silenciosamente. Probado end-to-end (create/list/rename/delete) contra la BD real tras el cambio.
>
> **Lo que NO cambió de nombre** (deliberado, para no tocar más superficie de la necesaria): rutas (`/iconRoutes`, `/icon`, `/icons`, etc.), nombres de archivo (`iconController.js`, `iconModel.js`, `IconServices.js`), nombres de función (`newIcon`, `saveIcon`, `getIcons`, etc.) y el nombre de la FK (`icons_icons_fk`, columna `id_icon`). Solo el accessor de Prisma Client (`prisma.icons` → `prisma.files`) y el nombre de tabla en el SQL crudo cambiaron. Los campos de relación en `txt`/`folder_styles` que apuntan a este modelo siguen llamándose `icons` en el schema (Prisma no los renombra automáticamente al reintrospeccionar, solo actualiza su tipo) — `icons: {pc_id}` en un `where` sigue siendo válido y no es un error, es el nombre de campo, no del modelo.

## Endpoints (`routes/iconRoutes.js`, montado en `/iconRoutes`, `requireAuth` en todo el router desde 2026-07-28)

| Método | Path | Controller |
|---|---|---|
| POST | `/icon` | `newIcon` |
| PATCH | `/editDesktopPlace` | `editDesktopPlace` |
| PATCH | `/editName` | `editName` |
| PATCH | `/editSrc` | `editSrc` |
| PATCH | `/editParent` | `editParent` |
| PATCH | `/editLastOpened` | `editLastOpened` |
| PATCH | `/editFav` | `editFav` |
| PATCH | `/editPin` | `editPin` |
| GET | `/icons` | `getUserIcons` |
| GET | `/icons/parent/:parent_id` | `getFolderIcons` |
| PATCH | `/moveToRecycleBin` | `trashIconHandler` |
| PATCH | `/restoreIcon` | `restoreIconHandler` |
| GET | `/icons/recycleBin` | `getRecycleBinIcons` |
| DELETE | `/icons/purge` | `purgeIconHandler` |

> [!success] Papelera de reciclaje / soft delete (2026-07-31)
> `DELETE /icons/delete` (`removeIcon`) se reemplazó por dos endpoints separados: `PATCH /moveToRecycleBin` (soft delete — lo que antes hacía "Eliminar") y `DELETE /icons/purge` (el viejo borrado en cascada real, ahora reservado a vaciar la papelera). Se agregaron `PATCH /restoreIcon` y `GET /icons/recycleBin` para sacar de la papelera y listarla. Ver detalle abajo y [[RecycleBin]].

> [!info] `:pc_id` salió de las URLs (2026-07-28)
> Antes eran `/icons/:pc_id` y `/icons/:pc_id/parent/:parent_id`. El `pc_id` ya no lo manda el cliente en ningún lado (ni URL ni body) — lo deriva `requireAuth` del JWT como `req.pcId`. Ver [[Módulo Session]].

## Controllers (`controllers/iconController.js`)

- **`newIcon`**: recibe `name, ext, src, desktop_place, parent_id, size, file_type` (`parent_id`/`size`/`file_type` opcionales) del body + `req.pcId` del token. Llama `saveIcon(...)`, devuelve `{id_icon}`. En error responde `{ error: "Error al crear el icono" }` (2026-07-27 — antes era un texto copiado de otro controller que concatenaba el objeto `err` completo, exponiendo detalles internos).
  - **`size` agregado (2026-07-29)**: valida con el nuevo helper `isNonNegativeInteger` (`utils/validation.js`) y por defecto `0` si se omite. Antes `newIcon` nunca mandaba `size` al crear un ícono — quedaba siempre en el default de columna (`0`), sin importar qué `File.size` tuviera el objeto en memoria (ver la nota de `size` en [[File]]). Esto hacía que los tamaños "hardcodeados" de Doom/KneAI/KFruit (ver [[Apps]]) se perdieran apenas se recargaba la página, porque `DesktopManager._aplicarMeta` los pisaba con la columna de BD. Ahora `IconServices.newIcon` manda `file.size` en el POST, así que sobrevive al reload para los íconos creados desde que se agregó esto (los ya existentes en BD, creados antes, siguen en `0` hasta que algo los actualice — no hay migración retroactiva).
  - **`file_type` agregado (2026-07-31)**: valida con `isValidId` (mismo helper que `parent_id`), `null` si se omite. Igual patrón de confianza que `size` — el cliente ya sabe la categoría de cada archivo (`File.fileType`, derivado de la extensión vía `model/fileTypes.js`, ver [[File]]), así que el backend no vuelve a calcularla ni la valida contra `ext`, solo persiste lo que le llega. Ver la nota de `files_type`/`file_type` más abajo.
- **`editDesktopPlace` / `editParent` / `editName` / `editSrc` / `editLastOpened` / `editFav` / `editPin`**: todas reciben `id_icon` + el campo a modificar del body, y `req.pcId` del token; llaman al modelo `change*` correspondiente (scoping por `{id_icon, pc_id}`), responden `{ success: true }`.
  - `editLastOpened` no recibe valor del body; el modelo setea `last_opened_at: new Date()` server-side.
  - **`editPin` (2026-07-30)**: mismo patrón exacto que `editFav` (`isValidId(id_icon) && isBoolean(pin)`), columna nueva `pin` en `files` (`Boolean? @default(false)`, agregada junto a `fav`). Ver [[Window y Taskbar#`TaskBarManager` (`core/Taskbarmanager.js`)|TaskBarManager]] para el consumo — anclar un ícono a la taskbar.
- **`getUserIcons`**: `getIcons(req.pcId)` — árbol completo de íconos del usuario, plano.
- **`getFolderIcons`**: `getIconsByParent(req.pcId, parent_id)` — hijos directos de una carpeta.
- **`trashIconHandler`** (2026-07-31): `trashIcon(id_icon, req.pcId)` — soft delete, marca `deleted_at` en todo el subárbol.
- **`restoreIconHandler`** (2026-07-31): `restoreIcon(id_icon, req.pcId)` — inverso, limpia `deleted_at` del subárbol.
- **`getRecycleBinIcons`** (2026-07-31): `getDeletedIcons(req.pcId)` — solo las raíces de cada subárbol en la papelera.
- **`purgeIconHandler`** (2026-07-31, ex `removeIcon`): `purgeIcon(id_icon, req.pcId)` — el borrado en cascada real (transacción recursiva, ver más abajo), reservado a ítems ya en la papelera.

> [!success] Validación de entrada agregada a todo el módulo (2026-07-27)
> Todos los endpoints ahora validan tipo/formato antes de tocar Prisma, usando el helper compartido `utils/validation.js` (`isNonEmptyString`, `isValidId`, `isBoolean`, `isString`, y desde 2026-07-29 `isNonNegativeInteger` para `size`) — devuelven `400` en vez de dejar que un dato mal formado llegue al modelo o genere un 500 confuso. Ver [[Deuda Técnica]].

## Modelo (`models/iconModel.js`)

- **`saveIcon(name, ext, src, desktop_place, pc_id, parent_id, size=0, file_type=null)`**: `create`, select solo `id_icon`. `size` agregado 2026-07-29 (antes no se persistía en absoluto al crear un ícono — ver nota en Controllers arriba). Columna `size` en el schema: `Int? @default(0)`. `file_type` agregado 2026-07-31.
- **`changeDesktopPlace / changeParent / changeName / changeSrc / changeLastOpened / changeFav / changePin`**: `prisma.files.update({where:{id_icon, pc_id}, data:{...}})` (ex `prisma.icons.update`, ver nota de rename arriba) — el filtro compuesto PK+`pc_id` actúa como control de propiedad implícito. `changePin` (2026-07-30) calca `changeFav` campo por campo.
- **`ICON_FIELDS`**: objeto `select` reusado al listar íconos, incluye `pin` desde 2026-07-30 (igual que `fav`) y `file_type` desde 2026-07-31 — así `mapIcon`/`getIcons`/`getIconsByParent` lo devuelven sin cambios extra. **`RECYCLE_BIN_FIELDS`** (2026-07-31): `{...ICON_FIELDS, deleted_at: true}`, solo para `getDeletedIcons` (la papelera necesita mostrar la fecha de borrado, el resto de las consultas no).
- **`EXT_CARPETA`**: `Set(["fld", "desktop"])` — extensiones que representan carpetas.
- **`getTamanosAgregados(pc_id)`** *(interna)*: único query SQL crudo del backend — `$queryRaw` con CTE `WITH RECURSIVE` que suma el `size` de todo el subárbol de cada ícono. Devuelve `Map<id_icon, total>`.
  > [!info] Decisión de diseño
  > Se calcula on-the-fly en una sola consulta en vez de mantener un contador acumulado que habría que corregir en cada save/move/delete.
  > [!bug] Fix de agregado tras la papelera (2026-07-31)
  > Al agregar `deleted_at` (soft delete, ver abajo), esta query sumaba igual el tamaño de archivos ya en la papelera dentro del total de la carpeta que los contenía — el archivo se saca de la vista, pero la fila sigue en `files` con su `parent_id` intacto, así que la suma cruda por `parent_id` seguía contándolo. Se agregó un `JOIN files root ON root.id_icon = d.carpeta_id` + `WHERE root.deleted_at IS NOT NULL OR ic.deleted_at IS NULL`: si la carpeta en sí **no** está en la papelera, excluye los hijos que sí lo estén; si la carpeta en sí **está** en la papelera (un `trashIcon` marca todo el subárbol a la vez, ver abajo), incluye igual todo su contenido — para que el tamaño mostrado en la papelera sea el real del subárbol completo.
- **`mapIcon(icon, tamanos)`** *(interna)*: si `ext` está en `EXT_CARPETA` usa el tamaño agregado del Map; si no, el tamaño propio del ícono. Devuelve también `deleted_at` y `file_type` (2026-07-31, `undefined` si no vino seleccionado — inofensivo).
- **`getIcons(pc_id)`**: `Promise.all` de íconos + tamaños agregados, combinados con `mapIcon`. Filtra `deleted_at: null` (2026-07-31) — un ítem en la papelera no aparece en el escritorio.
- **`getIconsByParent(pc_id, parent_id)`**: igual, filtrando por `parent_id` y `deleted_at: null`.
- **`getDeletedIcons(pc_id)`** (2026-07-31): ítems de la papelera — solo la **raíz** de cada subárbol borrado, no cada descendiente por separado. `trashIcon` marca `deleted_at` en todo el subárbol a la vez, así que un hijo de una carpeta borrada también tiene `deleted_at` seteado; se lo excluye exigiendo `parent_id IS NULL OR` la relación `files` (el padre) tenga `deleted_at: null` — si el padre también está borrado, este ítem es un descendiente, no la raíz que hay que listar. Usa `RECYCLE_BIN_FIELDS`, ordena por `deleted_at desc`.
- **`trashIcon(id_icon, pc_id)`** (2026-07-31): soft delete — `$executeRaw` con `WITH RECURSIVE subtree` que junta el ícono y todo su subárbol (mismo recorrido por `parent_id` que `getTamanosAgregados`) y hace `UPDATE files SET deleted_at = NOW()` sobre esos ids en una sola query. No toca `txt`/`folder_styles`, así que no hace falta transacción ni recorrido en JS como `deleteIconRecursivo`.
- **`restoreIcon(id_icon, pc_id)`** (2026-07-31): inverso exacto de `trashIcon` — mismo CTE, `SET deleted_at = NULL` sobre el subárbol completo.
- **`purgeIcon(id_icon, pc_id)`** (2026-07-31, ex `deleteIcon`): sin cambios de comportamiento, solo de nombre — transacción interactiva que delega en `deleteIconRecursivo(tx, id_icon, pc_id)` *(interna)*. Reservado para "Eliminar definitivamente"/"Vaciar papelera" desde [[RecycleBin]]; el "Eliminar" normal del menú contextual ahora pasa por `trashIcon`.
- **`deleteIconRecursivo(tx, id_icon, pc_id)`** *(interna, 2026-07-27)*: busca los hijos directos (`parent_id`, scoping por `pc_id`) y se llama a sí misma sobre cada uno **antes** de tocar el ícono actual (post-order, hijos antes que el padre — necesario porque `icons_icons_fk` es `onDelete: NoAction`); recién entonces borra su `txt`, su `folder_styles` (ver [[Módulo Folder Styles]]) y el propio `icon`.
  > [!success] Borrado en cascada agregado (2026-07-27)
  > Antes, borrar una carpeta con contenido directamente por este modelo (sin pasar por la recursión manual del frontend en `ContextMenuManager._deleteIcon`) fallaba con un 500 por la FK. Ahora el modelo mismo recorre y borra todo el subárbol. Probado con un árbol de 3 niveles (con `txt` y `folder_styles` incluidos) borrado desde la raíz, y también contra el endpoint HTTP real. Ver [[Deuda Técnica]].
  > [!info] Ya no lo llama el frontend hijo por hijo (2026-07-31)
  > Hasta la papelera, el frontend (`ContextMenuManager._deleteIcon`) recorría y borraba hijo por hijo antes que el padre, así que en ese flujo esta recursión nunca encontraba nada pendiente — cubría solo el caso de pegarle al endpoint directo sobre una carpeta con contenido. Ahora `_deleteIcon` llama a `trashIcon` (soft delete), no a esta función: `purgeIcon`/`deleteIconRecursivo` corren siempre sobre ítems sin ventana/ícono vivo (ya están en la papelera), así que esta es la única que camina el subárbol al purgar.

> [!info] Tabla `files_type` + columna `files.file_type` (2026-07-31)
> Catálogo de categorías de archivo, mismo patrón que `folder_group_by`/`folder_views` (id + desc, ver [[Módulo Folder Group By]]/[[Módulo Folder Views]]): `files_type { file_type_id Int @id @default(autoincrement()), file_type_desc String }`, y `files.file_type Int?` con FK `files_files_type_fk` (`onDelete: NoAction`). Sembrada a mano (no hay endpoint de alta), en dos tandas el mismo día — el orden importa porque el `id` autoincremental tiene que coincidir con el enum `FileType` del frontend (ver [[File]]/`model/fileTypes.js`): primero `1 Juego, 2 Productividad, 3 Utilidades, 4 Otros`, después se agregaron `5 Sistema, 6 IA` (categorías propias para `Folder`/`RecycleBin` y `KneAI` respectivamente, que hasta entonces caían en Utilidades/Productividad) — se **agregaron** como filas nuevas al final en vez de reordenar las 4 ya existentes, para no invalidar los `file_type` que ya tenían las filas de `files` sembradas con esos ids.
>
> **Sin catálogo expuesto al cliente** (a diferencia de `folder_group_by`/`folder_views`, que sí tienen un `GET .../Options` propio): acá el frontend no necesita *listar* las categorías dinámicamente, solo tiene que persistir el id correcto — por eso el enum vive directo en el JS (`fileTypes.js`) en vez de pedirse por HTTP. Si el catálogo cambia (se agrega/renombra una categoría), hay que actualizar la tabla y el enum a mano, en el mismo id.
>
> **Backfill de una sola vez, ya sin efecto práctico:** se corrió `UPDATE files SET file_type = X WHERE ext = 'Y'` directo contra Postgres para poblar el `file_type` de los `files` que ya existían al momento de agregar la columna (y de nuevo tras sumar Sistema/IA, para los `ext` que cambiaron de categoría) — no es un endpoint ni un script versionado. En la práctica la tabla `files` se vació después (reset manual, ver `prisma/reset.sql`), así que hoy no queda ninguna fila vieja que backfillear: todo `files` nuevo entra con `file_type` ya seteado porque lo manda el cliente en `newIcon` (ver Controllers), reflejando el `FileType.X` explícito que cada app pasa en su `super()` (ver [[File]]).
>
> [!bug] `prisma/reset.sql` no está en el repo — setup local desde cero rompe (2026-08-14)
> Armando el entorno en una máquina nueva (Node, Postgres y `.env` recién creados) `npx prisma db push` crea `files_type` vacía — el `README.md` no menciona sembrarla. Resultado: **todo** `POST /icon` fallaba con `P2003` ("Foreign key constraint violated", `field_name: '(not available)'`) apenas KneOS intentaba crear el primer ícono del escritorio, porque el `file_type` que manda el cliente (1-6, ver `fileTypes.js` arriba) no matcheaba ninguna fila real. Se sembró a mano por `psql` (`INSERT INTO files_type (file_type_id, file_type_desc) VALUES (1,'Juego'), (2,'Productividad'), (3,'Utilidades'), (4,'Otros'), (5,'Sistema'), (6,'IA')` + `setval` de la secuencia) para destrabarlo. `prisma/reset.sql`, mencionado arriba como donde se vació `files` alguna vez, no existe en el árbol del repo (ni en `.gitignore` explícitamente) — si existió, no quedó versionado. Pendiente: falta un script de seed versionado (o al menos sumar el paso al README) para que un setup limpio no dependa de saberlo de memoria.

> [!info] Columna `deleted_at` (2026-07-31)
> `files.deleted_at DateTime? @db.Timestamp(6)` — soft delete de la papelera de reciclaje. `NULL` = ítem vivo, con fecha = en la papelera. Se evaluó agregar una tabla `recyclebin(id, file_id, deleted_date)` aparte en vez de esta columna; se descartó porque la papelera necesita seguir jugando con la misma estructura de árbol (`parent_id`, `getTamanosAgregados`, borrado en cascada de `txt`/`folder_styles`) que ya usan `getIcons`/`getIconsByParent`/`Folder` — con una tabla aparte, cada una de esas queries necesitaría un JOIN, y el borrado/restauración recursivo de una carpeta con hijos se complicaría (hijos repartidos entre "vivos" en `files` y "marcados" en la otra tabla). Con la columna, todo sigue siendo el mismo árbol: "listar" es un filtro, "restaurar" es limpiar la columna. Ver [[RecycleBin]].

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|IconServices]], usado extensamente por [[DesktopManager]], [[Menús Contextuales]] y [[Folder]].
