---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Kchess

⬅️ Volver a [[Backend]]

Backend del modo ONLINE de [[Kchess]] (2026-11-04): lobby, invitaciones, relay de jugadas con control de turno e historial. Calco de [[Módulo Truco]] (mismo handshake de WebSocket, mismo patrón de tres capas) pero más simple: un solo "modo", así que el lobby es un único `Set` y no hay selector de modo.

## Decisión de arquitectura: las dos PCs corren el motor, el server solo relaya

El motor de ajedrez (`public/KneOS/js/apps/Kchess.js`: `_pseudoMoves`, `_legalMoves`, jaque/mate/tablas) corre ENTERO en el navegador de cada jugador, igual en los dos. A diferencia de Truco no hay anfitrión autoritativo ni snapshots de estado (en ajedrez no hay información oculta): cada jugada se manda como `{from, to}` y el rival la **valida con su propio motor** antes de aplicarla (si no es legal, la ignora). El servidor no entiende de ajedrez; solo verifica dos cosas en `pushMove`: que quien manda sea parte de la partida y que **le toque** (el hub guarda `turn` por partida, alterna blancas/negras).

> [!warning] Riesgo de confianza aceptado (mismo criterio que Truco)
> Quien edite su propio JS puede mandar una jugada ilegal; el rival la rechaza, pero nadie lo valida server-side. El peor caso es una partida falseada entre dos personas, no una ventaja sobre un tercero. El resultado también se informa desde el cliente.

## Endpoints (`routes/kchessRoutes.js`, montado en `/kchessRoutes`)

Todo pasa por `requireAuth`. Identidad siempre de `req.pcId`. Reusa los dos limiters de Truco (`trucoRelayLimiter` para jugadas, `trucoLobbyLimiter` para lobby/invitaciones/cierre).

| Método | Path | Controller | Notas |
|---|---|---|---|
| POST | `/lobby` | `joinLobby` | Exige alias propio (403 si no). Devuelve `{players}` y avisa a los demás (`kchess-lobby-update`) |
| DELETE | `/lobby` | `leaveLobby` | |
| POST | `/invite` | `invite` | `{pcId, minutes?}` (`minutes` null = normal, entero 1–180 = partida por tiempo): los dos tienen que estar en el lobby. Push `kchess-invite` (con `minutes`) |
| POST | `/invite/:inviteId/accept` | `acceptInvite` | Solo el destinatario. Crea la fila (`createMatch`), registra la partida en memoria y manda `kchess-match-start` a los dos: **el que invita juega con blancas** |
| POST | `/invite/:inviteId/decline` | `declineInvite` | Push `kchess-invite-declined` |
| POST | `/match/:matchId/move` | `pushMove` | `{from:[f,c], to:[f,c]}` (enteros 0–7). 403 si no es de la partida, **409 si no es su turno**. Push `kchess-move` al rival |
| POST | `/match/:matchId/clock` | `startClock` | Solo las blancas, una vez, al tocar la primera pieza en partida por tiempo. Push `kchess-clock-start` a las negras |
| POST | `/match/:matchId/finish` | `finishMatchRoute` | `{winnerPcId?, result}` (`mate`\|`stalemate`\|`abandon`\|`timeout`). Cualquiera de los dos puede cerrar; persiste y borra la partida de memoria |
| GET | `/history` | `getHistory` | `{stats:{won,lost,draws}, matches}` — solo partidas terminadas, con el alias del rival |

## `realtime/kchessHub.js` (path `/ws/kchess`)

Mismo handshake que `trucoHub.js` (cookie de sesión vía `readSessionToken`, chequeo de origin, `__kneosWsClaimed` para el catch-all de `server.js`) y mismo heartbeat. Estado en memoria: conexiones por `pc_id`, `lobby` (Set), invitaciones (vencen a los 60 s → `kchess-invite-expired`), y `matchByPcId` con `{matchId, whitePcId, blackPcId, turn}`.

Eventos push: `kchess-lobby-update`, `kchess-invite`, `kchess-invite-declined`, `kchess-invite-cancelled`, `kchess-invite-expired`, `kchess-match-start`, `kchess-move`, `kchess-clock-start`, `kchess-opponent-left`. `kchess-match-start` incluye `minutes`.

> [!bug] Abandono: la partida no se cierra al desconectarse el rival
> Primera versión (calcada de `trucoHub`): `_handleDisconnect` hacía `endMatch()` al irse un jugador. Probando con dos navegadores, el que se quedaba mandaba `POST /finish` con el abandono y el server contestaba **403** (el endpoint exige que la partida siga registrada), así que el historial nunca registraba abandonos. Ahora se suelta solo la referencia de quien se fue (`match.abandoned = true`), se avisa al otro y la partida queda viva hasta que el sobreviviente la cierre (o se vaya también, que la limpia). (Truco tiene el mismo patrón de `endMatch()` en la desconexión; no se tocó.)

## Persistencia (`models/kchessModel.js`, tabla `kchess_matches`)

Log de resultados, no estado de juego: `white_pc_id`, `black_pc_id`, `winner_pc_id?` (null = tablas), `result?`, `created_at`, `finished_at?`. Se crea la fila al aceptar la invitación y se completa al cerrar; el historial/estadísticas solo cuentan filas con `finished_at`. Una partida cortada sin cerrar queda con `finished_at` null y no aparece.

## Qué pasa en el cliente

Ver [[Kchess#Online]]. Resumen: el que da el mate (o deja sin jugadas al rival) llama a `/move` y después a `/finish` **en ese orden** (el finish borra la partida del server y rechazaría una jugada posterior); el que recibe un mate no reporta nada.
