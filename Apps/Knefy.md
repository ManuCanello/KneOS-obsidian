---
tags:
  - portfolio/kneos
  - apps
---

# Knefy

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Knefy.js` — extiende [[File]]. Extensión `"knefy"`, ícono `sources/appIcon/knefy.svg` (disco/vinilo: anillo + punto central, `stroke="currentColor"` en vez del `fill="currentColor"` habitual del resto de `appIcon/` — ver "Ícono" más abajo), `src = null`, tamaño declarado `41_000` bytes (no persiste contenido propio en BD — la única fila que le corresponde, `spotify_accounts`, cuelga de la sesión, no de un ícono, ver [[Módulo Spotify]]). Agregada 2026-08-24.

> [!abstract] Qué hace
> Cliente de Spotify — a diferencia de un mando a distancia, el **audio sale de la propia pestaña**: al abrirse registra un dispositivo Spotify Connect real vía el **Web Playback SDK** (`sdk.scdn.co`, cargado en runtime — no es un paquete npm) y ese dispositivo es al que apunta toda reproducción. Requiere **Spotify Premium** (excluidos los planes "Premium mobile-only") y que la sesión esté autorizada en el dashboard de la app (development mode: 5-25 usuarios, ver [[Módulo Spotify#Restricciones de la plataforma (development mode)]]).

## Por qué no consume el backend de SpotifyController

Existe un proyecto aparte, `C:\Users\canel\Downloads\Codigo\SpotifyController` (fuera de este repo), pensado para controlar Spotify desde una Raspberry Pi Pico 2W: guarda un único `refresh_token` global en un archivo (`data/tokens.json`), nunca expone un access token al cliente (solo acepta un header `X-API-Key`), no tiene CORS configurado y le faltan la mayoría de los endpoints que Knefy necesita (`/search`, `/devices`, `/queue`, `/seek`, playlists, reproducir un URI puntual). Knefy **porta** su lógica de OAuth (Authorization Code Flow con client secret, refresh lazy con margen de 5s) a `utils/spotifyClient.js` en este mismo backend, pero **por sesión** (`pc_id`) en vez de un único refresh_token global — ver [[Módulo Spotify]] para el detalle completo, incluidos los dos bugs del original que no se portaron (`/callback` sin `state`, comparación no timing-safe del header).

## Estados de pantalla (`_state`)

| Estado | Qué se ve | Cuándo |
|---|---|---|
| `disconnected` | Pantalla "CONECTAR CUENTA" | `GET /spotifyRoutes/token` devuelve 401 (esta sesión no tiene cuenta vinculada), o el SDK tira `account_error` (cuenta sin Premium) |
| `initializing` | "INICIALIZANDO DISPOSITIVO" | Hay token pero el SDK todavía no confirmó el `device_id` (evento `ready`) |
| `ready` | Reproductor completo | `createPlayer()` resolvió — ver `utils/spotifySdk.js` |

`account_error` vuelve directo a `disconnected` **sin ningún cartel** — regla del proyecto (ver [[Reglas]]): un visitante sin Premium ve la misma pantalla de conectar que alguien sin cuenta vinculada, nunca un mensaje de error.

### Vínculo de cuenta

Conectar (`_conectarCuenta`) abre un `window.open('/spotifyRoutes/login', ...)` — el popup hace todo el intercambio OAuth server-side (ver [[Módulo Spotify]]) y termina respondiendo un HTML mínimo que hace `window.opener?.postMessage({type:"knefy:auth"}, "*")` y se cierra solo. Knefy escucha ese `message` validando `event.source !== this._authPopup` (compara la ventana emisora, no un string de origin) y vuelve a llamar `_iniciar()`. Si el popup queda bloqueado por el navegador, `window.open` devuelve `null` y no pasa nada más — sin cartel, la persona puede reintentar.
>
> [!bug] `event.origin` rompía si la pestaña principal y el callback terminaban en hosts distintos (2026-08-24, resuelto)
> Primera versión: el listener comparaba `event.origin !== location.origin`, y el servidor mandaba el `postMessage` con `targetOrigin: location.origin` (del popup). `SPOTIFY_REDIRECT_URI` está fijo a `127.0.0.1` (requisito de Spotify, ver [[Módulo Spotify#Variables de entorno]]) — si la persona probaba KneOS en `localhost:3000`, el popup terminaba navegando a `127.0.0.1:3000/spotifyRoutes/callback` tras el login, un **origin distinto** para el navegador aunque sea el mismo server. El `postMessage` con `targetOrigin` explícito nunca se entregaba (el navegador lo descarta en silencio si no matchea el origin real del receptor), así que ni siquiera llegaba a evaluarse el chequeo del lado de Knefy. Confirmado en producción: el callback guardaba el `refresh_token` en `spotify_accounts` correctamente (server-side, sin restricción de origin) pero la UI se quedaba en "CONECTAR CUENTA" sin ningún indicio — "conecté la cuenta pero no pasó nada". Diagnosticado firmando a mano un JWT de sesión para el `pc_id` ya vinculado y pegándole directo a `GET /spotifyRoutes/token`, que devolvía `200` con un access_token real: confirmaba que todo el backend (OAuth, refresh, proxy) funcionaba, acotando el bug al handshake del popup. Fix: `targetOrigin: "*"` del lado del servidor (el mensaje no lleva nada sensible) + `event.source` del lado del cliente, agnóstico a qué string de origin tenga cada ventana.

## Reproducción

**Transporte del día a día** (play/pausa/siguiente/anterior/seek/volumen) va directo por los métodos del propio `Spotify.Player` (`togglePlay`/`nextTrack`/`previousTrack`/`seek`/`setVolume`) — son instantáneos porque el dispositivo objetivo es esta misma pestaña, no hace falta ida y vuelta a la Web API.

**Arrancar algo puntual** (un track de búsqueda, una playlist, un álbum, una canción guardada) sí pasa por la Web API — `KnefyServices.playContext(deviceId, body)` → `PUT /me/player/play?device_id=...` a través del proxy (ver [[Módulo Spotify#Proxy — `ALL /spotifyRoutes/api/*splat`]]), con `{uris:[...]}` para un track suelto o `{context_uri, offset}` para reproducir dentro de una playlist/álbum a partir de un punto.

**Cola**: `GET /me/player/queue` al abrir el panel (`_toggleCola`) y en cada `player_state_changed` — sin panel abierto no se pide (`_cargarCola` chequea que `_queueListEl` exista).

**Progreso sin polling**: `player_state_changed` da `{position, duration, paused}` en el momento del evento — Knefy interpola localmente cada 250ms (`_tickProgreso`, `setInterval`) sumando el tiempo transcurrido desde ese evento, y se resincroniza en el próximo `player_state_changed`. Barra clickeable (`_crearFooter`, `.knefyProgressTrack`) llama `player.seek(ms)` directo.

## Portada bitmap (dither estilo Camera)

`_actualizarPortada(track)` arma una `<img>` apuntando a `/spotifyRoutes/cover?url=<url-real-de-i.scdn.co>` (mismo origen que KneOS) y, en `onload`, la pasa a `drawDithered(img, canvas, COVER_SIZE)` (`utils/dither.js`, `COVER_SIZE = 32`) — mismo dither Floyd–Steinberg a 4 niveles + degradé hacia `--primary-color` que [[Camera]], pintado sobre un `<canvas class="knefyCover">` (no un `<pre>` de texto: acá el color final va horneado en los píxeles, no vía CSS). Pedido explícito del usuario (2026-08-24) reemplazando una primera versión en ASCII (rampa de caracteres por luminancia) — se prefirió el mismo lenguaje visual que ya usa Camera/KPaint/ImgFile en vez de un estilo nuevo.

A diferencia del ASCII (donde el color lo ponía el CSS y cambiar de tema no requería recalcular nada), acá el color queda **horneado en los píxeles** del canvas — así que cambiar el tema con una portada ya dibujada necesita reditherizar de cero. `_alCambiarTema` (arrow function, mismo patrón que `Camera._alCambiarTema`) escucha `THEME_COLOR_EVENT` y vuelve a llamar `drawDithered` con `this._ultimaPortadaImg` (la `<img>` ya cargada, guardada aparte para no repetir el `fetch` a `/spotifyRoutes/cover` solo para recolorear) — se suscribe en `_crearContenido()` y se desuscribe en `_destruir()` (`onClose` de la `Window`).

**Por qué el proxy es necesario, no cosmético**: leer `getImageData` de una imagen de otro dominio sin las cabeceras CORS correctas tira `SecurityError` (canvas "tainted") — `/spotifyRoutes/cover` resuelve eso sirviendo la portada desde el propio origen (ver [[Módulo Spotify]]). Se reditheriza solo cuando cambia la URL de portada (`_ultimaPortadaUrl`) o el tema, no en cada tick de progreso.

`utils/dither.js` adapta el algoritmo de `Camera._aplicarEfectoGameboy` a ditherizar una imagen ya cargada en vez de un frame de video en vivo (sin espejado ni recorte cuadrado por centro — una portada de Spotify ya viene cuadrada) en un solo pase en vez de dos. **No** está refactorizado como una función compartida entre las dos apps — `Camera.js` conserva su propia copia del algoritmo, `utils/dither.js` es exclusivo de Knefy, a propósito (evitar tocar código de Camera ya probado solo para deduplicar).

## Biblioteca, playlists y búsqueda

- **Sidebar** (`_renderSidebar`): "Canciones", "Álbumes" y las playlists **propias** del usuario (2026-08-24, a pedido explícito) — `_cargarBiblioteca` pide `GET /me` (perfil, solo para el `id`) y `KnefyServices.getAllMyPlaylists()` en paralelo, y filtra por `playlist.owner.id === me.id`. `getAllMyPlaylists` pagina de a 10 (máximo por página tras la migración de feb-2026) hasta agotar `next` o un techo de 200 — **no** se puede filtrar antes de traer todo: una playlist propia puede caer en cualquier página, no solo la primera (confirmado con esta cuenta: 8 propias de 39 totales, la mayoría fuera de la primera página de 10). Si `GET /me` falla, se degrada a mostrar todas sin filtrar en vez de vaciar la sidebar.
- **Panel principal** (`_cargarListaActual`): según la vista activa, pide `GET /me/tracks`, `GET /me/albums` o `GET /playlists/{id}/items` — este último **no** es `/playlists/{id}/tracks` (removido en esa misma migración de Spotify, ver [[Módulo Spotify#Restricciones de la plataforma (development mode)]]).
>
> [!bug] `/playlists/{id}/items` devuelve `403` para playlists que la sesión no es dueña (2026-08-24, plataforma, no bug propio)
> Confirmado pegándole directo a la Web API con las dos playlists que fallaban: `GET /playlists/{id}` (metadata) responde `200` sin problema, pero `GET /playlists/{id}/items` responde `403 {"error":{"status":403,"message":"Forbidden"}}` — **específicamente** para playlists que la cuenta conectada sigue pero no creó (ej. una playlist pública de otro usuario). La misma llamada contra una playlist propia (dueño = la cuenta conectada) devuelve `200` con el tracklist completo. No está documentado en ningún changelog de Spotify — es consistente con el resto de la migración de feb-2026 (restringir cosecha de datos vía apps de terceros), pero esta restricción específica (ownership de la playlist, no el tipo track/álbum/artista) no aparece en ningún lado escrito, solo se detectó probando en vivo. **No hay workaround**: ningún otro endpoint expone el tracklist completo de una playlist ajena para una app en development mode. Knefy ya lo absorbe sin cambios: `KnefyServices.getPlaylistItems` cae al sentinel `null` en cualquier error (mismo patrón que el resto de los servicios), y `_cargarListaActual` lo traduce a lista vacía ("Vacío") sin ningún cartel — el mismo comportamiento por el que se optó para cualquier otro fallo, así que esta restricción de plataforma quedó cubierta gratis por el diseño original, no necesitó un caso especial.
- **Filas de track** (`_crearFilaTrack`): índice, título, artista, duración y un botón `+` para encolar (`addToQueue`) sin interrumpir la reproducción actual. Click en la fila reproduce — si viene de una playlist, con `context_uri` + `offset` para seguir reproduciendo el resto de la lista después.
- **Filas "de contexto"** (`_crearFilaContexto`, álbumes/artistas/playlists de un resultado de búsqueda): reproducen el contexto entero (`context_uri`). **Sin drill-down** a la lista de temas de un álbum/artista en esta v1 — click reproduce directo, no abre su tracklist.
- **Búsqueda** (`_bindBusqueda`, debounce 350ms): `GET /search?type=track,album,artist,playlist&limit=10` (máximo permitido tras la migración) — resultados agrupados por sección (CANCIONES/ÁLBUMES/ARTISTAS/PLAYLISTS) en el mismo panel principal, filtrando entradas `null` del array de `playlists.items` (quirk conocido de esa respuesta).

## Ícono

`sources/appIcon/knefy.svg` es la única excepción de forma entre los íconos de `appIcon/`: el resto son `fill="currentColor"` (silueta sólida), este es `fill="none" stroke="currentColor"` (anillo + punto central, un disco de vinilo). Funciona igual como máscara CSS (`aplicarIconoImagen`, ver [[Frontend Model Services Utils]]) porque `mask-image` usa el alfa/luminancia de lo renderizado, no el atributo `fill` en sí — un trazo opaco sirve exactamente igual que un relleno opaco.

## Cambio fuera de KneOS

`public/js/main.js` (el bootstrap Three.js de la escena 3D) crea el `<iframe>` que carga `/KneOS/index.html` sin atributo `allow` — se le agregó `iframe.allow = 'encrypted-media; autoplay'`, requisito del Web Playback SDK (EME) que siendo same-origin probablemente hubiera andado igual, pero es una línea gratis de seguro contra que Chrome deniegue el permiso dentro del iframe.

## Registros

Los 5 puntos de siempre (ver plantilla en [[KPaint]]): `model/iconSrc.js` (`knefy` → `Knefy.js`), `model/defaultFiles.js` (`espacio67`, "Knefy"), `model/filesUndeletable.js` (`"knefy"`), `utils/formato.js` (`TIPOS.knefy = "Aplicación"`), `styles/main.css` (`@import 'apps/knefy.css'`). **No** está en el submenú "Nuevo" de `Folder`/`ContextMenuManager` — es una app de sistema, no un documento que se pueda crear en blanco (mismo criterio que [[Config]]/[[Camera]]).

## Estilos

`styles/apps/knefy.css` — todo con prefijo `knefy*`, monocromático sin excepciones (a diferencia de [[Config]], acá no hay ningún valor "real" que mostrar aparte del propio chrome). Layout de franjas: barra superior (buscador + toggle de cola) → cuerpo (`display:flex`, sidebar de biblioteca + lista principal) con el panel de cola flotando encima en posición absoluta (`.knefyQueuePanel`, `transform: translateX(100%/0)`, no reserva espacio cerrado) → reproductor fijo abajo (portada bitmap + info + transporte + progreso + volumen). `.knefyCover` (`<canvas>`, 2026-08-24): `image-rendering:pixelated` ya es global (`base.css`), no se redeclara; `display:block` evita el hueco de línea base que un canvas inline arrastra por default dentro del flex centrado de `.knefyCoverWrap`. Filas de lista (`.knefyRow`) invierten `background`/`color` en hover, mismo criterio que `.kpaintTool`. `.knefyVolumeInput` es un `<input type="range">` nativo sin reestilar el thumb — es un control de sistema, no una barra de progreso propia como `.knefyProgressTrack`.

## Consumido por / consume

Servicio frontend `KnefyServices` (`public/KneOS/js/services/KnefyServices.js`) — proxy hacia la Web API (`/spotifyRoutes/api/*`) más vínculo de cuenta (`getAccessToken`/`disconnect`), ver [[Módulo Spotify]]. `utils/spotifySdk.js` (carga perezosa del script del SDK, singleton) y `utils/dither.js` (portada bitmap, ver [[Frontend Model Services Utils#Utils (`public/KneOS/js/utils/`)]]) son utilidades puras sin estado propio.

## Verificación end-to-end

Probado contra una cuenta Premium real ya vinculada, firmando a mano un JWT de sesión para su `pc_id` (mismo secreto que `middlewares/auth.js`) y automatizando con Playwright + `channel: 'chrome'` (no el Chromium headless que trae Playwright por default: no soporta Widevine/EME, el SDK tira `initialization_error` con "No supported keysystem was found" — limitación del entorno de test, no del código). Con Chrome real: login ya vinculado → `ready` → reproducir un track de una playlist propia → `player_state_changed` llega, la portada se ditheriza en vivo. El algoritmo de `utils/dither.js` en sí se verificó aparte con una página HTML de prueba standalone (gradiente radial sintético, sin depender del SDK ni de la cuenta) — confirmó el patrón de dithering de 4 niveles + paleta verde antes de integrarlo a Knefy.
