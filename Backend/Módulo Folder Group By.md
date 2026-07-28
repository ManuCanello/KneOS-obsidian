---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Folder Group By

⬅️ Volver a [[Backend]]

Catálogo de solo lectura de los criterios de "Ordenar por" de [[Folder]] (Nombre/Tipo/Tamaño), para que el frontend los obtenga junto a su id en vez de tenerlos hardcodeados.

## Endpoint (`routes/folderGroupByRoutes.js`, montado en `/folderGroupByRoutes`)

| Método | Path | Controller |
|---|---|---|
| GET | `/` | `getFolderGroupBy` |

## Controller (`controllers/folderGroupByController.js`)

- **`getFolderGroupBy`**: sin parámetros ni scoping por `pc_id` (catálogo global, igual que `kfruit_score`). Llama `getFolderGroupByOptions()`, devuelve el array tal cual.

## Modelo (`models/folderGroupByModel.js`)

- **`getFolderGroupByOptions()`**: `prisma.folder_group_by.findMany({orderBy: {folder_group_by_id: "asc"}})`.

## Datos (tabla `folder_group_by`)

Ya seedeada manualmente en la BD (no vía script de seed, no existe esa infra en el proyecto):

| `folder_group_by_id` | `folder_group_by_desc` |
|---|---|
| 1 | Nombre |
| 2 | Tipo |
| 3 | Tamaño |

## Consumido por

[[Frontend Model Services Utils#Services|FolderGroupByServices.getOptions()]], cargado **una sola vez al arranque** en `KNEOS.js` (`window.folderGroupByOptions`), consumido por [[Folder]] para construir el submenú "Ordenar por" dinámicamente.

> [!info] Relación con `folder_styles`
> Esta tabla es el catálogo de criterios; [[Módulo Folder Styles]] es quien persiste, por carpeta, cuál de estas opciones está eligiendo el usuario (más la vista y la dirección). Un `prisma generate` detectó que la BD ganó la FK `folder_styles_folder_group_by_fk` (`folder_styles.folder_group_by → folder_group_by.folder_group_by_id`), que no estaba en el pull original — confirma que cada fila de `folder_styles` referencia una opción real de esta tabla en vez de guardar un id suelto sin integridad referencial.
