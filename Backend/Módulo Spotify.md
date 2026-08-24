---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Spotify

⬅️ Volver a [[Backend]]

Vínculo de cuenta con Spotify y proxy autenticado hacia su Web API, para [[Knefy]]. Agregado 2026-08-24. A diferencia del resto de los proxies del proyecto ([[Módulo Groq]], [[Módulo Map]] — una URL fija de terceros, sin estado por sesión), acá hay un flujo OAuth completo **por `pc_id`**, con su propia tabla (`spotify_accounts`).

## Por qué no se consume el backend de SpotifyController

`C:\Users\canel\Downloads\Codigo\SpotifyController` (proyecto aparte, fuera de este repo) ya resuelve el OAuth de Spotify, pero está pensado para una Raspberry Pi Pico 2W: un único `refresh_token` global en `data/tokens.json` (no por sesión), nunca expone un access token al cliente (solo acepta `X-API-Key`), sin CORS, y le faltan `/search`, `/devices`, `/queue`, `/seek`, playlists y reproducir un URI puntual (su `/play` es solo *resume*). Este módulo **porta** su lógica de tokens (intercambio `authorization_code`→tokens con `Basic base64(id:secret)`, refresh lazy con margen de 5s, re-guardar el refresh token si Spotify lo rota) a `utils/spotifyClient.js`, pero stateless por `pc_id` en vez de un único refresh_token en disco. Dos bugs del original que **no** se portaron: `/callback` sin `state` (CSRF de autorización) y la comparación del header con `!==` (no timing-safe) — acá el `state` es un JWT firmado y no hay comparación de secreto alguna (la Web API se llama con el bearer, no con una key propia).

## Restricciones de la plataforma (development mode)

