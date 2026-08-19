---
tags:
  - portfolio/kneos
  - apps
---

# Apps

⬅️ Volver a [[KneOS Portfolio]]

Las 17 apps de escritorio de KneOS (`public/KneOS/js/apps/`). Todas extienden [[File]] y sobreescriben `_crearContenido()`; su tipo/extensión se resuelve vía `iconSrc.js` (ver [[Frontend Model Services Utils#Model]]).

| App | Extensión | Qué hace | Librería externa |
|---|---|---|---|
| [[TxtFile]] | `txt` | Editor de texto enriquecido (negrita/cursiva/subrayado, guardado manual o Ctrl+G) | ninguna (`contentEditable` nativo) |
| [[Folder]] | `fld` | Explorador de archivos estilo Windows (sidebar, breadcrumb, buscador, vistas, orden, drag&drop) | ninguna |
| [[KneAI]] | `ai` | Chat con IA estilo ChatGPT, historial de conversaciones, auto-titulado | ninguna (delega el LLM al backend, ver [[Módulo Groq]]) |
| [[Doom]] | `exe` | Emulador de DOOM vía DOSBox compilado a WASM | **js-dos** |
| [[Kmd]] | `kmd` | Terminal estilo CMD sobre el sistema de archivos real (`dir`/`cd`/`mkdir`/`touch`/`rmdir`/`move`/`ren`/`del`/`type`/`echo`/`tree`/`kneai`/`curl`/etc.) | ninguna |
| [[Kfruit]] | `kfruit` | Juego de fusión de frutas estilo Suika Game, con física real | **planck** (Box2D portado a JS) |
| [[Maxwell]] | `maxwell` | Visor 3D: carga un `.glb` y lo muestra girando sobre su eje | **Three.js** (import map propio, ver nota en [[Escena 3D]]) |
| [[RecycleBin]] | `recyclebin` | Papelera de reciclaje: lista lo mandado a la papelera (soft delete), con Restaurar/Eliminar definitivamente/Vaciar papelera | ninguna |
| Calculator (`apps/Calculator.js`, sin nota propia todavía) | `calc` | Calculadora simple, sin precedencia de operadores (evalúa de izquierda a derecha) | ninguna |
| [[User]] | `contacts` | Ficha estilo ID card retro con links reales a redes (GitHub, LinkedIn, Instagram, X), cada uno con ícono monocromo y abre en pestaña nueva del navegador | ninguna |
| [[BlackJack]] | `blackjack` | Port del BlackJack de consola en Java original (mazo/reparto/pedir-plantarse-doblar-dividir, mismo pago); menúes de `Scanner` pasados a botones, sin persistencia en BD | ninguna |
| [[Hangman]] | `ahorcado` | Port del Ahorcado de consola en Java original (7 vidas, 8 escenas ASCII del muñeco); `Scanner` de "pedí una letra" pasado a un abecedario en pantalla + teclado físico, letra usada se tacha; catálogo de palabras (cualquier largo) en la tabla `words` (ex `palabras`), compartida con Kdle desde 2026-08-13 | ninguna |
| [[FlipCoin]] | `flipcoin` | Port de la app de consola "Flip-Coin" en Java original (sorteo 50/50 + animación de giro); el archivo de texto donde grababa cada tirada pasó a la tabla `flipcoin_results` (ex `flipcoin_resultados`) en BD (crece con cada partida, a diferencia del catálogo fijo de Ahorcado) | ninguna |
| [[Kdle]] | `kdle` | Port del Wordle de consola en Java original, renombrado a Kdle (5 intentos, palabra de 5 letras, se tipea directo sobre la grilla con el teclado físico y la fila se confirma sola al completar, validando contra un diccionario en español que la palabra exista; grilla coloreada verde sólido/semitransparente/tachada, leyenda a la izquierda); palabras de 5 letras filtradas de la tabla `words`, compartida con Ahorcado desde 2026-08-13 | ninguna |
| [[CarRace]] | `carreraauto` | Port del CarreraAuto de consola en Java original (4 autos que avanzan al azar hasta la meta, apuesta libre a uno de ellos antes de largar); el archivo donde el original grababa cada ganador para calcular las cuotas pasó a la tabla `car_race_results` (ex `carreraauto_resultados`) en BD (crece con cada carrera, mismo criterio que FlipCoin); cuota dinámica = total de carreras / victorias del auto, capturada antes de correr cada carrera; pista a 100% del ancho de la ventana, autos identificados como "Auto 1"–"Auto 4" | ninguna |
| [[Tetris]] | `tetris` | Port del Tetris de consola en Java original (tablero 24×12, 7 piezas con sus rotaciones, mismo puntaje +10/pieza +100/línea); niveles activados (código muerto en el original, nunca se llamaba) subiendo cada 10 líneas; menú/teclas configurables/leaderboard estilo Kfruit (`tetris_keybinds`/`tetris_score`, agregados nuevos — el original no tenía ninguno de los dos); animación de línea con flash CSS | ninguna |
| [[KneChat]] | `chat` | Chat en vivo por WebSocket entre las sesiones conectadas: sala pública `#GENERAL` + DMs 1-a-1, historial en Postgres; alias elegido al entrar (no una cuenta), colgado de la sesión anónima existente | **ws** (backend) / `WebSocket` nativo (cliente) |

[[DesktopGrid y DesktopFolder|DesktopFolder]] (el "Escritorio" navegable) extiende `Folder`, pero se documenta junto al resto del core por ser infraestructura, no una app de usuario.

> [!info] Únicas apps con ícono de escritorio por defecto (`defaultFiles.js`)
> No las 16 aparecen automáticamente en un escritorio nuevo: `defaultFiles.js` solo sembraba Doom/Terminal/Kfruit/Kne, después Maxwell (2026-07-30), RecycleBin (2026-07-31), Calculator, User (ex Contacts, 2026-08-04; renombrado 2026-08-07 — nombre en escritorio "Usuario"), BlackJack (2026-08-11), Hangman (2026-08-12, ex Ahorcado), FlipCoin (2026-08-12), Kdle (2026-08-12, renombrada de Wordle), CarRace (2026-08-13, ex CarreraAuto; archivos/tabla renombrados a inglés el mismo día, ver [[Deuda Técnica#Nombres en español traducidos a inglés (2026-08-13)]]) Tetris (2026-08-13) y ahora también KneChat (2026-08-18) — `TxtFile`/`Folder` no están ahí porque se crean a demanda desde el menú "Nuevo" (ver [[Folder#Funciones clave|Folder._abrirSubMenuCrear]] y el equivalente en `ContextMenuManager`). Igual que las otras apps "fijas" (Doom/Kmd/Kfruit/KneAI/Maxwell/RecycleBin/Calculator), User/BlackJack/Hangman/FlipCoin/Kdle/CarRace/Tetris/KneChat **no** están en esos menús "Nuevo" y además están en `filesUndeletable` (no tendría sentido poder eliminarlas ni recrearlas). `defaultFiles` solo se usa si `IconServices.getIcons()` devuelve **cero** filas (escritorio nunca inicializado) — en un escritorio con datos ya persistidos, agregar una entrada nueva ahí no hace aparecer el ícono retroactivamente.
