---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Kfruit

⬅️ Volver a [[Backend]]

Keybinds personalizables y leaderboard global del minijuego de fusión de frutas ([[Kfruit]]).

## Endpoints (`routes/kfruitRoutes.js`, montado en `/kfruitRoutes`)

| Método | Path | Controller | Auth |
|---|---|---|---|
| GET | `/keybinds` | `getUserKeybinds` | requiere token |
| PATCH | `/keybinds` | `editKeybinds` | requiere token |
| GET | `/score` | `getScores` | público — leaderboard de solo lectura, no expone datos de nadie |
| POST | `/score` | `addScore` | requiere token (2026-07-28) |

Único router "mixto" del proyecto: todo requiere `requireAuth` excepto `GET /score`. Ver [[Módulo Session]].

## Controllers (`controllers/kfruitController.js`)

- **`getUserKeybinds`**: `getKeybinds(req.pcId)` (upsert atómico: si no existen, se crean con los defaults del schema). Devuelve `{id_keybinds, pc_id, moveleft, moveright, drop}`.
- **`editKeybinds`**: `moveleft, moveright, drop` del body + `req.pcId` del token → valida los 3 como strings no vacíos (2026-07-27); `updateKeybinds(...)` (`updateMany` por `pc_id`, no hay filtro por PK única). Responde `{ success: true }`. No valida que existan filas previas.
- **`addScore`**: `name, score` → valida (`name` string no vacío, máx. 3 caracteres tras `trim`; `score` entero entre 0 y 999999) devolviendo 400 si falla; si pasa, `insertScore(name.trim(), score)`, devuelve el registro creado completo. Ruta requiere token desde 2026-07-28, además del rate limiting.
  > [!info] Validación + auth (2026-07-27, endurecido 2026-07-28)
  > Antes no validaba nada — cualquiera podía insertar `{name, score}` arbitrarios sin ninguna sesión. Se agregó validación básica de tipo/rango en el controller, rate limiting (`middlewares/rateLimiters.js` → `scoreLimiter`, 5 requests/minuto por IP vía `express-rate-limit`), y desde 2026-07-28 también `requireAuth` — hace falta un token de sesión válido, ya no es un endpoint totalmente anónimo. Sigue siendo posible mandar puntajes "legítimos" falsos con una sesión propia dentro del rango permitido — eso es aceptado, solo se acotó el abuso burdo. Ver [[Deuda Técnica]].
- **`getScores`**: sin params (el controller no lee `req.query.limit`) → `getTopScores()` (default 10, siempre 10 en la práctica). Sin `requireAuth` — es el leaderboard público.

## Modelo (`models/kfruitModel.js`)

- **`getKeybinds(pc_id)`**: `prisma.kfruit_keybinds.upsert({where:{pc_id}, update:{}, create:{pc_id}})` — con los defaults del schema (`ArrowLeft`, `ArrowRight`, `ArrowDown;Space`) si es la primera vez.
  > [!success] Condición de carrera cerrada (2026-07-27)
  > Antes era `findFirst` + `create` manual: dos requests concurrentes antes del primer `create` podían generar dos filas para el mismo `pc_id`, porque no había constraint único ahí. Se agregó `UNIQUE (pc_id)` en Postgres (`kfruit_keybinds_pc_id_unique`, tabla estaba vacía) y se reescribió como `upsert` atómico, que requiere ese unique para funcionar. Probado con 10 llamadas concurrentes al mismo `pc_id` nuevo: las 10 devuelven el mismo `id_keybinds` y queda una sola fila. Ver [[Deuda Técnica]].
- **`updateKeybinds(pc_id, moveleft, moveright, drop)`**: `updateMany` filtrando por `pc_id`.
- **`insertScore(name, score)`**: `create` en `kfruit_score`.
- **`getTopScores(limit = 10)`**: `findMany` ordenado `score desc`, `take: limit`.

## Dominio de negocio

Juego tipo "fruit-catching/fusión" (estilo Suika Game) — `moveleft`/`moveright`/`drop` sugieren mover una fruta horizontalmente y soltarla para que caiga. Cada sesión (`pc_id`) tiene su propia configuración de teclas. El leaderboard (`kfruit_score`) es **independiente del `pc_id`** — solo guarda `name` + `score`, es un ranking global tipo arcade con nombre libre, no ligado a la sesión ni deduplicado por jugador.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|KfruitServices]], usado por [[Kfruit]].
