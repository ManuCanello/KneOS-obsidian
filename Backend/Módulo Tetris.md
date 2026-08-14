---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Tetris

⬅️ Volver a [[Backend]]

Keybinds personalizables y leaderboard global del minijuego de bloques ([[Tetris]]). Agregado 2026-08-13, calcado 1:1 de [[Módulo Kfruit]] — mismo par de tablas, mismos endpoints, misma división de responsabilidades — salvo que Tetris tiene una acción más (rotar) que Kfruit.

## Endpoints (`routes/tetrisRoutes.js`, montado en `/tetrisRoutes`)

| Método | Path | Controller | Auth |
|---|---|---|---|
| GET | `/keybinds` | `getUserKeybinds` | requiere token |
| PATCH | `/keybinds` | `editKeybinds` | requiere token |
| GET | `/score` | `getScores` | público — leaderboard de solo lectura, no expone datos de nadie |
| POST | `/score` | `addScore` | requiere token + `scoreLimiter` |

Router "mixto" igual que `kfruitRoutes.js`: todo requiere `requireAuth` excepto `GET /score`. Reusa el `scoreLimiter` existente en vez de sumar un limiter propio — mismo criterio (5 req/min por IP) ya vale para cualquier leaderboard con forma `{name, score}`, no hace falta uno por juego. Ver [[Módulo Session]].

## Controllers (`controllers/tetrisController.js`)

- **`getUserKeybinds`**: `getKeybinds(req.pcId)` (upsert atómico, defaults del schema si es la primera vez). Devuelve `{id_keybinds, pc_id, moveleft, moveright, rotate, drop, softdrop}`.
- **`editKeybinds`**: `moveleft, moveright, rotate, drop, softdrop` del body + `req.pcId` del token → valida las 5 como strings no vacíos; `updateKeybinds(...)` (`updateMany` por `pc_id`). Responde `{ success: true }`. (`softdrop` agregado 2026-08-14, a pedido del usuario — "bajada rápida" configurable, default `KeyS;ArrowDown`, distinta de `drop`/hard-drop.)
- **`addScore`**: `name, score` → misma validación que Kfruit (`name` string no vacío, máx. 3 caracteres tras `trim`; `score` entero entre 0 y 999999); `insertScore(name.trim(), score)`, devuelve el registro creado.
- **`getScores`**: sin params → `getTopScores()` (default 10). Sin `requireAuth`.

## Modelo (`models/tetrisModel.js`)

- **`getKeybinds(pc_id)`**: `prisma.tetris_keybinds.upsert({where:{pc_id}, update:{}, create:{pc_id}})` — defaults del schema (`moveleft: "KeyA;ArrowLeft"`, `moveright: "KeyD;ArrowRight"`, `rotate: "KeyC;ArrowUp"`, `drop: "Space"`, `softdrop: "KeyS;ArrowDown"`).
- **`updateKeybinds(pc_id, moveleft, moveright, rotate, drop, softdrop)`**: `updateMany` filtrando por `pc_id`.
- **`insertScore(name, score)`**: `create` en `tetris_score`.
- **`getTopScores(limit = 10)`**: `findMany` ordenado `score desc`, `take: limit`.

## Dominio de negocio

El Java original (`Juego`/`Tablero`/`Niveles`/`Puntos`) no tenía nada de esto — ni teclas configurables (hardcodeadas a a/d/c/espacio) ni ningún guardado de puntaje, ver [[Tetris]]. Ambas features son enteramente nuevas, agregadas a pedido explícito para calzar con el resto de las apps `FileType.GAME` que ya tienen ese estándar (Kfruit). `tetris_score`, igual que `kfruit_score`, es independiente del `pc_id` — leaderboard global anónimo por nombre libre (3 iniciales), no ligado a la sesión.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|TetrisServices]], usado por [[Tetris]].
