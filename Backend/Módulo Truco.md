---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Truco

⬅️ Volver a [[Backend]]

Backend del modo ONLINE de [[Truco]] (2026-09-03): lobby por modo, invitaciones, relay de partida en curso, e historial. Segundo módulo del proyecto con WebSocket propio (el primero fue [[Módulo Chat]], que este módulo reusa para el alias y el chat entre jugadores en vez de reinventar ninguno de los dos).

## Decisión de arquitectura: anfitrión autoritativo, server solo relay

El motor de reglas de Truco (bazas/Envido/Flor/Truco 2.0, `public/KneOS/js/apps/Truco.js`, ~2900 líneas) corre ENTERO en el navegador del anfitrión, sin cambios propios más allá de tres enganches puntuales -- ver [[Truco#ONLINE -- PC vs PC (2026-09-03)]]. El server nunca interpreta el contenido de una jugada ni valida una regla de Truco: `POST /trucoRoutes/match/:id/state` y `/action` son relés puros, HTTP → WebSocket, sin tocar el payload. Esto es lo que hace que Punta y Hacha (2026-09-03, ver [[Truco#Punta y Hacha + 18 Power-Ups (2026-09-03) -- reemplaza la carta de poder]]) haya necesitado **cero cambios de backend**: sus 5 `remoteScreen` kinds nuevos (`"powerup"`, `"informante"`, `"revelar"`, `"sabotaje"`, `"memoria"`) viajan dentro del mismo `{state}`/`{action}` opaco que ya relayaban `/state`/`/action` para `"hand"`/`"discard"` -- el server nunca supo, ni necesitó saber, que existen.

> [!warning] Riesgo de confianza aceptado (decisión explícita con Manuel)
> Un anfitrión que edite su propio JS podría hacer trampa en su propia partida (nadie valida server-side que una jugada sea legal). Se aceptó a propósito: no hay cuentas de usuario en todo el proyecto (ver [[Deuda Técnica#Autenticación real — decisión revertida (2026-07-28)]]), es un portfolio sin nada en juego, y la alternativa -- portar el motor completo a Node para poder validarlo -- es varias veces más trabajo que el resto de la feature junta. El peor caso es una fila de `truco_matches` falseada en el propio historial de quien hizo trampa, no una ventaja real contra nadie más.

## Endpoints (`routes/trucoRoutes.js`, montado en `/trucoRoutes`)

Todo pasa por `requireAuth` (`router.use`), igual que `chatRoutes.js`. Identidad siempre de `req.pcId`, nunca del body.

| Método | Path | Controller | Notas |
|---|---|---|---|
| POST | `/lobby` | `joinLobby` | `{mode}` (`con_flor`\|`sin_flor`\|`v2`). Exige alias propio (403 si no, mismo criterio que `chatController.sendMessage`). Devuelve `{players}` (los demás del mismo modo) y dispara `truco-lobby-update` a todo el lobby |
| DELETE | `/lobby` | `leaveLobby` | Sale del lobby actual |
| POST | `/invite` | `invite` | `{pcId}` del destinatario -- exige que quien invita Y el destino estén en el lobby del MISMO modo. Push `truco-invite` |
| POST | `/invite/:inviteId/accept` | `acceptInvite` | Solo el destinatario. Crea la fila en `truco_matches` (`createMatch`), registra la partida en memoria del hub (`registerMatch`) y push `truco-match-start` a los dos con su `role` (`host`\|`guest`) |
| POST | `/invite/:inviteId/decline` | `declineInvite` | Solo el destinatario. Push `truco-invite-declined` |
| POST | `/match/:matchId/state` | `pushState` | Solo el anfitrión de esa partida. Relay puro (`{state}` → push `truco-state` al invitado) con un tope de tamaño (`MAX_STATE_JSON_LENGTH`, 20 KB) como única validación |
| POST | `/match/:matchId/action` | `pushAction` | Solo el invitado. Relay puro en sentido inverso (`{action}` → push `truco-action` al anfitrión) |
| POST | `/match/:matchId/finish` | `finishMatchRoute` | Cualquiera de los dos participantes (no solo el anfitrión -- el que sobrevive a un abandono también necesita poder cerrar la fila, ver "Abandono" abajo). Persiste el resultado (`finishMatch`) y libera la partida del hub (`endMatch`) |
| GET | `/history` | `getHistory` | `{byMode, matches}` para `req.pcId`, ver `models/trucoModel.js` |

`middlewares/rateLimiters.js`: `trucoLobbyLimiter` (30/min por `pc_id`, lobby/invitaciones/finish) y `trucoRelayLimiter` (240/min por `pc_id`, `/state`+`/action` -- techo alto a propósito, cada `_render()` del anfitrión dispara un `/state`).

## `realtime/trucoHub.js` -- WebSocket + estado efímero del lobby

Calco de `chatHub.js` en su transporte (mismo handshake -- cookie `kneos_token` + chequeo `Origin === Host`, mismo heartbeat de 30s, **sin `ws.on('message')`**, el socket es solo push), path propio `/ws/truco`.

Lo que NO tiene equivalente en `chatHub.js`: el lobby/las invitaciones/el emparejamiento de una partida viven acá, en memoria del proceso, no en Postgres -- son estado de "quién está esperando ahora mismo", no historial:

- `lobbyByPcId: Map<pc_id, {mode}>` -- `joinLobby`/`leaveLobby`/`getLobbyPcIds(mode, excludePcId)`.
- `invitesById: Map<invite_id, {fromPcId, toPcId, mode, timer}>` -- `createInvite` arma un timer de 60s (`INVITE_TTL_MS`) que la vence sola y avisa (`truco-invite-expired`) si nadie responde.
- `matchByPcId: Map<pc_id, {matchId, mode, hostPcId, guestPcId}>` -- la MISMA fila queda referenciada desde los dos `pc_id`, así `getMatchForPc(pcId)` resuelve "en qué partida estoy" en O(1) desde cualquiera de los dos lados; el controller la usa para validar quién puede pushear `/state` (solo `hostPcId`) vs `/action` (solo `guestPcId`).

### Abandono (desconexión a mitad de partida)

Al cerrarse el último socket de un `pc_id` (`_handleDisconnect`, mismo patrón multi-pestaña que `chatHub` -- un `pc_id` sale recién cuando cierra la última): sale del lobby, cancela cualquier invitación pendiente en cualquiera de los dos sentidos, y si estaba en una partida push `truco-opponent-left` al rival. El CLIENTE (no el hub) decide qué hacer con eso: el que queda llama `POST /match/:matchId/finish` con el último marcador que alcanzó a ver y se declara ganador -- por eso `finishMatchRoute` acepta la llamada de CUALQUIERA de los dos participantes, no solo del anfitrión (ver `Truco.js#_reportAbandonment`/`_handleOpponentLeft`).

## Modelo (`models/trucoModel.js`)

- `createMatch(mode, hostPcId, guestPcId)` / `finishMatch(matchId, {hostScore, guestScore, winnerPcId})`: `winnerPcId` puede ser `null` (partida sin definirse, no debería pasar en la práctica ya que hasta un abandono siempre define un ganador).
- `getMatchHistory(pcId, limit=30)`: últimas N partidas jugadas como anfitrión O invitado (`OR` en el `where`), con el nickname del oponente resuelto en el mismo roundtrip (`include: {host_session, guest_session}`).
- `getStatsByMode(pcId)`: ganadas/perdidas por `truco_mode`, ignorando partidas sin `winner_pc_id`.

## Esquema (`prisma/schema.prisma`)

```prisma
enum truco_mode { con_flor  sin_flor  v2 }

model truco_matches {
  match_id      Int         @id @default(autoincrement())
  mode          truco_mode
  host_pc_id    String
  guest_pc_id   String
  host_score    Int         @default(0)
  guest_score   Int         @default(0)
  winner_pc_id  String?
  created_at    DateTime?   @default(now())
  finished_at   DateTime?
  host_session  sessions    @relation("truco_matches_host", ...)
  guest_session sessions    @relation("truco_matches_guest", ...)
}
```

Log de resultados, no estado de juego -- mismo criterio que `car_race_results`/`flipcoin_results`. `winner_pc_id` es un `VarChar` suelto (siempre igual a `host_pc_id` o `guest_pc_id`) en vez de una TERCERA relación con `sessions` -- una fila solo necesita dos relaciones nombradas (`truco_matches_host`/`truco_matches_guest`) para poder tener dos FKs distintas al mismo modelo `sessions`; agregar una tercera para `winner_pc_id` habría sido redundante, mismo criterio que `car_race_results.car` (VarChar suelto, no FK).

## Reuso deliberado de `Módulo Chat`

- **Alias**: cero código nuevo -- `sessions.nickname` y `POST /chatRoutes/nickname` tal cual, ver [[Truco#ONLINE -- PC vs PC (2026-09-03)]].
- **Chat entre jugadores**: DM real (`POST /chatRoutes/dm` + `chat_messages`), no un chat efímero propio -- se seguía viendo después desde KneChat.
- **`chatSocket.js` parametrizado** (2026-09-03): `constructor(path = "/ws/chat")` en vez de la URL fija que tenía antes -- el modo ONLINE de Truco reusa la clase entera (backoff exponencial, pub/sub, `close()`) contra `/ws/truco` sin duplicar ni una línea. KneChat sigue instanciándola sin argumentos, comportamiento idéntico.

## Bug real encontrado y corregido: `server.on('upgrade')` con dos hubs

A diferencia de los middlewares de Express (`next()`, cortocircuito), Node llama a **todos** los listeners registrados de un mismo evento del `EventEmitter` -- incluido `'upgrade'` de `http.Server`. Con `chatHub.js` y `trucoHub.js` enganchados los dos sobre el mismo `server`, cada handshake de `/ws/chat` disparaba TAMBIÉN el listener de `trucoHub` (registrado después): veía que el path no era `/ws/truco` y destruía el socket que `chatHub` le acababa de entregar a su propio `WebSocketServer` un instante antes -- el chat de la partida se conectaba y se cortaba en loop (backoff de `chatSocket.js` reintentando sin parar, nunca estable).

**Fix**: ningún hub destruye por mismatch de path -- solo hace `return` y, si el path SÍ es el suyo, marca `socket.__kneosWsClaimed = true` antes de seguir. `server.js` agrega un único listener de `'upgrade'` al final (después de adjuntar los dos hubs) que destruye lo que ningún hub reclamó. Encontrado y corregido jugando una partida real con Playwright (dos `browser.newContext()`, dos `pc_id`) -- el mensaje de chat llegaba al servidor (200) pero nunca al otro lado, porque el socket de push ya estaba muerto para cuando el broadcast intentaba usarlo.

## Consumido por

App frontend [[Truco]], vía `public/KneOS/js/services/TrucoServices.js` (HTTP, calco de `ChatServices.js`) y `chatSocket.js` (WebSocket, parametrizado -- ver arriba).
