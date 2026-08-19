---
tags:
  - portfolio/kneos
  - apps
---

# KneAI

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/KneAI.js` — extiende [[File]]. Extensión `"ai"`, ícono `sources/appIcon/kneAi.png`.

> [!abstract] Qué hace
> Chat con IA estilo ChatGPT: historial de conversaciones a la izquierda (colapsable), botón de nuevo chat, input multilínea con auto-resize, Enter envía (Shift+Enter salto de línea), auto-titulado del chat a partir del primer mensaje, persistencia de mensajes e historial por sesión, y desde 2026-07-30 borrado de chats enteros (click derecho en el menú izquierdo) y de mensajes individuales (click derecho en la burbuja).

## Constructor(name)

`super()`; `_bodyElement=null`, `_chatElement=null`, `chatHistory=[]`, `chatHistoryMaxSize=10`, `_chats=[]` (array de [[Frontend Model Services Utils#Model|KneAiChat]]), `_kneAiServices = new KneAiServices()`, `_activeChatId=null`, `_contextMenu = new ContextMenu()` (2026-07-30, instancia propia — mismo patrón que [[Folder]]). Desde 2026-07-29, `super()` pasa además un `size` inicial de `860_000` bytes (~840 KB), que ahora sí se persiste al crear el ícono — ver la nota de `size` en [[File]]. Desde 2026-07-31 pasa `FileType.AI` (ver [[File]]) — categoría propia, separada de "Productividad" — en el lugar del viejo parámetro `direction`.

## Funciones

- **`_crearContenido()`**: crea `#iaContainer` (sección izquierda + caja de chat); llama `_sendMessage()` (registra listeners) y `_loadChats()` (carga async).
- **`_createLeftSection()`**: título, botón de nuevo chat (ícono `sources/accions/more.svg` vía `aplicarIconoImagen` — 2026-07-30, antes texto plano "N"; mismo ícono que usa [[Folder]] para su ítem "Nuevo"), botón de colapsar (ícono `sources/accions/menu.svg` — 2026-07-30, antes texto plano "+"), `#chatContainer`. Ambos botones de `.iaLeftSectionTitleContainer` tienen `cursor: pointer` explícito (2026-07-30) — un `<button>` no lo trae por default en el UA stylesheet de Chrome.
  > [!info] Se probó y se descartó un hover-invert tipo `.txtIconSave`
  > Se intentó un `:hover` que pintara el botón entero (fondo verde, ícono invertido a negro, mismo patrón que `.txtIcon:hover`), pero pedía volver atrás — la versión final usa `aplicarIconoImagen` directo sobre el botón (sin `::before`, sin hover propio), igual que antes de ese experimento. Queda documentado el motivo técnico por si se retoma: `aplicarIconoImagen` pinta el ícono seteando `background-color: currentColor` **inline** en el elemento — un inline style siempre gana sobre cualquier regla de `:hover` en la hoja de estilos, así que un `background-color` de hover ahí nunca se ve. `.txtIconSave` (ver [[TxtFile]]) evita esto pintando el ícono en un `::before` aparte, dejando el `background-color` del botón real libre para el hover — sería el camino si en algún momento se quiere ese efecto acá.
- **`_expandLeftMenu()`**: toggle de colapso.
- **`_createChatBox()`**: `#iaChatBox` con `#messages` y `<textarea>` con auto-resize (JS setea `style.height` según `scrollHeight`, tope en el `max-height:200px` de `.iaInput`).
  > [!bug] Desbordaba su contenedor (arreglado 2026-07-30)
  > `.iaInputContainer` tenía `height: 70px` fijo mientras `.iaInput` era `position: absolute` y podía crecer hasta 200px — al escribir varias líneas, el textarea se salía del contenedor fijo y se superponía visualmente al área de mensajes de arriba. Arreglado sacando el `position: absolute` (pasa a flujo normal) y cambiando el contenedor a `min-height: 70px` sin tope propio — al ser flex-column con `.iaChatBoxMessages{flex:1}` como hermano, el contenedor de input crece con su contenido (acotado solo por el `max-height` del textarea) y los mensajes se encogen para darle lugar, sin overlap.
- **`_sendMessage()`**: Enter (sin Shift) envía, delegando en `_handleSend(input)`.
- **`async _handleSend(input)` (2026-08-19, extraído de `_sendMessage`)**: mismo camino de envío disparado tanto por Enter como por el botón `.iaSendButton` (ver abajo) — antes vivía inline dentro del listener de `keydown`, sin forma de reusarlo desde otro trigger. Si es el primer mensaje del chat activo, pide título automático (`window.groq.getTitle`) y renombra (`KneAiServices.editChatName`) — **si `getTitle` falla** (2026-07-27, devuelve `null`), no bloquea el envío: se salta el renombrado y sigue con el nombre por defecto del chat. Crea burbuja de usuario (`await _createTextBox`, 2026-07-30: antes no se esperaba, ver más abajo), agrega al historial con su `message_id`, dispara `_recibeMessage`.
  > [!success] Botón "Enviar" (2026-08-19)
  > `.iaSendButton` junto al `<textarea>`, mismo patrón que el `.chatSendButton` de [[KneChat]] — pensado para touch/mobile, donde Enter en un `<textarea>` no siempre está a mano. Requirió envolver el textarea+botón en un nuevo `.iaInputRow` (antes el `<textarea>` sola ocupaba `width:80%` centrada en `.iaInputContainer`; ahora esa fila de 80% es la que reparte `flex:1` para el input y ancho por contenido para el botón, ver `kneai.css`).
