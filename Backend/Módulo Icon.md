---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Icon

⬅️ Volver a [[Backend]]

CRUD de íconos del escritorio (archivos y carpetas), incluida su jerarquía (`parent_id`) y cálculo de tamaños agregados. Es el modelo de backend más complejo del proyecto.

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
| GET | `/icons` | `getUserIcons` |
| GET | `/icons/parent/:parent_id` | `getFolderIcons` |
| DELETE | `/icons/delete` | `removeIcon` |

> [!info] `:pc_id` salió de las URLs (2026-07-28)
> Antes eran `/icons/:pc_id` y `/icons/:pc_id/parent/:parent_id`. El `pc_id` ya no lo manda el cliente en ningún lado (ni URL ni body) — lo deriva `requireAuth` del JWT como `req.pcId`. Ver [[Módulo Session]].

## Controllers (`controllers/iconController.js`)

- **`newIcon`**: recibe `name, ext, src, desktop_place, parent_id` (opcional) del body + `req.pcId` del token. Llama `saveIcon(...)`, devuelve `{id_icon}`. En error responde `{ error: "Error al crear el icono" }` (2026-07-27 — antes era un texto copiado de otro controller que concatenaba el objeto `err` completo, exponiendo detalles internos).
- **`editDesktopPlace` / `editParent` / `editName` / `editSrc` / `editLastOpened` / `editFav`**: todas reciben `id_icon` + el campo a modificar del body, y `req.pcId` del token; llaman al modelo `change*` correspondiente (scoping por `{id_icon, pc_id}`), responden `{ success: true }`.
  - `editLastOpened` no recibe valor del body; el modelo setea `last_opened_at: new Date()` server-side.
- **`getUserIcons`**: `getIcons(req.pcId)` — árbol completo de íconos del usuario, plano.
- **`getFolderIcons`**: `getIconsByParent(req.pcId, parent_id)` — hijos directos de una carpeta.
- **`removeIcon`**: `deleteIcon(id_icon, req.pcId)` (transacción, ahora recursiva — ver más abajo).

> [!success] Validación de entrada agregada a todo el módulo (2026-07-27)
> Todos los endpoints ahora validan tipo/formato antes de tocar Prisma, usando el helper compartido `utils/validation.js` (`isNonEmptyString`, `isValidId`, `isBoolean`, `isString`) — devuelven `400` en vez de dejar que un dato mal formado llegue al modelo o genere un 500 confuso. Ver [[Deuda Técnica]].

## Modelo (`models/iconModel.js`)

- **`saveIcon(name, ext, src, desktop_place, pc_id, parent_id)`**: `create`, select solo `id_icon`.
- **`changeDesktopPlace / changeParent / changeName / changeSrc / changeLastOpened / changeFav`**: `prisma.icons.update({where:{id_icon, pc_id}, data:{...}})` — el filtro compuesto PK+`pc_id` actúa como control de propiedad implícito.
- **`ICON_FIELDS`**: objeto `select` reusado al listar íconos.
- **`EXT_CARPETA`**: `Set(["fld", "desktop"])` — extensiones que representan carpetas.
- **`getTamanosAgregados(pc_id)`** *(interna)*: único query SQL crudo del backend — `$queryRaw` con CTE `WITH RECURSIVE` que suma el `size` de todo el subárbol de cada ícono. Devuelve `Map<id_icon, total>`.
  > [!info] Decisión de diseño
  > Se calcula on-the-fly en una sola consulta en vez de mantener un contador acumulado que habría que corregir en cada save/move/delete.
- **`mapIcon(icon, tamanos)`** *(interna)*: si `ext` está en `EXT_CARPETA` usa el tamaño agregado del Map; si no, el tamaño propio del ícono.
- **`getIcons(pc_id)`**: `Promise.all` de íconos + tamaños agregados, combinados con `mapIcon`.
- **`getIconsByParent(pc_id, parent_id)`**: igual, filtrando por `parent_id`.
- **`deleteIcon(id_icon, pc_id)`**: transacción interactiva que delega en `deleteIconRecursivo(tx, id_icon, pc_id)` *(interna)*.
- **`deleteIconRecursivo(tx, id_icon, pc_id)`** *(interna, 2026-07-27)*: busca los hijos directos (`parent_id`, scoping por `pc_id`) y se llama a sí misma sobre cada uno **antes** de tocar el ícono actual (post-order, hijos antes que el padre — necesario porque `icons_icons_fk` es `onDelete: NoAction`); recién entonces borra su `txt`, su `folder_styles` (ver [[Módulo Folder Styles]]) y el propio `icon`.
  > [!success] Borrado en cascada agregado (2026-07-27)
  > Antes, borrar una carpeta con contenido directamente por este modelo (sin pasar por la recursión manual del frontend en `ContextMenuManager._deleteIcon`) fallaba con un 500 por la FK. Ahora el modelo mismo recorre y borra todo el subárbol. En el flujo normal (frontend, que ya borra hijo por hijo antes que el padre) esto no cambia nada — para cuando se llega a borrar un nodo, sus hijos ya fueron borrados uno por uno y esta función no encuentra nada pendiente. Probado con un árbol de 3 niveles (con `txt` y `folder_styles` incluidos) borrado desde la raíz, y también contra el endpoint HTTP real. Ver [[Deuda Técnica]].

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|IconServices]], usado extensamente por [[DesktopManager]], [[Menús Contextuales]] y [[Folder]].
