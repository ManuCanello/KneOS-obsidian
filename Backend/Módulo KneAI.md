---
tags:
  - portfolio/kneos
  - backend
---

# Módulo KneAI

⬅️ Volver a [[Backend]]

CRUD de chats y mensajes del asistente de IA (app [[KneAI]]).

## Endpoints (`routes/kneaiRoutes.js`, ex `kneAI.js` — renombrado 2026-07-29 para seguir la convención `<módulo>Routes.js` del resto de los routers (ver `kfruitRoutes.js`, `txtRoutes.js`); montado en `/kneAI`, `requireAuth` en todo el router desde 2026-07-28)

| Método | Path | Controller |
|---|---|---|
| POST | `/chat` | `newChat` |
| POST | `/message` | `newMessage` |
| PATCH | `/editChatName` | `editChatName` |
| GET | `/chats` | `getUserChats` |
| GET | `/chat/:chat_id/messages` | `getMessages` |
| DELETE | `/chat/:chat_id` | `deleteChat` (2026-07-30) |
| DELETE | `/message/:message_id` | `deleteMessage` (2026-07-30) |

## Controllers (`controllers/kneaiController.js`, ex `kneAI.js` — renombrado 2026-07-29)

- **`newMessage`**: `chat_id, role, message` del body + `req.pcId` del token → valida `chat_id` (id válido), `role` (debe ser `"user"` o `"system"`, matching el enum `role_type` del schema), `message` (string no vacío); llama `saveMessage(...)`. Si el modelo devuelve `null` (el `chat_id` no pertenece a `req.pcId`, ver más abajo), `404 { error: "Chat no encontrado" }`; si no, `{ success: true, message_id }` — **el `message_id` se agregó recién en 2026-07-30**: antes la respuesta no lo incluía y el frontend no tenía forma de identificar un mensaje puntual (necesario para poder borrarlos, ver `deleteMessage` y [[KneAI]]).
- **`newChat`**: `chat_name` del body + `req.pcId` del token → valida `chat_name` como string no vacío; llama `saveChat(...)`, responde `{ success: true, chat_id }`.
- **`getUserChats`**: `getChats(req.pcId)`, devuelve `[{chat_id, chat_name}]`.
- **`getMessages`**: `req.params.chat_id` + `req.pcId` → valida `chat_id` como id válido; `getChatMessages(chat_id, req.pcId)`, hasta 100 mensajes, filtrando por ambos (control de pertenencia).
- **`editChatName`**: `chat_id, chat_name` del body + `req.pcId` del token → valida `chat_id` (id válido) y `chat_name` (string no vacío); `changeChatName(chat_id, req.pcId, chat_name)`. Si el modelo devuelve falsy (chat ajeno), `404 { error: "Chat no encontrado" }`; si no, `{ success: true }`.
- **`deleteChat` (2026-07-30)**: `req.params.chat_id` + `req.pcId` → valida `chat_id`; `deleteChatDb(chat_id, req.pcId)` (importado con alias del modelo — mismo nombre `deleteChat` en ambos archivos). Si el modelo devuelve `null` (chat ajeno o inexistente), `404`; si no, `{ success: true }`.
- **`deleteMessage` (2026-07-30)**: `req.params.message_id` + `req.pcId` → valida `message_id`; `deleteMessageDb(message_id, req.pcId)` (mismo alias que `deleteChat`/`deleteChatDb`). Si el modelo devuelve falsy (mensaje ajeno o inexistente), `404`; si no, `{ success: true }`.

> [!success] Validación de entrada agregada (2026-07-27)
> Igual que en [[Módulo Icon]], usando `utils/validation.js`. Ver [[Deuda Técnica]].

> [!warning] Dos agujeros de ownership cerrados (2026-07-28)
> Al implementar el JWT de sesión se encontró que `changeChatName` y `saveMessage` **no filtraban por `pc_id` en la query** — cualquier `chat_id` ajeno (adivinado o visto en la red) se podía renombrar o recibir mensajes inyectados, aunque el request viniera con un token propio válido. No era un problema de autenticación sino de *ownership* a nivel de query. Corregido en el modelo (ver abajo). Ver [[Deuda Técnica]].