- **`async _recibeMessage(text)`**: prompt de sistema forzando HTML puro. Desde 2026-07-30 arma `chatHistory` limpio de `message_id` (`this.chatHistory.map(({role, content}) => ({role, content}))`) antes de mandarlo — ver la nota de `message_id` en `_addChatHistory` — y llama `window.groq.ask(context, text, chatHistory)`. **Si falla** (2026-07-27, devuelve `null`), no muestra nada (ni error ni burbuja vacía/"null"), se corta ahí. Si tuvo éxito, crea la burbuja de IA y agrega la respuesta al historial con su `message_id`.
- **`async _createTextBox(type, html)`** *(2026-07-30: ahora async, devuelve el `message_id`)*: arma el DOM vía `_buildMessageElement`, lo agrega al chat, `await`-ea `KneAiServices.newMessage` (antes fire-and-forget) para tener el `message_id` real antes de cablear el menú contextual (`_wireMessage`), hace scroll al final y devuelve el id — el caller lo necesita para `_addChatHistory`.
- **`_buildMessageElement(type, html)` (2026-07-30)**: solo arma el DOM de una burbuja (sin persistir ni cablear nada) — extraído de `_createTextBox` para reusarlo también en `_loadChats`, que antes duplicaba la misma construcción a mano.
- **`_wireMessage(message, messageText, message_id)` (2026-07-30)**: si hay `message_id`, lo guarda en `message.dataset.messageId` y cuelga `contextmenu` en `messageText` (la burbuja visible, no la fila full-width) → `_openMessageContextMenu`. No hace nada si `message_id` es `null` (falló la persistencia — no habría qué borrar en el servidor).
- **`_openMessageContextMenu(e, message_id, messageElement)` (2026-07-30)**: abre un `ContextMenu` (`MESSAGE_CONTEXT_MENU_ID`) con "Eliminar mensaje" — mismo patrón que el menú de chats.
- **`async _deleteMessage(message_id, messageElement)` (2026-07-30)**: `KneAiServices.deleteMessage(message_id)`; si tuvo éxito, lo saca de `activeChat.chatHistory` (por `message_id`, no por índice — `chatHistory` es una ventana acotada a `chatHistoryMaxSize`, así que un mensaje viejo puede no estar ahí) y de la fila del DOM (`messageElement`, el wrapper full-width, no solo la burbuja).
- **`_addChatHistory(role, content, message_id)`** *(2026-07-30: ahora recibe `message_id`)*: agrega `{role, content, message_id}` al historial; si supera `chatHistoryMaxSize` (10), recorta los 2 más viejos. El `message_id` viaja en el objeto en memoria para poder ubicar y borrar una entrada puntual (`_deleteMessage`), pero se saca antes de mandar el array a Groq (ver `_recibeMessage`) — la API solo espera `{role, content}`.
- **`_buildChatElement(chat_id, chat_name)`**: ítem clickeable de la lista de chats; activa el chat, reemplaza `#messages` visible. Desde 2026-07-30 también escucha `contextmenu` → `_openChatContextMenu`.
- **`_openChatContextMenu(e, chat_id, chatElement)` (2026-07-30)**: abre un `ContextMenu` (`CHAT_CONTEXT_MENU_ID`) con una única opción, "Eliminar chat" — mismo patrón que `Folder._abrirMenuQuitarFavorito` (ver [[Folder]]).
- **`async _deleteChat(chat_id, chatElement)` (2026-07-30)**: llama `KneAiServices.deleteChat(chat_id)` (el borrado real, incluidos los mensajes, pasa por el modelo — ver [[Módulo KneAI]]); si tuvo éxito, saca el chat de `_chats` y del DOM. Si era el chat activo, activa el primero que haya quedado o, si no queda ninguno, crea uno nuevo (`_createNewChat`) — mismo criterio que `_loadChats` cuando el usuario no tiene chats.
- **`async _createNewChat()`**: crea chat en backend (`newChat(pc_id, "New Chat")`), instancia `KneAiChat`.
- **`async _loadChats()`**: trae chats (`getUserChats`) y sus mensajes en paralelo (`Promise.all` de `getChatsMessage`); si no hay chats, crea uno nuevo; activa el primero. Desde 2026-07-30 usa `_buildMessageElement`/`_wireMessage` para las burbujas del historial (cada mensaje ya trae su `message_id` del servidor, ver [[Módulo KneAI]]), en vez de reconstruir el DOM a mano.
- **`_scrollDown()`**: scroll al final de `#messages`.

## Persistencia

[[Frontend Model Services Utils#Services|KneAiServices]] → [[Módulo KneAI]]. El LLM en sí pasa por [[Frontend Model Services Utils#Services|Groq.js]] → [[Módulo Groq]].
