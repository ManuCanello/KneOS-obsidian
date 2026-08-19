---
tags:
  - portfolio/kneos
  - apps
---

# KneChat

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/KneChat.js` (clase `KneChat`) — extiende [[File]]. Extensión `"chat"`, ícono propio `sources/appIcon/knechat.svg` (burbuja de diálogo), `FileType.UTILITY`. Agregado 2026-08-18. No es un port de Java (a diferencia de BlackJack/Hangman/FlipCoin/Kdle/CarRace/Tetris) — es la primera pieza de tiempo real de todo el proyecto.

> [!abstract] Qué hace
> Chat en vivo entre las sesiones que estén con la app abierta en ese momento: una sala pública `#GENERAL` compartida por todo el mundo, una sala `#NOVEDADES` de solo lectura para anuncios (2026-08-19), más DMs 1-a-1 que se abren clickeando un nombre en la lista de conectados. Historial persistido en Postgres. No hay cuentas de usuario — al entrar se elige un alias (3-16 caracteres, único) que queda colgado de la sesión anónima existente (`sessions.nickname`) y se reutiliza en visitas siguientes. Ver [[Módulo Chat]] para el backend y el protocolo WebSocket.

## Identidad: "online" = ventana abierta, no "KneOS abierto"

Suposición de diseño explícita: una sesión cuenta como conectada mientras tiene la ventana de KneChat abierta, no mientras tiene KneOS abierto en general. El socket se conecta al entrar a la app (`_enterChat()`) y se cierra al cerrar la ventana (`onClose` de `Window`, ver más abajo) — nadie aparece "online" sin poder recibir en ese momento. Si se prefiriera presencia a nivel de todo KneOS, la conexión se movería a `KNEOS.js`; es un cambio chico, pero cambia el significado de la lista de conectados.

## Arquitectura: REST para escribir/leer, WebSocket solo para push

El socket transporta únicamente eventos salientes (`message`, `presence`). Mandar un mensaje sigue siendo un `POST /chatRoutes/room/:roomId/message` normal — el broadcast vuelve por el socket, incluido al propio emisor, así que no hay renderizado optimista: la burbuja se pinta recién cuando llega el push. Esto evita reimplementar auth/validación/rate-limit dentro del handler del socket; se reusan tal cual `requireAuth`, `utils/validation.js` y `express-rate-limit`, igual que cualquier otro endpoint del proyecto. Ver [[Módulo Chat]].

## Gate de alias

Antes de ver nada, `_buildGate()` pide un nombre (`#chatNicknameInput`, `.game-input`/`.game-boton` reusados de `apps/game.css`). Si el alias es inválido o ya está tomado, el backend responde 400/409 y **no se muestra ningún indicador de error** — el input queda igual, listo para reintentar (regla del proyecto, ver [[Reglas]]). Confirmado el alias, `_enterChat()` oculta el gate, carga el historial de `#GENERAL`, renderiza pestañas/mensajes/lista de conectados y recién ahí conecta el `ChatSocket`.

## Salas y pestañas

`this._rooms` es un `Map<room_id, {kind, label, otherPcId, messages, unread}>`. La sala global y la de novedades se conocen desde el arranque (`getMe().globalRoomId`/`newsRoomId`, se registran como pestañas fijas en ese orden antes de recorrer `dmRooms`); los DMs existentes se traen en el mismo `getMe()` (`dmRooms`). Un DM nuevo puede aparecer de dos formas:
- **Lo abre uno mismo**: click en un nombre de `#chatOnline` → `POST /chatRoutes/dm` (`findOrCreateDmRoom`) → `_switchRoom`.
- **Lo abre la otra persona primero**: el mensaje que llega por el socket ya trae `nickname`/`pc_id` del remitente — alcanza para construir la pestaña ahí mismo (`_onSocketMessage`, rama "sala que no conocíamos") sin pedir nada más al servidor. No hace falta un endpoint de "traer info de una sala".

