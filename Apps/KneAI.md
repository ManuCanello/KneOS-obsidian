---
tags:
  - portfolio/kneos
  - apps
---

# KneAI

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/KneAI.js` — extiende [[File]]. Extensión `"ai"`, ícono `sources/icon/kneAi.png`.

> [!abstract] Qué hace
> Chat con IA estilo ChatGPT: historial de conversaciones a la izquierda (colapsable), botón de nuevo chat, input multilínea con auto-resize, Enter envía (Shift+Enter salto de línea), auto-titulado del chat a partir del primer mensaje, persistencia de mensajes e historial por sesión.

## Constructor(name)

`super()`; `_bodyElement=null`, `_chatElement=null`, `chatHistory=[]`, `chatHistoryMaxSize=10`, `_chats=[]` (array de [[Frontend Model Services Utils#Model|KneAiChat]]), `_kneAiServices = new KneAiServices()`, `_activeChatId=null`.

## Funciones

- **`_crearContenido()`**: crea `#iaContainer` (sección izquierda + caja de chat); llama `_sendMessage()` (registra listeners) y `_loadChats()` (carga async).
- **`_createLeftSection()`**: título, botón "N" (nuevo chat), botón "+" (colapsar), `#chatContainer`.
- **`_expandLeftMenu()`**: toggle de colapso.
- **`_createChatBox()`**: `#iaChatBox` con `#messages` y `<textarea>` con auto-resize.
- **`_sendMessage()`**: Enter (sin Shift) envía. Si es el primer mensaje del chat activo, pide título automático (`window.groq.getTitle`) y renombra (`KneAiServices.editChatName`) — **si `getTitle` falla** (2026-07-27, devuelve `null`), no bloquea el envío: se salta el renombrado y sigue con el nombre por defecto del chat. Crea burbuja de usuario, agrega al historial, dispara `_recibeMessage`.
- **`async _recibeMessage(text)`**: prompt de sistema forzando HTML puro, llama `window.groq.ask(context, text, chatHistory)` — **si falla** (2026-07-27, devuelve `null`), no muestra nada (ni error ni burbuja vacía/"null"), se corta ahí. Si tuvo éxito, agrega la respuesta al historial y crea la burbuja de IA.
- **`_createTextBox(type, html)`**: crea el DOM del mensaje, persiste vía `KneAiServices.newMessage`, hace scroll al final.
- **`_addChatHistory(role, content)`**: agrega al historial; si supera `chatHistoryMaxSize` (10), recorta los 2 más viejos.
- **`_buildChatElement(chat_id, chat_name)`**: ítem clickeable de la lista de chats; activa el chat, reemplaza `#messages` visible.
- **`async _createNewChat()`**: crea chat en backend (`newChat(pc_id, "New Chat")`), instancia `KneAiChat`.
- **`async _loadChats()`**: trae chats (`getUserChats`) y sus mensajes en paralelo (`Promise.all` de `getChatsMessage`); si no hay chats, crea uno nuevo; activa el primero.
- **`_scrollDown()`**: scroll al final de `#messages`.

## Persistencia

[[Frontend Model Services Utils#Services|KneAiServices]] → [[Módulo KneAI]]. El LLM en sí pasa por [[Frontend Model Services Utils#Services|Groq.js]] → [[Módulo Groq]].
