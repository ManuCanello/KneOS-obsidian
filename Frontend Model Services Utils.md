---
tags:
  - portfolio/kneos
  - frontend
---

# Frontend Model Services Utils

⬅️ Volver a [[KneOS Portfolio]]

## Model (`public/KneOS/js/model/`)

Datos y registros estáticos.

| Archivo | Exporta | Propósito |
|---|---|---|
| `iconSrc.js` | `iconsSrc`, `getIconMeta(ext)` | Registro central `extensión → {css, Class}`; punto único usado por [[DesktopManager]] para resolver qué clase instanciar por tipo de archivo (`txt`→TxtFile, `fld`/`desktop`→Folder, `ai`→KneAI, `exe`→Doom, `kmd`→Kmd, `kfruit`→KFruit, `maxwell`→Maxwell, `recyclebin`→RecycleBin (2026-07-31), `calc`→Calculator, `contacts`→[[Contacts]] (2026-08-04), `default`→File) |
| `defaultFiles.js` | `defaultFiles` | Íconos por defecto para un escritorio nuevo: Escritorio, Doom, Terminal, KFruit, Kne, Maxwell, Papelera de reciclaje (`recyclebin`, `espacio61`, 2026-07-31), Calculadora (`calc`, `espacio2`) y ahora Contactos (`contacts`, `espacio3`, 2026-08-04) |
| `fileTypes.js` (2026-07-31) | `FileType`, `getFileTypeLabel(fileType)` | Enum de categorías (`GAME`/`PRODUCTIVITY`/`UTILITY`/`OTHER`/`SYSTEM`/`AI` = 1/2/3/4/5/6) equivalente a los ids de la tabla `files_type` (ver [[Módulo Icon]]). Cada app en `apps/` importa `FileType` y pasa su propio valor en su `super(...)` (ver [[File]]) — no hay una función que derive la categoría sola de la extensión. `getFileTypeLabel` devuelve el mismo texto en español que `file_type_desc` en BD (Juego/Productividad/Utilidades/Otros/Sistema/IA), sin pedirlo al backend — lo consume [[Menús Contextuales#`FileProperties`\|FileProperties]] para mostrar la categoría en "Tipo de archivo" |
| `KneAiChat.js` | `class KneAiChat` | Value object: `constructor(id, name, chatHistory, chatContent)` — `chatContent` es el `div` DOM de mensajes de ese chat, creado on-demand si no se pasa uno |
| `KfruitFruit.js` | `class KfruitFruit` | Value object de una fruta: `constructor(name, level, size, points, final, color, src)`, sin métodos |
| `filesUndeletable.js` | `filesUndeletable` | `Set` de extensiones que no se pueden borrar (2026-07-29, `recyclebin` sumada 2026-07-31, `contacts` sumada 2026-08-04) — las de `defaultFiles`, ninguna recreable desde "Nuevo". Consumido por [[Menús Contextuales]] |

## Services (`public/KneOS/js/services/`)

Clientes HTTP hacia el backend, uno por dominio, todos montados sobre un wrapper único.

> [!info] `apiFetch.js` (2026-07-28, simplificado 2026-07-29)
> Reemplaza el `fetch` + boilerplate que antes se repetía a mano en las 29 llamadas del proyecto. `apiFetch(url, {method, body})` arma los headers (`Content-Type`), manda `credentials: "same-origin"` para que el navegador adjunte la cookie de sesión, chequea `response.ok` y parsea el JSON — y **rechaza** en error (a diferencia de los servicios, que nunca rechazan: cada uno mantiene su propio `try/catch` alrededor de `apiFetch` para conservar su sentinel de siempre, `null`/`[]`/`undefined` según el método).
>
> Hasta el 2026-07-29 la sesión viajaba en `localStorage['kneos_token']` + header `Authorization: Bearer`, y este archivo exportaba `getToken`/`setToken`/`clearToken` para manejarla — todo eso se eliminó al pasar la sesión a una cookie `httpOnly` (ver [[Arquitectura#Modelo de sesión y autenticación (pc_id + JWT en cookie httpOnly)]]): el navegador adjunta la cookie solo, ningún JS —ni este archivo, ni la consola— puede leerla o escribirla. Con eso también se fue el manejo especial de `401` (antes limpiaba el token del `localStorage`; ahora no hay nada que un `fetch` del cliente pueda limpiar). `session.js` sigue siendo la única excepción que no pasa por `apiFetch`: ahí un `401` de `/session/verificar` es un resultado esperado, no un error.
>
> El `pc_id` ya no viaja en ningún body ni URL — lo deriva el backend de la cookie (`req.pcId`). Antes casi todos los métodos leían `localStorage.getItem("kneos_pc_id")` inline; esa clave ya no existe (y `startSession()` la borra activamente si la encuentra, resto del sistema pre-JWT — ver [[Módulo Session]]).

| Servicio | Métodos | Backend |
|---|---|---|
| `IconServices` | `newIcon`, `changeDesktopPlace`, `changeParent`, `changeSrc`, `changeName`, `changeLastOpened`, `changeFav`, `changePin`, `getIcons`, `getIconsByParent`, `trashIcon`, `restoreIcon`, `getRecycleBinIcons`, `purgeIcon` (2026-07-31, ex `deleteIcon` — ver [[RecycleBin]]) | [[Módulo Icon]] |
| `KneAiServices` | `newChat(chat_name)`, `newMessage(chat_id, role, message)` (2026-07-30: ahora devuelve `message_id`, antes devolvía `chat_id` — nunca llegaba en la respuesta, siempre era `undefined`), `getUserChats`, `editChatName`, `getChatsMessage`, `deleteChat(chat_id)`, `deleteMessage(message_id)` (2026-07-30) | [[Módulo KneAI]] |
| `Groq` | `ask(context, message, chatHistory)`, `getTitle(firstMessage)` | [[Módulo Groq]] |
| `session` (función `startSession()`, antes `obtenerPcId()`) | — | [[Módulo Session]] |
| `TxtServices` | `saveContent(id_icon, txtcontent)`, `getContent(id_icon)` | [[Módulo Txt]] |
| `KfruitServices` | `getKeybinds`, `updateKeybinds`, `getScores`, `insertScore` | [[Módulo Kfruit]] |
| `FolderGroupByServices` | `getOptions()` | [[Módulo Folder Group By]] |
| `FolderStylesServices` | `saveStyle(folder_id, folder_view, folder_group_by, folder_group_order)` | [[Módulo Folder Styles]] |
| `FolderViewsServices` | `getOptions()` | [[Módulo Folder Views]] |

> [!success] `Groq.js` ya no es la excepción (2026-07-27)
> Antes `ask()`/`getTitle()` no tenían try/catch propio — un fallo de red se propagaba como excepción no controlada. Ahora siguen el mismo patrón que el resto: `try/catch` + chequeo de `response.ok`, devuelven `null` en error. El caller ([[KneAI]]) se ajustó para no romper con ese `null`: si `getTitle` falla no bloquea el envío del mensaje (salta el renombrado nomás), y si `ask` falla no muestra nada — ni mensaje de error ni burbuja "null", queda tal cual para reintentar. Ver [[Deuda Técnica]].

`IconServices.trashIcon`/`restoreIcon`/`purgeIcon` devuelven `data.success` (mismo patrón unificado desde 2026-07-27, ver [[Deuda Técnica]] — antes era `data.succes`, y hasta 2026-07-31 el método se llamaba `deleteIcon`).

## Utils (`public/KneOS/js/utils/`)

- **`avisos.js`** → `advertirSiFalla(promesa, mensaje)`: `promesa?.then((ok) => { if (!ok) console.warn(mensaje); })` — no espera (`await`) la promesa, se ejecuta en paralelo. Usado por [[DesktopManager]], [[File]] y [[Folder]] para llamadas de persistencia "fire and forget".
- **`formato.js`** → `formatearFecha(fecha)`, `formatearTipo(extension)`, `formatearTamano(bytes)`: helpers puros de formateo (sin `Intl`, a diferencia de [[Clock]]) para las columnas fecha/tipo/tamaño de la vista de lista. `formatearRuta(src)` (2026-07-31): `src.replaceAll("_", " ")` — deshace el reemplazo de espacios por `_` que hace `File.nombreParaRuta` al armar `src`, para que "Copiar ruta de acceso" (ver [[Menús Contextuales]], [[Window y Taskbar#`Home`|Home]]) copie el nombre legible en vez de la versión "segura para path". No es 100% reversible (un `_` real en el nombre de origen también se ve como espacio acá) — mismo trade-off que ya acepta el resto del proyecto con `nombreParaRuta`.
- **`kmdPath.js`** (2026-07-31, nuevo) → `tokenize(line)` (separa una línea en tokens respetando comillas dobles) y `resolveEntry(pathStr, cwd)` (resuelve una ruta relativa/absoluta estilo CMD contra el árbol real de archivos, cargando cada carpeta intermedia que falte con `_loadContent()`). Usados solo por [[Kmd]]; mismo truco de carga perezosa que `resolveArchivo.js`, pero recorriendo hacia abajo desde un `cwd` en vez de subir desde una fila plana de BD.
