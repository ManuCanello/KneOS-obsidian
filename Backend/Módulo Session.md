---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Session

⬅️ Volver a [[Backend]]

Emisión y renovación del JWT de sesión que autentica al resto del backend. Ver [[Arquitectura#Modelo de sesión y autenticación (pc_id + JWT)]] para el flujo completo desde el frontend, y [[Deuda Técnica]] para el porqué del cambio (2026-07-28, reemplaza la decisión previa de no implementar auth real).

## Endpoints (`routes/session.js`)

| Método | Path | Controller | Auth |
|---|---|---|---|
| POST | `/session/nueva` | `nuevaSesion` | público, `sessionLimiter` (20/hora por IP — es la única escritura pública del proyecto, sin techo cualquiera podría inflar `sessions`) |
| POST | `/session/verificar` | `verificarSesion` | público (lee la cookie, no requiere `requireAuth` porque es el propio endpoint de renovación) |

## Controllers (`controllers/session.js`)

- **`nuevaSesion(req, res)`**: no lee body. Genera `pcId` con `uuidv4()`, llama `guardarSesion(pcId)`, firma el token y lo pone en la cookie vía `setSessionCookie` (ver abajo), responde `{ success: true }` — el body nunca contiene el token, no hay nada que el cliente necesite leer. Catch → `500 { error: 'Error al crear sesión' }`.
- **`verificarSesion(req, res)`**: lee el token de `req.cookies` (`readSessionToken`, ver [[Backend#middlewares]]). Si el token no es válido **o** `existeSesion(pcId)` da falso, responde `200 { valid: false }` *(hasta 2026-08-13 era `401 { error: "Sesión inválida" }` — cambiado porque este endpoint es un chequeo, no un recurso protegido: cada primera visita sin cookie disparaba un 401 real de red que Lighthouse marcaba como error de consola aunque el cliente ya lo manejara con `.ok`; el cliente solo necesitaba un booleano, no un status code de error)*. Si pasa ambos, renueva la cookie (mismo `pcId`, vencimiento nuevo) y responde `200 { valid: true }`.
  > [!info] Por qué revalida contra la base y no solo la firma
  > El JWT es autocontenido — una firma válida no garantiza que la fila siga en `sessions`. Sin este chequeo, un token firmado para un `pc_id` borrado de la base quedaría "válido" para siempre pero cada escritura fallaría por FK (`kneai_chats`, `kneai_messages`, `kfruit_keybinds` referencian `sessions.pc_id`). Chequear la base acá es lo que permite que la sesión se autorepare (mismo comportamiento que el viejo `existe:false`, pero ahora también revalida la firma).

## Middleware de auth (`middlewares/auth.js`)

- **`signSessionToken(pcId)`**: `jwt.sign({ pcId }, JWT_SECRET, { expiresIn: '365d' })`.
- **`setSessionCookie(req, res, token)`** *(2026-07-29)*: `res.cookie('kneos_token', token, { httpOnly: true, sameSite: 'strict', secure: req.secure, maxAge: 365d })`.
- **`readSessionToken(cookies)`** *(firma cambiada 2026-07-29 — antes tomaba el header `Authorization`)*: lee `cookies.kneos_token`, verifica firma y vencimiento con `jwt.verify`, devuelve `pcId` o `null` (nunca lanza — usa `try/catch` interno). No toca la base.
- **`requireAuth(req, res, next)`**: si `readSessionToken(req.cookies)` no devuelve nada → `401 { error: "No autorizado" }`. Si devuelve, setea `req.pcId` y sigue. Cookie ausente, mal formada, con firma inválida o vencida dan todas el mismo 401 — distinguirlas no cambia la reacción del cliente (reautenticar) y solo filtraría información.
- Montado con `router.use(requireAuth)` en cada archivo de rutas protegido (no en `server.js` — así la decisión de qué es público queda al lado del handler). Único router mixto: `kfruitRoutes` (`GET /score` público, el resto requiere token).
- `server.js` monta `cookie-parser` (`app.use(cookieParser())`, antes de las rutas) para que `req.cookies` exista — única dependencia nueva de este cambio.

> [!warning] Cookie `httpOnly` en vez de `localStorage` (2026-07-29)
> Hasta acá el token vivía en `localStorage['kneos_token']` y viajaba como header `Authorization: Bearer`, leído/escrito por `apiFetch.js` — legible por **cualquier** JS que corra en la página, incluida la consola del navegador (justo lo que se discutió: un XSS en cualquier parte de la app, o simplemente abrir la consola, alcanzaba para leerlo). El usuario pidió explícitamente que ni el token ni el `pc_id` fueran accesibles desde la consola de JS. Se cambió a una cookie `httpOnly` — el navegador la manda solo en cada request al mismo origen, y ninguna API de JS (`document.cookie`, `localStorage`, nada) puede leerla ni escribirla; el server es el único que la pone (`Set-Cookie`) y la lee (`req.cookies`, vía `cookie-parser`).
>
> `sameSite: 'strict'` es la mitigación de CSRF — sin ella, una cookie de sesión se manda automáticamente en cualquier request cross-site (a diferencia del header `Authorization`, que requiere JS para adjuntarse, y por eso no tenía este problema). `Strict` corta la cookie en cualquier contexto cross-site, así que no hace falta un token CSRF aparte. `secure: req.secure` evita romper el desarrollo local por HTTP (una cookie `Secure` sobre HTTP simplemente no se setea) y se activa sola en producción sobre HTTPS.
>
> Simplificó bastante el cliente: `apiFetch.js` perdió `getToken`/`setToken`/`clearToken` y el manejo especial de 401 (ya no hay nada que limpiar desde JS) — cada `fetch` solo necesita `credentials: 'same-origin'` para que el navegador mande la cookie. `session.js` (`startSession()`) ya no lee ni guarda nada: los dos endpoints ponen la cookie como efecto secundario de la respuesta, invisible para el cliente.
>
> Verificado con Playwright: `document.cookie` devuelve `""` después de arrancar la sesión (nada visible para la página ni para la consola), mientras que a nivel de red la cookie existe con `httpOnly: true, sameSite: 'Strict'` y los requests autenticados siguen funcionando (`GET /iconRoutes/icons` → 200).

## Modelo (`models/session.js`, sin cambios)

- **`guardarSesion(pcId)`**: `prisma.sessions.create({data:{pc_id: pcId}})`. Devuelve la fila creada.
- **`existeSesion(pcId)`**: `prisma.sessions.findUnique({where:{pc_id: pcId}})`, devuelve `!!session`.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|session.js (startSession())]], llamado una única vez en `KNEOS.js` al arrancar, bloqueando el resto del bootstrap hasta resolverse. Dos pasos secuenciales sin recursión: intenta renovar (`/session/verificar`), si el body devuelto no trae `valid: true` pide una nueva (`/session/nueva`). Como mucho dos requests, nunca un bucle. *(2026-08-13)* Como ambos endpoints ahora responden siempre `200`, `pingSession()` en el cliente dejó de mirar `response.ok` para decidir éxito — lee el JSON y chequea el flag correspondiente (`valid` o `success`) por endpoint.

> [!info] Limpieza de `kneos_pc_id` (2026-07-29)
> `startSession()` empieza borrando `localStorage.removeItem('kneos_pc_id')` — la clave del sistema anterior al JWT, que ningún código lee ni escribe desde la migración pero que quedaba pegada en el navegador de cualquiera que hubiese usado la app antes. No es un riesgo de seguridad (no otorga ningún acceso, el backend ya no confía en ningún `pc_id` que venga del cliente), es limpieza de dead data. Se agregó a partir de que el usuario notó dos UUIDs distintos conviviendo en su `localStorage` (`kneos_pc_id` viejo + el `pcId` real dentro del `kneos_token` de esa sesión, sin relación entre sí). Sigue siendo válida después del cambio a cookie — `localStorage` queda completamente vacío en el flujo normal.
