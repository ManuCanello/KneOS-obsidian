---
tags:
  - portfolio/kneos
  - backend
---

# Módulo FlipCoin

⬅️ Volver a [[Backend]]

Log de tiradas del minijuego de cara/cruz ([[FlipCoin]]) — global (sin `pc_id`), crece con cada partida jugada. A diferencia de [[Módulo Hangman]] (catálogo fijo, solo lectura), este módulo tiene un endpoint de escritura: mismo criterio que el leaderboard de [[Módulo Kfruit]] (`kfruit_score`).

## Endpoints (`routes/flipcoinRoutes.js`, montado en `/flipcoinRoutes`)

| Método | Path | Controller | Auth |
|---|---|---|---|
| POST | `/resultado` | `addResultado` | requiere token + `flipcoinLimiter` (30/min) |
| GET | `/porcentaje` | `getPorcentaje` | requiere token |
| GET | `/historial` | `getHistorialResultados` | requiere token |

Todo requiere `requireAuth`, incluidas las lecturas — a diferencia de `GET /kfruitRoutes/score` (público, leaderboard de solo lectura). Ver [[Módulo Session]].

## Controllers (`controllers/flipcoinController.js`)

- **`addResultado`**: `resultado` del body → valida que sea literalmente `0` o `1` (400 si no); si pasa, `insertResultado(resultado)`, responde `{ success: true }`.
- **`getPorcentaje`**: sin params → `getPorcentajes()`, responde `{ cara, cruz, total }` (conteos, no porcentajes ya calculados — el cálculo final es del frontend, ver [[FlipCoin]]).
- **`getHistorialResultados`**: sin params → `getHistorial()`, responde `{ historial }` (array de `0`/`1`, más reciente primero).

## Modelo (`models/flipcoinModel.js`)

- **`insertResultado(resultado)`**: `create` en `flipcoin_resultados`.
- **`getPorcentajes()`**: `count()` total + `count({where:{resultado:0}})` para cara; cruz se deriva restando (`total - cara`), evita un tercer round-trip.
- **`getHistorial()`**: `findMany` ordenado `id_resultado desc`, `select: {resultado: true}` — devuelve solo la columna necesaria, mapeada a un array plano de enteros.

## Dominio de negocio

`resultado`: `0` = cara, `1` = cruz — mismo mapeo que `girarMoneda()`/`grabar()` del Java original (que escribía literalmente `"0"`/`"1"` a un archivo de texto, una tirada por línea). El sorteo en sí (`Math.random()`) se hace client-side en `FlipCoin.js`, igual que el resto de la lógica de juego del proyecto (reparto de BlackJack, física de Kfruit) — el backend acá es puramente persistencia, no autoridad del resultado.

## Consumido por

Servicio frontend `FlipCoinServices` (ver [[Frontend Model Services Utils#Services]]), usado por [[FlipCoin]].
