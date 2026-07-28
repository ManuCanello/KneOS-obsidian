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
| `iconSrc.js` | `iconsSrc`, `getIconMeta(ext)` | Registro central `extensión → {css, Class}`; punto único usado por [[DesktopManager]] para resolver qué clase instanciar por tipo de archivo (`txt`→TxtFile, `fld`/`desktop`→Folder, `ai`→KneAI, `exe`→Doom, `kmd`→Kmd, `kfruit`→KFruit, `default`→File) |
| `defaultIcons.js` | `defaultIcons` | 5 íconos por defecto para un escritorio nuevo: Escritorio, Doom, Terminal, KFruit, Kne |
| `KneAiChat.js` | `class KneAiChat` | Value object: `constructor(id, name, chatHistory, chatContent)` — `chatContent` es el `div` DOM de mensajes de ese chat, creado on-demand si no se pasa uno |
| `KfruitFruit.js` | `class KfruitFruit` | Value object de una fruta: `constructor(name, level, size, points, final, color, src)`, sin métodos |
| `iconsUndeletable.js` | `iconsUndeletable` | `Set` de extensiones que no se pueden borrar (2026-07-29) — las de `defaultIcons`, ninguna recreable desde "Nuevo". Consumido por [[Menús Contextuales]] |

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
| `IconServices` | `newIcon`, `changeDesktopPlace`, `changeParent`, `changeSrc`, `changeName`, `changeLastOpened`, `changeFav`, `getIcons`, `getIconsByParent`, `deleteIcon` | [[Módulo Icon]] |
| `KneAiServices` | `newChat(chat_name)`, `newMessage(chat_id, role, message)`, `getUserChats`, `editChatName`, `getChatsMessage` | [[Módulo KneAI]] |
| `Groq` | `ask(context, message, chatHistory)`, `getTitle(firstMessage)` | [[Módulo Groq]] |
| `session` (función `startSession()`, antes `obtenerPcId()`) | — | [[Módulo Session]] |
| `TxtServices` | `saveContent(id_icon, txtcontent)`, `getContent(id_icon)` | [[Módulo Txt]] |
| `KfruitServices` | `getKeybinds`, `updateKeybinds`, `getScores`, `insertScore` | [[Módulo Kfruit]] |
| `FolderGroupByServices` | `getOptions()` | [[Módulo Folder Group By]] |
| `FolderStylesServices` | `saveStyle(folder_id, folder_view, folder_group_by, folder_group_order)` | [[Módulo Folder Styles]] |
| `FolderViewsServices` | `getOptions()` | [[Módulo Folder Views]] |

> [!success] `Groq.js` ya no es la excepción (2026-07-27)
> Antes `ask()`/`getTitle()` no tenían try/catch propio — un fallo de red se propagaba como excepción no controlada. Ahora siguen el mismo patrón que el resto: `try/catch` + chequeo de `response.ok`, devuelven `null` en error. El caller ([[KneAI]]) se ajustó para no romper con ese `null`: si `getTitle` falla no bloquea el envío del mensaje (salta el renombrado nomás), y si `ask` falla no muestra nada — ni mensaje de error ni burbuja "null", queda tal cual para reintentar. Ver [[Deuda Técnica]].

`IconServices.deleteIcon` devuelve `data.success` (unificado, 2026-07-27 — antes era `data.succes`, ver [[Deuda Técnica]]).

## Utils (`public/KneOS/js/utils/`)

- **`avisos.js`** → `advertirSiFalla(promesa, mensaje)`: `promesa?.then((ok) => { if (!ok) console.warn(mensaje); })` — no espera (`await`) la promesa, se ejecuta en paralelo. Usado por [[DesktopManager]], [[File]] y [[Folder]] para llamadas de persistencia "fire and forget".
- **`formato.js`** → `formatearFecha(fecha)`, `formatearTipo(extension)`, `formatearTamano(bytes)`: helpers puros de formateo (sin `Intl`, a diferencia de [[Clock]]) para las columnas fecha/tipo/tamaño de la vista de lista.