## Modelo (`models/kneaiModel.js`, ex `kneAI.js` — renombrado 2026-07-29)

- **`saveMessage(chat_id, role, message, pc_id)`** *(2026-07-28: valida ownership)*: primero `findFirst` en `kneai_chats` por `{chat_id, pc_id}` — si no existe (el chat no es de este `pc_id`), devuelve `null` sin insertar nada. Si existe, `create` en `kneai_messages`.
- **`saveChat(pc_id, chat_name)`**: `create` en `kneai_chats`, devuelve la fila (incluye `chat_id`).
- **`getChats(pc_id)`**: `findMany` ordenado por `created_at desc`, select `{chat_id, chat_name}`.
- **`getChatMessages(chat_id, pc_id)`**: `findMany` filtrando por ambos campos, orden asc, límite 100, mapeado a `{message_id, role, content, created_at}` — **`message_id` agregado 2026-07-30** (select + mapeo) para que el frontend pueda identificar y borrar mensajes puntuales del historial cargado, no solo los nuevos de la sesión actual.
- **`changeChatName(chat_id, pc_id, chat_name)`** *(2026-07-28: firma cambiada, ahora valida ownership)*: antes `prisma.kneai_chats.update({where:{chat_id}})` — sin filtro de dueño, cualquiera podía renombrar cualquier chat. Ahora `updateMany({where:{chat_id, pc_id}})` y devuelve `count > 0`; se usa `updateMany` en vez de `update` para que un `chat_id` ajeno dé `count === 0` en vez de tirar `P2025` (record not found) y devolver un 500.
- **`deleteChat(chat_id, pc_id)` (2026-07-30)**: transacción (`prisma.$transaction`, mismo patrón que `deleteIconRecursivo` en [[Módulo Icon]]) — primero `findFirst` para validar ownership (si no es del `pc_id`, devuelve `null` sin borrar nada), después `kneai_messages.deleteMany({where:{chat_id}})` y recién entonces `kneai_chats.delete(...)`. El orden importa: la FK `kneai_messages_kneai_chats_fk` es `onDelete: NoAction`, así que borrar el chat antes que sus mensajes tira un error de violación de FK. El `deleteMany` de mensajes filtra solo por `chat_id` (no también por `pc_id`) a propósito — la ownership ya se validó una vez sobre el chat, y filtrar de más ahí dejaría mensajes huérfanos si alguna fila tuviera `pc_id` distinto (el campo es nullable en el schema), lo que haría fallar el delete del chat por la misma FK.
- **`deleteMessage(message_id, pc_id)` (2026-07-30)**: `deleteMany({where:{message_id, pc_id}})`, devuelve `count > 0` — mismo patrón que `changeChatName` (confía en `kneai_messages.pc_id`, que `saveMessage` siempre setea, en vez de validar ownership indirectamente vía el chat). Sin transacción: a diferencia de un chat, un mensaje no tiene nada que cascadear.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|KneAiServices]], usado por [[KneAI]]. La generación del LLM en sí pasa por [[Módulo Groq]].

> [!info] Endpoint `getMessagesContext` eliminado (2026-07-27)
> Revisando la Deuda Técnica, se confirmó el bug (`result[0]` en vez del array de `getChatContext`), pero también que **nadie del frontend llamaba a este endpoint**: `KneAiServices` no tiene ningún método hacia `/chat/:chat_id/context`. El contexto real que recibe el LLM lo arma `KneAI.js` en el cliente (un string fijo de instrucciones + `this.chatHistory` en memoria), pasado directo a `window.groq.ask(...)` — nunca pasaba por el backend. Al ser código muerto, se eliminaron la ruta, el controller `getMessagesContext` y el modelo `getChatContext`, en vez de arreglar el bug.
