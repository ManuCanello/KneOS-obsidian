---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Groq

⬅️ Volver a [[Backend]]

Proxy hacia la [API de Groq](https://groq.com/) (modelo `openai/gpt-oss-120b`), para no exponer la API key al cliente. Es el único módulo sin `controllers/`/`models/` dedicados — la lógica vive inline en `routes/groq.js`.

> [!bug] `llama-3.3-70b-versatile` decomisionado por Groq (2026-08-19)
> Groq sacó de circulación `llama-3.3-70b-versatile` (el modelo original de este proyecto) — cualquier request devolvía `404 model_not_found`, y como `/groq/chat` no validaba la respuesta antes de indexar `data.choices[0]` (ver el bug de abajo, ya resuelto), esto explotaba como un `TypeError` sin contexto (`Cannot read properties of undefined (reading '0')`) en vez de mostrar el error real de Groq.
>
> **Reemplazo:** `openai/gpt-oss-120b` — evaluado contra `qwen/qwen3.6-27b` (descartado: mete su bloque `<think>...</think>` completo dentro de `message.content`, mezclado con la respuesta visible — necesitaría parseo extra) y `groq/compound-mini` (descartado: es un sistema agéntico con herramientas propias tipo búsqueda web, no un modelo de chat plano). `gpt-oss-120b` separa `reasoning` de `content` en la respuesta, así que es un reemplazo directo sin parseo.
>
> **Efecto secundario — modelos de razonamiento consumen tokens "pensando" antes de responder:** `/groq/title` usaba `max_tokens: 20`, suficiente para el modelo viejo (no razonaba) pero no para este — el razonamiento solo ya consumía esos 20 tokens y la respuesta llegaba vacía (`finish_reason: "length"`). Fix: `max_tokens` subido a `100` + `reasoning_effort: "low"` agregado (reduce tokens de razonamiento de ~130 a ~40 en pruebas) — verificado con varios mensajes de ejemplo en foco, sin truncar. `/groq/chat` no necesitó ajuste: ya usaba `max_tokens: 1024`, de sobra para razonamiento + respuesta completa (~100-200 tokens en pruebas).
>
> **Efecto secundario — identidad del asistente:** sin ninguna instrucción de identidad en el system prompt, `gpt-oss-120b` respondía "Soy ChatGPT, desarrollado por OpenAI" ante "¿quién sos?" (a diferencia de Llama, que no tenía ese sesgo). El prompt de [[KneAI]] (`KneAI.js`, `_recibeMessage`) solo definía reglas de formato HTML, sin persona — se le agregó una línea explícita ("Sos KneAI, el asistente de IA de KneOS... nunca digas que sos ChatGPT ni que te desarrolló OpenAI"). El de la terminal (`Kmd.js`, `_cmdKneAi`) ya traía identidad propia desde antes y no necesitó cambios; los de [[TxtFile]] (reescritura/inserción de HTML) no son conversacionales, tampoco se tocaron.

## Endpoints (montado en `/groq`, `requireAuth` + `groqLimiter` en todo el router desde 2026-07-28)

| Método | Path | Qué hace |
|---|---|---|
| POST | `/chat` | Consulta al LLM con contexto + historial |
| POST | `/title` | Genera un título corto para un chat nuevo |

> [!warning] Requiere token de sesión (2026-07-28)
> Antes era el único módulo del proyecto totalmente anónimo — cualquiera podía pegarle sin sesión y consumir cuota de la `GROQ_API_KEY` propia. Ahora requiere `requireAuth` (JWT válido) y tiene su propio rate limit (`groqLimiter`, 10 req/min, `keyGenerator: req.pcId` en vez de IP porque la ruta ya está autenticada) — ver [[Backend#middlewares]]. Motivo: es la única ruta del proyecto que le cuesta dinero real al dueño por cada request, más allá del abuso.

## `POST /groq/chat`

- Recibe `{context, message, chatHistory}` (`context` = system prompt, `chatHistory` = array `{role, content}` previos).
- Arma `[system(context), ...chatHistory, user(message)]`, llama a `https://api.groq.com/openai/v1/chat/completions` con `model: openai/gpt-oss-120b`, `temperature: 0.7`, `max_tokens: 1024`, `stream: true` (2026-09-02), `Authorization: Bearer <GROQ_API_KEY>`.
- Si Groq responde `ok` con body, reenvía el stream SSE crudo tal cual (`Readable.fromWeb(groqResponse.body).pipe(res)`, `Content-Type: text/event-stream`) — no arma un JSON `{message}`, el frontend parsea las líneas `data: {...}` él mismo (ver [[Frontend Model Services Utils#Services|Groq.js]]). Si Groq no responde `ok` (falla antes de arrancar el stream: 401, rate limit, etc.), ese error sí llega completo de una y se reenvía como `502 { error }`.

> [!success] Validación defensiva agregada (2026-08-19)
> Antes no chequeaba que `data.choices` existiera antes de indexar `[0]` — un error de Groq (ver el bug de arriba, así se destapó) tiraba un `TypeError` opaco capturado por el catch genérico. Ahora: `if (!response.ok || !data.choices?.[0]?.message?.content) { console.error(...); return res.status(502)... }` antes de leer el contenido — loguea el `status` + body completo de Groq en el server (para diagnosticar la próxima vez que pase algo así) y responde `502` con el mismo mensaje genérico al cliente. Esta validación era sobre la respuesta NO-streaming; con `stream:true` (2026-09-02) el chequeo equivalente es solo `!groqResponse.ok || !groqResponse.body`, ya no hay `data.choices` que indexar acá — el parseo de cada `delta.content` pasó al frontend.

> [!success] Compresión rompía el streaming — corregido (2026-09-02)
> `app.use(compression())` en `server.js` comprime **todas** las respuestas por defecto, `/groq/chat` incluida — gzip/brotli necesitan juntar suficiente buffer antes de flushear, así que el navegador recibía el SSE completo de una sola vez al final en vez de ir mostrando texto (confirmado con un `fetch`+reader crudo: sin el fix, un solo `chunk` de ~190KB llegaba a los ~1.8s; con el fix, decenas de chunks chicos repartidos a lo largo de toda la generación). El primer intento de excluir la ruta con `filter: (req) => req.path === '/groq/chat' ? false : ...` no funcionó: dentro del router montado en `/groq`, Express reescribe `req.url`/`req.path` al path relativo (`/chat`) mientras dura ese middleware/handler, así que el filtro (evaluado en `onHeaders`, ya dentro del handler) nunca matcheaba. Fix: comparar contra `req.originalUrl` en cambio, que se mantiene intacto durante todo el ciclo del request sin importar el mounting de routers.

> [!info] `gpt-oss-120b` separa el streaming de razonamiento del de contenido
> Cada chunk SSE trae `delta.reasoning` (chain-of-thought, `channel: "analysis"`) o `delta.content` (la respuesta visible), nunca ambos a la vez — confirmado capturando el stream crudo con `curl -N`. El parser de `Groq.js` solo mira `delta.content`, así que los tokens de razonamiento se descartan solos sin lógica extra para filtrarlos. Para una pregunta simple, la fase de razonamiento puede tardar más que la respuesta final en sí (varios segundos de `reasoning` antes de que arranque el primer `content`).

## `POST /groq/title`

- Recibe `{firstMessage}`.
- Prompt de sistema fijo (título corto, 3-5 palabras, máx. 40 caracteres), `temperature: 0.2`, `max_tokens: 100` (subido de `20` el 2026-08-19, ver bug arriba), `reasoning_effort: "low"` (mismo día).
- Extrae el título con `?.` y fallback `"Nuevo Chat"` si la respuesta no trae contenido.
- Devuelve `{ title }`.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|Groq.js]] (`window.groq`, instancia global creada en `KNEOS.js`), usado por:
- [[KneAI]] para responder mensajes (`ask`, con historial) y auto-titular chats nuevos (`getTitle`).
- [[TxtFile]] (desde 2026-07-30) para dos botones del editor, ambos con `ask` sin historial (`chatHistory=[]`, cada pedido independiente):
  - Botón general: la respuesta se va escribiendo en el preview del popup a medida que streamea (2026-09-02); al terminar se agregan Aceptar/Volver/Cancelar — no se agrega automáticamente ni queda en una burbuja de chat.
  - Menú contextual "Preguntarle a KneAI" (click derecho, solo aparece con texto seleccionado): la respuesta va reemplazando en vivo el fragmento seleccionado (un placeholder de una sola línea que se llena a medida que streamea, restaurando el texto original si falla) — sin preview ni confirmación; el prompt de sistema es distinto (reescribe un fragmento dado según una instrucción, en vez de responder libremente).
- [[Kmd]] (`_cmdKneAi`) va reescribiendo la misma línea de espera con el texto acumulado a medida que streamea, en vez de esperar la respuesta completa (2026-09-02).

> [!info] `ask(context, message, chatHistory, onChunk)` — streaming agregado (2026-09-02)
> `Groq.js` (frontend) ya no llama a `apiFetch` para `/chat` (ese helper espera un JSON de una sola respuesta) — hace su propio `fetch` + `response.body.getReader()`, decodifica de a chunks (`TextDecoder`, `{stream:true}`), separa por `\n` guardando la última línea (puede venir cortada a la mitad) para completarla en la vuelta siguiente, y por cada línea `data: {...}` completa con `delta.content` no vacío llama a `onChunk(fullAcumuladoHastaAhora)` — el caller solo pisa su `innerHTML`/`textContent` con ese valor en cada llamada, no concatena nada él mismo. `onChunk` es opcional (`getTitle` no lo usa). Devuelve el texto completo final (o `null` si falló) para persistir/loguear como antes.
>
> El try/catch alrededor de todo esto sigue existiendo (ver el punto de abajo, ya resuelto desde 2026-07-27) — un error de red o `!response.ok` cae ahí, loguea y devuelve `null`, igual que la versión no-streaming.
