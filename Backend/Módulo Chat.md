---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Chat

⬅️ Volver a [[Backend]]

Backend de [[KneChat]]: alias, sala global + sala de novedades (solo lectura) + DMs, historial y envío de mensajes, más la capa de tiempo real (`realtime/chatHub.js`) que hace el broadcast. Agregado 2026-08-18 — primer módulo del proyecto con WebSocket. Sala de novedades sumada 2026-08-19.

## Endpoints (`routes/chatRoutes.js`, montado en `/chatRoutes`)

Todo pasa por `requireAuth` (`router.use`), igual que `kneaiRoutes.js`.

| Método | Path | Controller | Notas |
|---|---|---|---|
| GET | `/me` | `getMe` | Bootstrap de la app: `{ pcId, nickname, globalRoomId, newsRoomId, dmRooms, online }` |
| POST | `/nickname` | `setNickname` | `nicknameLimiter`. 3-16 chars, `/^[a-zA-Z0-9_-]+$/`. 409 si ya está tomado |
| GET | `/room/:roomId/messages` | `getRoomMessages` | Paginado hacia atrás por `message_id` (`?before=`), no por timestamp |
| POST | `/room/:roomId/message` | `sendMessage` | `chatMessageLimiter` (20/min por `pc_id`). Exige alias propio y pertenencia a la sala. `403` si la sala es `news` (2026-08-19) — de solo lectura desde el cliente, ver más abajo |
| POST | `/dm` | `openDm` | `{ pcId }` del destinatario → `findOrCreateDmRoom`. Exige alias propio (mismo motivo que `sendMessage`) |
| POST | `/room/:roomId/read` | `markRoomRead` | Marca `last_read_at`; no-op silencioso en la sala global (no tiene filas en `chat_room_members`) |
| DELETE | `/room/:roomId` | `deleteChat` | "Eliminar chat" (2026-08-19) — solo DMs, `400` si es la sala global. Oculta del lado del que la pide, no borra nada, ver más abajo |
| PATCH | `/message/:messageId` | `editMessage` | (2026-08-19) Solo el autor, `chatMessageLimiter`. `404` si el mensaje no es propio o ya está borrado |
| DELETE | `/message/:messageId` | `deleteMessage` | (2026-08-19) Solo el autor, `chatMessageLimiter`. Soft delete, no un DELETE real |

**Pertenencia a la sala** (`isRoomMember`, `models/chatModel.js`): las salas `global` y `news` las puede *leer* cualquier sesión autenticada; una sala `dm` exige que `req.pcId` esté en `chat_room_members`. `getRoomMessages`/`sendMessage` devuelven `403` si no. Verificado con Playwright/curl: una tercera sesión pegándole directo a la API sobre el `room_id` de un DM ajeno recibe `403` en lectura y escritura.

**`news` es de solo lectura para todos, sin excepción** — `isRoomMember` la deja leer igual que `global`, pero `sendMessage` corta antes con un chequeo aparte de `getRoomKind(room_id) === "news"` (no delegado a `isRoomMember`, que solo resuelve pertenencia, no permisos de escritura). No hay ningún endpoint para publicar ahí: los anuncios se insertan directo en `chat_messages` contra el `room_id` de la fila `chat_rooms.kind = 'news'`, no hay UI de admin (2026-08-19, ver [[Deuda Técnica]] si en algún momento se necesita un flujo real de publicación).

## Controllers (`controllers/chatController.js`)