- Desde el 9-mar-2026, una app en development mode exige que **el dueño de la app tenga Spotify Premium activo** o deja de funcionar para todo el mundo.
- Solo pueden loguearse los usuarios agregados a mano en el dashboard de developer.spotify.com: **5** para apps nuevas, **25** si la app es anterior al 11-feb-2026 (grandfathered) — *Extended Quota* pide empresa registrada con 250k MAU, inviable acá.
- Los `preview_url` de 30s están muertos para apps nuevas — sin fallback de audio para quien no tenga cuenta vinculada.
- `GET /search` sigue vivo pero con `limit` máximo **10** (antes 50).
- `GET /playlists/{id}/tracks` fue **removido**, reemplazado por `GET /playlists/{id}/items` (el que usa [[Knefy]]) — y el shape también cambió: cada elemento trae `item` (track *o* episode), no `track` (deprecated, sin fecha de remoción pero ya sin poblar — ver [[Knefy#Biblioteca, playlists y búsqueda]]).
- Muertos para apps nuevas: `GET /tracks`/`/albums`/`/artists` (batch), `/browse/*`, `/artists/{id}/top-tracks`, `/users/{id}`, `/markets`.
- Vivos y usados acá: todo `/me/player/*`, `GET /me/playlists`, `GET /me/tracks`, `GET /me/albums`, `GET /playlists/{id}/items`, `GET /albums/{id}`, `GET /artists/{id}`, `GET /tracks/{id}`, `GET /search`.

## Endpoints (`routes/spotifyRoutes.js`, montado en `/spotifyRoutes`)

`/callback` se registra **antes** de `router.use(requireAuth)` — Spotify redirige ahí desde su propio dominio, sin garantía de que la cookie de sesión viaje (el `pc_id` sale del `state` firmado, no de `req.pcId`). El resto del router sí requiere auth + `spotifyLimiter` (60 req/min por `pc_id`, ver `middlewares/rateLimiters.js`).

| Método | Path | Auth | Controller |
|---|---|---|---|
| GET | `/callback` | público (state firmado) | `callback` |
| GET | `/login` | requireAuth | `login` |
| GET | `/token` | requireAuth | `token` |
| ALL | `/api/*splat` | requireAuth | `proxy` |
| GET | `/cover` | requireAuth | `cover` |
| DELETE | `/account` | requireAuth | `logout` |

> [!info] `/api/*splat`, no `/api/*` (Express 5)
> Express 5 corre sobre una versión de `path-to-regexp` que exige nombrar todo wildcard (`Missing parameter name`) — `/api/*` tira en el arranque, hay que escribir `/api/*splat` y leer `req.params.splat` (array de segmentos, se reconstruye con `.join('/')`). Sin precedente en el resto del proyecto: es el único router con un proxy de path abierto.

## Controllers (`controllers/spotifyController.js`)

- **`login`**: firma `state = jwt.sign({pcId: req.pcId, nonce}, JWT_SECRET, {expiresIn:'10m'})` y redirige a `getAuthorizeUrl(state)`. El `state` viaja como JWT (no un id de sesión en memoria del server) para no perder el vínculo `pc_id` si el proceso se reinicia entre el click de "CONECTAR CUENTA" y que la persona vuelva de accounts.spotify.com.
- **`callback`**: sin `code`/`state`/con `error` → cierra el popup en silencio (`<script>window.close()</script>`). Verifica el JWT del `state` (de ahí sale el `pcId`); si es inválido/expiró, también cierra en silencio — **regla del proyecto, cero cartel de error** (ver [[Reglas]]). Si todo resuelve: `exchangeCode(code)` → `saveAccount(pcId, refresh_token, scope)` → responde un HTML que hace `window.opener?.postMessage({type:"knefy:auth"}, location.origin); window.close()`.
- **`token`**: `ensureAccessToken(req.pcId)` → `{access_token, expires_in}`, o `401` si la sesión no tiene cuenta vinculada. Es lo que [[Knefy]] llama tanto para decidir qué pantalla mostrar como para el `getOAuthToken` del Web Playback SDK.
- **`proxy`**: reconstruye el path desde `req.params.splat`, lo valida contra una whitelist de prefijos (`me/player`, `me/playlists`, `me/tracks`, `me/albums`, `search`, `playlists/`, `albums/`, `artists/`, `tracks/`, y `me` a secas — sumado 2026-08-24, perfil propio, ver [[Knefy#Biblioteca, playlists y búsqueda]] — fuera de eso, `404` silencioso, para no dejar un open proxy autenticado hacia cualquier endpoint de Spotify), arma la query desde `req.query`, y reenvía con el bearer de `ensureAccessToken`. `204` se reenvía tal cual (la mayoría de `/me/player/*` no devuelve body); si Spotify devuelve error, se reenvía el `status` + body tal cual, sin traducir.
- **`cover`**: proxea binario `?url=` — **solo si el host es `i.scdn.co`** (si no, `400`), `Content-Type` heredado de la respuesta real, `Cache-Control: public, max-age=86400`. Necesario (no cosmético) para que [[Knefy]] pueda leer `getImageData` del `<canvas>` de `utils/asciiArt.js`: una imagen de otro dominio sin las cabeceras correctas deja el canvas "tainted" (`SecurityError`).
- **`logout`**: `deleteAccount(req.pcId)` + `clearAccessToken(req.pcId)` (la cache en memoria sobrevive a un logout si no se limpia, y un re-login inmediato con otra cuenta reusaría el access token viejo hasta que expire).

## Cliente OAuth (`utils/spotifyClient.js`)

- **`SCOPES`**: `streaming user-read-email user-read-private user-read-playback-state user-modify-playback-state user-read-currently-playing playlist-read-private playlist-read-collaborative user-library-read` — `streaming` (lo que habilita el Web Playback SDK) **exige** pedir también `user-read-email`/`user-read-private` junto a él, aunque acá no se use el perfil para nada más.
- **`getAuthorizeUrl(state)`** / **`exchangeCode(code)`**: Authorization Code Flow clásico contra `accounts.spotify.com`, con `Basic base64(SPOTIFY_CLIENT_ID:SPOTIFY_CLIENT_SECRET)`.
- **`ensureAccessToken(pc_id)`**: cache en memoria `Map<pc_id, {accessToken, expiresAt}>`, margen de 5s antes del vencimiento (mismo patrón que `SpotifyController/src/spotify/client.js`). Si no hay entrada vigente, busca `spotify_accounts` por `pc_id` — sin fila, `null` (sesión no vinculada); con fila, refresca contra `accounts.spotify.com/api/token` con `grant_type=refresh_token`. Si Spotify devuelve un `refresh_token` nuevo (rota ocasionalmente), se regraba con `saveAccount` antes de cachear el access token — si no, el próximo refresh usaría uno ya inválido.
- **`clearAccessToken(pc_id)`**: borra la entrada de cache — usado por `logout` y tras guardar un vínculo nuevo (`callback`), para que un re-login no arrastre el access token de la cuenta anterior.

El access token **nunca se persiste** — vive 1h, se recalcula solo con el refresh_token guardado. Consistente con la regla del proyecto de que un secreto de terceros nunca llega al cliente ([[Módulo Groq]], [[Módulo Map]]): al browser solo llega el access token efímero, nunca el refresh_token ni el client secret.

## Modelo (`models/spotifyModel.js`)

Mismo molde upsert-por-`pc_id` que `theme_settings`/`kfruit_keybinds` (ver [[Módulo Theme]]):

- **`getAccount(pc_id)`**: `findUnique({where:{pc_id}})`.
- **`saveAccount(pc_id, refresh_token, scope)`**: `upsert`.
- **`deleteAccount(pc_id)`**: `deleteMany({where:{pc_id}})` (no `delete`, para no tirar si la fila ya no existe).

## Tabla `spotify_accounts` (`prisma/schema.prisma`, 2026-08-24)

```prisma
model spotify_accounts {
  pc_id         String   @id
  refresh_token String
  scope         String?
  created_at    DateTime @default(now())
  updated_at    DateTime @updatedAt
  sessions      sessions @relation(fields: [pc_id], references: [pc_id], onDelete: NoAction, onUpdate: NoAction)
}
```

A diferencia de `theme_settings`/`kfruit_keybinds` (PK `id_setting`/`id_keybinds` autoincremental separada + `pc_id @unique`), acá `pc_id` es directamente la PK — no hay ningún otro campo que necesite su propio id, una cuenta vinculada es 1:1 puro con la sesión. No cuelga de `files` (no hay relación con ningún ícono), así que no hace falta sumarla al purge de `models/iconModel.js` (a diferencia de `kpaint_files`/`camera_photos`/`txt`, ver [[Módulo KPaint]]).

## Variables de entorno

`SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET`, `SPOTIFY_REDIRECT_URI` (`http://127.0.0.1:3000/spotifyRoutes/callback` en local — Spotify exige HTTPS salvo loopback, y específicamente `127.0.0.1`, **no** `localhost`). El Client ID/Secret salen del dashboard de developer.spotify.com; se recomienda reusar la app ya creada para `SpotifyController` si es anterior al 11-feb-2026 (queda grandfathered con 25 usuarios en vez de 5) — los scopes se piden por autorización (`SCOPES` de `utils/spotifyClient.js`), no se configuran en el dashboard, así que agregar `streaming` acá no requiere tocar esa app.

## Consumido por

Servicio frontend `KnefyServices` (`public/KneOS/js/services/KnefyServices.js`), usado por [[Knefy]].
