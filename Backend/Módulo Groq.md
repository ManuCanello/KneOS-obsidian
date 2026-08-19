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
- Arma `[system(context), ...chatHistory, user(message)]`, llama a `https://api.groq.com/openai/v1/chat/completions` con `model: openai/gpt-oss-120b`, `temperature: 0.7`, `max_tokens: 1024`, `Authorization: Bearer <GROQ_API_KEY>`.
- Devuelve `{ message: <primera choice> }`.

> [!success] Validación defensiva agregada (2026-08-19)
> Antes no chequeaba que `data.choices` existiera antes de indexar `[0]` — un error de Groq (ver el bug de arriba, así se destapó) tiraba un `TypeError` opaco capturado por el catch genérico. Ahora: `if (!response.ok || !data.choices?.[0]?.message?.content) { console.error(...); return res.status(502)... }` antes de leer el contenido — loguea el `status` + body completo de Groq en el server (para diagnosticar la próxima vez que pase algo así) y responde `502` con el mismo mensaje genérico al cliente.

## `POST /groq/title`

- Recibe `{firstMessage}`.
- Prompt de sistema fijo (título corto, 3-5 palabras, máx. 40 caracteres), `temperature: 0.2`, `max_tokens: 100` (subido de `20` el 2026-08-19, ver bug arriba), `reasoning_effort: "low"` (mismo día).
- Extrae el título con `?.` y fallback `"Nuevo Chat"` si la respuesta no trae contenido.
- Devuelve `{ title }`.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|Groq.js]] (`window.groq`, instancia global creada en `KNEOS.js`), usado por:
- [[KneAI]] para responder mensajes (`ask`, con historial) y auto-titular chats nuevos (`getTitle`).
- [[TxtFile]] (desde 2026-07-30) para dos botones del editor, ambos con `ask` sin historial (`chatHistory=[]`, cada pedido independiente):
  - Botón general: la respuesta se previsualiza en un popup (Aceptar/Volver/Cancelar) antes de insertarse al final de la nota — no se agrega automáticamente ni queda en una burbuja de chat.
  - Menú contextual "Preguntarle a KneAI" (click derecho, solo aparece con texto seleccionado): la respuesta reemplaza directamente el fragmento seleccionado, sin preview ni confirmación — el prompt de sistema es distinto (reescribe un fragmento dado según una instrucción, en vez de responder libremente).

> [!warning] El cliente frontend no maneja errores
> A diferencia de los demás servicios, `Groq.js` (frontend) no tiene try/catch propio — un fallo de red o `!response.ok` se propaga como excepción no controlada hacia `KneAI.js`. Ver [[Deuda Técnica]].
