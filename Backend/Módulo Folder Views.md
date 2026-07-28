---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Folder Views

⬅️ Volver a [[Backend]]

Catálogo de solo lectura de las opciones de "Ver" de [[Folder]] (Íconos grandes/Íconos pequeños/Lista), análogo a [[Módulo Folder Group By]] pero para la vista en vez del criterio de orden.

## Endpoint (`routes/folderViewsRoutes.js`, montado en `/folderViewsRoutes`)

| Método | Path | Controller |
|---|---|---|
| GET | `/` | `getFolderViews` |

## Controller (`controllers/folderViewsController.js`)

- **`getFolderViews`**: sin parámetros ni scoping por `pc_id` (catálogo global). Llama `getFolderViewsOptions()`, devuelve el array tal cual.

## Modelo (`models/folderViewsModel.js`)

- **`getFolderViewsOptions()`**: `prisma.folder_views.findMany({orderBy: {folder_view_id: "asc"}})`.

## Datos (tabla `folder_views`)

Ya seedeada manualmente en la BD:

| `folder_view_id` | `folder_view_desc` | `icon_path` |
|---|---|---|
| 1 | Íconos grandes | `sources/icon/file.svg` |
| 2 | Íconos pequeños | `sources/icon/file.svg` |
| 3 | Lista | `sources/icon/list.svg` |

> [!info] `icon_path` (2026-07-28)
> Columna nueva, `String?` en `prisma/schema.prisma`, sin lógica propia en modelo/controller — viaja tal cual porque `getFolderViewsOptions()` no usa `select`. Guarda solo la *ruta* del ícono; el tamaño con el que se renderiza en el menú lo decide `Folder.js` en el frontend según la vista (`TAMANO_ICONO_VISTA = {grande: 56, pequeno: 28}`), no la BD — por eso "Íconos grandes" e "Íconos pequeños" comparten el mismo `icon_path` (`file.svg`) pero se ven distinto.

## Consumido por

[[Frontend Model Services Utils#Services|FolderViewsServices.getOptions()]], cargado **una sola vez al arranque** en `KNEOS.js` (`window.folderViewsOptions`), consumido por [[Folder]] para construir el submenú "Ver" dinámicamente (ícono incluido, vía `icon_path`).

> [!info] Mapeo id ↔ clave interna, no por texto
> A diferencia de "Ordenar por" (que normaliza la descripción de texto — "Tamaño" → "tamano" — para llegar a la clave interna), acá el mapeo es por **id**: `ID_A_VISTA = {1: "grande", 2: "pequeno", 3: "lista"}` en `Folder.js`, porque las descripciones ("Íconos grandes") no se pueden normalizar mecánicamente a las claves internas de `_vistaActual` ("grande"). `VISTA_A_ID` (usado por [[Módulo Folder Styles]] para persistir) se deriva invirtiendo ese mismo mapeo, así hay una sola fuente de verdad.