Un punto verde (`.chatTabUnread`) marca una pestaña con mensajes sin leer; se limpia al cambiar a esa pestaña (`_switchRoom` llama `markRead`, que solo importa para DMs — en la sala global y en novedades es un no-op silencioso, ninguna de las dos tiene filas en `chat_room_members`).

## `#NOVEDADES`: sala de solo lectura (2026-08-19)

Mismo tratamiento que `#GENERAL` como pestaña fija (ni se puede "eliminar" ni lleva menú contextual), pero de solo lectura desde el cliente: nadie escribe ahí vía la app, el contenido se carga directo en la base (ver [[Módulo Chat#`ensureNewsRoom()`]]). `_updateInputState()` deshabilita el `<textarea>` y el botón de emojis al entrar a esa sala (`placeholder` cambia a "Solo lectura", estilo atenuado vía `:disabled` en `knechat.css`) en vez de ocultar el input — la idea es que quede claro *por qué* no se puede escribir, no que la caja desaparezca sin explicación. `_sendCurrentInput` y el menú contextual de mensajes (editar/eliminar) tienen el mismo guard por las dudas, aunque en la práctica nunca hay un mensaje propio ahí para editar.

## Editar, eliminar mensajes y eliminar chat (2026-08-19)

Solo sobre lo propio: un mensaje ajeno o ya borrado nunca lleva menú contextual (`_buildMessageBubble` chequea `mine && !message.deleted && !editing` antes de agregar el listener de `contextmenu`, ver [[ContextMenu#addItem]]).

- **Editar** (`_startEdit`/`_buildEditInput`/`_confirmEdit`): reemplaza el `.chatMessageBody` por un `<textarea>` in-place con el texto actual seleccionado. Enter confirma, Escape o perder el foco cancela — **sin `prompt()`**, regla del proyecto. `_cancelEdit()` tiene un guard (`if (this._editingMessageId == null) return`) porque `_confirmEdit` saca el `<textarea>` del DOM al re-renderizar, lo que dispara su propio evento `blur` — sin el guard, ese blur reentra en `_cancelEdit()` en medio del render y duplica los mensajes en pantalla (encontrado y corregido durante el testeo con Playwright). El texto final no se pinta de forma optimista: llega por el evento de socket `message-updated`, igual que un mensaje nuevo llega por `message`.
- **Eliminar mensaje** (`_deleteMessage`): sin confirmación extra, mismo criterio que [[KneAI]] con sus propios mensajes — un clic en "Eliminar mensaje" borra directo. No es un DELETE real: el backend hace soft-delete (`deleted: true`, `body` vaciado) y el mensaje se sigue viendo en su lugar del historial con el placeholder **"Se eliminó este mensaje"** (itálica, opacidad reducida — `.chatMessageBodyDeleted`), estilo WhatsApp. Un mensaje editado muestra **"(editado)"** junto al timestamp en el header (`message.edited`).
- **Eliminar chat** (solo DMs — la sala global no lleva esta opción en absoluto, no tiene sentido "eliminarla" al ser única y compartida): clic derecho en la pestaña → "Eliminar chat" → confirmación real con `AlertWindow` (mismo patrón que `RecycleBin.confirmarVaciarPapelera`, **nunca `window.confirm()`**) → `DELETE /chatRoutes/room/:roomId`. Es un ocultamiento del lado de quien lo pide, no un borrado real: la otra persona conserva la sala y todo su historial intacto. Si llega un mensaje nuevo después, la conversación reaparece sola en la próxima carga de `dmRooms` — y si el socket sigue conectado, el mismo mecanismo de "sala que no conocíamos" de `_onSocketMessage` la trae de vuelta en vivo, sin distinguir "nunca existió" de "estaba oculta". Ver [[Módulo Chat#`hidden_at` — "eliminar chat" del lado del cliente]].

## Emoticones ASCII/kaomoji (2026-08-19)

Botón `.chatEmojiButton` (`^_^`) junto al `<textarea>` abre un [[Menús Contextuales|ContextMenu]] con la lista fija de `model/asciiEmojis.js` (`:)`/`:D`/kaomoji tipo `¯\_(ツ)_/¯`, `(╯°□°)╯︵ ┻━┻`) — sin ninguna librería: es texto estático curado a mano, no glifos Unicode (los paquetes npm de "emoji" trabajan con esos, no con ASCII/kaomoji), así que no había nada real que instalar.

- **Inserción en la posición del cursor**: el click en el botón le roba el foco al `<textarea>` antes de que el menú termine de abrir, así que `_openEmojiPicker` captura `selectionStart`/`selectionEnd` en ese mismo momento (siguen siendo legibles después del blur) — clickear un emoticono lo inserta ahí (`_insertEmoji`), no al final del texto, y devuelve el foco con el cursor justo después de lo insertado.
- **Scroll propio del menú**: con 18 ítems, este `ContextMenu` es más alto que cualquier otro del sistema — y `ContextMenu._reancharSiNoEntra` solo reancla por overflow de abajo/derecha, nunca de arriba. Anclado `"bottom"` contra un botón que vive cerca del piso de la ventana, el menú se salía por arriba de la pantalla. Fix acotado al propio menú (`#chatEmojiContextMenu { max-height: 60vh; overflow-y: auto; }` en `knechat.css`), sin tocar el componente `ContextMenu` compartido — el resto de los menús del sistema (pocos ítems) no lo necesita.

## Seguridad: XSS

**Todo el contenido de otra sesión se renderiza con `textContent`, nunca `innerHTML`** (`_buildMessageBubble`) — a diferencia de [[KneAI]], que sí usa `innerHTML` porque ahí el HTML lo genera el modelo bajo un prompt controlado. Acá el texto lo escribe otra persona; meterlo con `innerHTML` sería XSS persistente guardado en la base. Verificado con Playwright: mandar `<img src=x onerror=alert(1)>` como mensaje se ve como texto plano literal del otro lado, nunca se ejecuta.

## `chatSocket.js` (cliente WebSocket)

`public/KneOS/js/services/chatSocket.js` — no sigue el patrón de los demás `services/*.js` (no es HTTP). `connect()` abre `ws(s)://<host>/ws/chat` (la cookie de sesión viaja sola, same-origin); `on(type, handler)` suscribe por tipo de evento (`message`/`presence`/`open`/`close`); reconecta solo con backoff exponencial (1s→30s tope) si la desconexión no fue pedida por `close()`. Al cerrar la ventana (`onClose` agregado a `Window` — ver [[Window y Taskbar]] — porque el botón X de la barra de título llama `Window.cerrar()` directo, no pasa por `File.cerrarVentana()`), el socket se corta; reabrir la app reconecta y recarga todo desde cero (`_crearContenido()` vuelve a correr `_init()`).

## Estilos (`styles/apps/knechat.css`)

Monocromo verde estricto, como el resto del core/apps de KneOS — ningún color fuera de `--primary-color`/`--primary-dim`, acentos con `rgba(0,255,65,.1)` u opacidad (ni siquiera para "desconectado"). Reusa `.game-input`/`.game-boton` de `apps/game.css` para el gate. Tipografía con `clamp(..., cqw, ...)`, atada al `container-type: inline-size` que ya define `.app` (no se redeclara).

## Persistencia

`chat_rooms`/`chat_room_members`/`chat_messages` + columna `nickname` en `sessions`. `chat_rooms.kind` es un enum con tres valores: `global`, `dm` y `news` (2026-08-19) — ver [[Módulo Chat]].

## Consumido por

Servicios frontend `ChatServices` (HTTP) y `chatSocket` (WebSocket), ver [[Frontend Model Services Utils#Services]].