Regla dura, igual que el resto del backend: **nunca se confía en un `pc_id` del body** — siempre sale de `req.pcId`. `sendMessage`/`openDm` exigen que la propia sesión ya tenga alias (`getNickname(req.pcId)`), si no `403 { error: "Falta elegir un alias" }` — sin mensaje diferenciado entre "alias inválido" y "alias tomado" más allá del status code, porque el frontend no muestra ningún error de todos modos (ver [[KneChat#Gate de alias]]). `sendMessage`, tras persistir, llama `broadcastToRoom(room_id, recipients, payload)` de `realtime/chatHub.js` — el POST y el push van en la misma request.

## Modelo (`models/chatModel.js`)

- **`ensureGlobalRoom()`/`getGlobalRoom()`**: hay una única fila `chat_rooms.kind = 'global'`; `ensureGlobalRoom` la busca y la crea si no existe (idempotente, se llama al arrancar `chatHub.js`).
- **`setNickname(pc_id, nickname)`**: `update` directo; atrapa la violación del `UNIQUE` (`P2002`) y devuelve `false` en vez de dejar propagar la excepción — el `UNIQUE` es lo que de verdad cierra la carrera entre chequear y guardar, no una validación previa.
- **`findOrCreateDmRoom(pcIdA, pcIdB)`**: busca una sala `dm` cuyos miembros sean exactamente esos dos (no "alguno de los dos"); si no existe, la crea + inserta las dos filas de `chat_room_members` dentro de `prisma.$transaction`, mismo patrón de atomicidad que `kneaiModel.deleteChat`.
- **`getMessages(room_id, {limit, before})`**: `orderBy: message_id desc` + `take` + `.reverse()` — paginación por cursor de ID, no de timestamp.
- **`getRoomMemberPcIds(room_id)`**: devuelve `null` para la sala global (sin filas de miembros — la ve todo el mundo) y un array para DMs. Ese `null` es una señal explícita que `chatHub.broadcastToRoom` interpreta como "mandale a todos los conectados": se prefirió a inferirlo de un array vacío, que sería ambiguo con una sala `dm` sin miembros por algún dato corrupto.
- **`getUserDmRooms(pc_id)`**: para cada sala `dm` de la sesión, trae la *otra* persona (`pc_id != this`) + su nickname + el último mensaje (con su `pc_id`, para que el frontend sepa si el no-leído es de la otra persona o de uno mismo) — alimenta `dmRooms` de `GET /me`.
- **`getNicknamesByPcIds(pcIds)`**: usada solo por `chatHub.getOnlineUsers()` — traduce los `pc_id` conectados por socket a nicknames en un solo roundtrip, filtrando los que todavía no pasaron el gate de alias.
- **`editMessage(message_id, pc_id, body)`** (2026-08-19): `findFirst` con `{message_id, pc_id, deleted: false}` para validar autoría antes de tocar nada — sin esa fila, devuelve `null` (el controller lo traduce a `404`, sin distinguir "no es tuyo" de "no existe" ni de "ya está borrado", mismo criterio que el resto del backend). Si pasa, `update({body, edited: true})`. `edited` no tiene vuelta atrás, no hay historial de versiones.
- **`deleteMessage(message_id, pc_id)`** (2026-08-19): mismo `findFirst` de autoría que `editMessage`. Soft delete: `update({body: "", deleted: true})` — la fila se conserva en su lugar del historial (mismo `message_id`), pero `body` se vacía server-side para que ni el JSON de la API exponga el contenido borrado a la otra persona. El frontend pinta un placeholder fijo a partir de `deleted`, no de `body`.

### `hidden_at` — "eliminar chat" del lado del cliente (2026-08-19)

`hideDmRoom(room_id, pc_id)`/`unhideDmRoom(room_id, pc_id)`: la sala y sus mensajes no se tocan para nada — solo se marca `chat_room_members.hidden_at = now()` en la fila de quien la "elimina". `getUserDmRooms` filtra: una sala con `hidden_at` seteado se saca de la lista **salvo que exista un mensaje con `created_at > hidden_at`**, en cuyo caso se muestra igual — así una conversación "eliminada" reaparece sola con actividad nueva, sin que nada tenga que ir a limpiar la columna cuando eso pasa. `openDm` llama `unhideDmRoom` siempre, sin condicional: si la sala no estaba oculta, es un `no-op` (`updateMany` con `hidden_at: {not: null}` en el where, cuenta 0 filas).

El controller (`deleteChat`) rechaza con `400` si `getRoomKind(room_id)` es `global` — no tiene sentido "ocultar" la sala única y compartida por todo el mundo.

FKs (`chat_room_members`, `chat_messages` → `chat_rooms`/`sessions`) con `onDelete: NoAction`, misma convención que el resto del schema — no hay ningún flujo de borrado de mensajes/salas todavía, así que no aplica el patrón de "borrar hijos antes que el padre" de [[Módulo KneAI]].

## `realtime/chatHub.js` — WebSocket

No es router/controller/model: es la única pieza de infraestructura de transporte del proyecto, fuera de la cadena clásica (ver [[Backend#`server.js` — entry point]]).

- **Montaje**: `attachChatHub(server)` se llama desde `server.js` sobre el `http.Server` explícito (`http.createServer(app)`, reemplazó a `app.listen()` directo) — se engancha al evento `upgrade` en el path `/ws/chat`; cualquier otro path lo destruye.
- **Auth del handshake**: parsea `req.headers.cookie` (paquete `cookie`, exports `parseCookie`/`stringifyCookie` desde la v2, no `parse`/`serialize` como las versiones 0.x/1.x) y llama a la misma `readSessionToken()` de `middlewares/auth.js` — sin `pcId` válido, `401` + `socket.destroy()`. Chequea además que `Origin` (si viene) coincida con `Host`, cinturón y tiradores sobre `sameSite: 'strict'`.
- **Presencia**: `Map<pc_id, Set<WebSocket>>` — una sesión puede tener varias pestañas abiertas; sale de "online" recién cuando cierra la última. `broadcastPresence()` se dispara en cada connect/close y en cada cambio de alias (`setNickname` del controller la llama).
- **Heartbeat**: `ping` cada 30s, `terminate()` al que no respondió el `pong` anterior — sin esto, cerrar el browser de golpe (sin handshake de cierre limpio) dejaría esa conexión "online" para siempre.
- **API interna**: `broadcastToRoom(roomId, recipients, payload)`, `broadcastPresence()`, `getOnlineUsers()` — consumidas por `chatController.js`, no por el frontend directo. `broadcastToRoom` es genérico respecto al `payload`: `sendMessage` manda `{type: "message", message}` (mensaje nuevo, se agrega); `editMessage`/`deleteMessage` mandan `{type: "message-updated", message}` (2026-08-19 — mismo objeto serializado, pero el cliente lo usa para reemplazar el mensaje existente por `message_id` en vez de agregarlo, ver [[KneChat#`chatSocket.js` (cliente WebSocket)|_onSocketMessageUpdated]]) — no hizo falta tocar `chatHub.js` para esto, ya soportaba cualquier forma de `payload`.

## Dominio de negocio

Primer módulo del proyecto con dos canales (HTTP + WebSocket) para el mismo dominio, y primera vez que `sessions` gana una columna que no es puramente técnica (`nickname`, un alias elegido por la persona). No reabre la decisión de "sin autenticación real" — ver [[Deuda Técnica#Autenticación real — decisión revertida (2026-07-28)]] y la nota agregada ahí mismo el 2026-08-18: el alias es un nombre para mostrar colgado de la sesión anónima existente, no una cuenta.

## Consumido por

App frontend [[KneChat]], vía `ChatServices` (HTTP) y `chatSocket` (WebSocket) — ver [[Frontend Model Services Utils#Services]].
