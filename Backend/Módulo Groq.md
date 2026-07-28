---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Groq

⬅️ Volver a [[Backend]]

Proxy hacia la [API de Groq](https://groq.com/) (modelo `llama-3.3-70b-versatile`), para no exponer la API key al cliente. Es el único módulo sin `controllers/`/`models/` dedicados — la lógica vive inline en `routes/groq.js`.

## Endpoints (montado en `/groq`, `requireAuth` + `groqLimiter` en todo el router desde 2026-07-28)

| Método | Path | Qué hace |
|---|---|---|
| POST | `/chat` | Consulta al LLM con contexto + historial |
| POST | `/title` | Genera un título corto para un chat nuevo |

> [!warning] Requiere token de sesión (2026-07-28)
> Antes era el único módulo del proyecto totalmente anónimo — cualquiera podía pegarle sin sesión y consumir cuota de la `GROQ_API_KEY` propia. Ahora requiere `requireAuth` (JWT válido) y tiene su propio rate limit (`groqLimiter`, 10 req/min, `keyGenerator: req.pcId` en vez de IP porque la ruta ya está autenticada) — ver [[Backend#middlewares]]. Motivo: es la única ruta del proyecto que le cuesta dinero real al dueño por cada request, más allá del abuso.

## `POST /groq/chat`

- Recibe `{context, message, chatHistory}` (`context` = system prompt, `chatHistory` = array `{role, content}` previos).
- Arma `[system(context), ...chatHistory, user(message)]`, llama a `https://api.groq.com/openai/v1/chat/completions` con `model: llama-3.3-70b-versatile`, `temperature: 0.7`, `max_tokens: 1024`, `Authorization: Bearer <GROQ_API_KEY>`.
- Devuelve `{ message: <primera choice> }`.

> [!bug] Sin validación defensiva
> No valida que `data.choices` exista antes de indexar `[0]`. Si Groq devuelve un error (rate limit, API key inválida), `data.choices` puede ser `undefined` y lanzar una excepción — capturada por el catch genérico, pero el mensaje de error al cliente es poco informativo. Contrasta con `/groq/title`, que sí usa optional chaining. Ver [[Deuda Técnica]].

## `POST /groq/title`

- Recibe `{firstMessage}`.
- Prompt de sistema fijo (título corto, 3-5 palabras, máx. 40 caracteres), `temperature: 0.2`, `max_tokens: 20`.
- Extrae el título con `?.` y fallback `"Nuevo Chat"` si la respuesta no trae contenido.
- Devuelve `{ title }`.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|Groq.js]] (`window.groq`, instancia global creada en `KNEOS.js`), usado por [[KneAI]] para responder mensajes (`ask`) y auto-titular chats nuevos (`getTitle`).

> [!warning] El cliente frontend no maneja errores
> A diferencia de los demás servicios, `Groq.js` (frontend) no tiene try/catch propio — un fallo de red o `!response.ok` se propaga como excepción no controlada hacia `KneAI.js`. Ver [[Deuda Técnica]].
