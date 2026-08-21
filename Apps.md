---
tags:
  - portfolio/kneos
  - apps
---

# Apps

⬅️ Volver a [[KneOS Portfolio]]

Las 22 apps de escritorio de KneOS (`public/KneOS/js/apps/`). Todas extienden [[File]] y sobreescriben `_crearContenido()`; su tipo/extensión se resuelve vía `iconSrc.js` (ver [[Frontend Model Services Utils#Model]]).

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
| [[Camera]] | `camera` | Cámara de fotos monocromática: captura de la webcam recortada a 64×64 con dither Floyd–Steinberg a 4 niveles (efecto bitmap estilo Game Boy Camera, visible en vivo en el visor), paleta armada en runtime entre `--primary-background`/`--primary-color`; abre en `ViewWindow`, flujo Disparar → Guardar (crea un `img` nuevo) o Tomar otra | ninguna |
| [[ImgFile]] | `img` | Visor de una foto de Camera: array de 4096 bytes de gris persistido en `camera_photos` traducido a un `<canvas>` de 64×64 con `putImageData`, mezclando cada gris con el `--primary-color` vigente (repinta con el tema actual, no el de captura); solo se crea desde `Camera._guardarFoto`, no hay forma de crear un `img` en blanco | ninguna |
| [[Config]] | `config` | Panel de configuración del sistema — por ahora, un único ajuste: elegir el color de toda la interfaz entre 9 opciones (grilla de swatches, repinta al instante vía `--primary-*`, persistido por sesión) | ninguna |
| [[KPaint]] | `kp` | Editor de dibujo en pixel art: `<canvas>` de 64×64 pintado a mano con una paleta fija de 8 colores (negro + degradé hacia `--primary-color` vigente, un índice de paleta por píxel); `Ctrl+Z` con historial propio (tope 20), guardado manual; se crea en blanco desde el menú "Nuevo" (igual que `txt`/`fld`) | ninguna |
| [[KneMap]] | `map` | Puerto de MapSCII: mapa mundial navegable como grilla de texto ASCII verde monocromático (sin Braille/color ANSI real); tiles de OpenFreeMap pedidos directo del navegador; flechas/`hjkl` paneán, `a`/`z` zoomean, click/rueda/arrastrar por mouse; buscador de lugar en el escritorio (ciudad/dirección/provincia/país, geocoding real vía Nominatim, ver [[Módulo Map]]); corre igual en `Window` que dentro de [[Kmd]] (`map`) | `@mapbox/vector-tile` + `pbf` + `earcut` + `rbush` + `bresenham` (motor vendorizado, ver nota) |

[[DesktopGrid y DesktopFolder|DesktopFolder]] (el "Escritorio" navegable) extiende `Folder`, pero se documenta junto al resto del core por ser infraestructura, no una app de usuario.

> [!info] Únicas apps con ícono de escritorio por defecto (`defaultFiles.js`)
> No las 18 aparecen automáticamente en un escritorio nuevo: `defaultFiles.js` solo sembraba Doom/Terminal/Kfruit/Kne, después Maxwell (2026-07-30), RecycleBin (2026-07-31), Calculator, User (ex Contacts, 2026-08-04; renombrado 2026-08-07 — nombre en escritorio "Usuario"), BlackJack (2026-08-11), Hangman (2026-08-12, ex Ahorcado), FlipCoin (2026-08-12), Kdle (2026-08-12, renombrada de Wordle), CarRace (2026-08-13, ex CarreraAuto; archivos/tabla renombrados a inglés el mismo día, ver [[Deuda Técnica#Nombres en español traducidos a inglés (2026-08-13)]]) Tetris (2026-08-13), KneChat (2026-08-18), Camera y [[Config]] (2026-08-19), [[KPaint]] (2026-08-20) y ahora también [[KneMap]] (2026-08-21) — `TxtFile`/`Folder` no están ahí porque se crean a demanda desde el menú "Nuevo" (ver [[Folder#Funciones clave|Folder._abrirSubMenuCrear]] y el equivalente en `ContextMenuManager`). Igual que las otras apps "fijas" (Doom/Kmd/Kfruit/KneAI/Maxwell/RecycleBin/Calculator), User/BlackJack/Hangman/FlipCoin/Kdle/CarRace/Tetris/KneChat/Config/KneMap **no** están en esos menús "Nuevo" y además están en `filesUndeletable` (no tendría sentido poder eliminarlas ni recrearlas). `defaultFiles` solo se usa si `IconServices.getIcons()` devuelve **cero** filas (escritorio nunca inicializado) — en un escritorio con datos ya persistidos, agregar una entrada nueva ahí no hace aparecer el ícono retroactivamente.
>
> **`KPaint` (`kp`) es la excepción a "fija = undeletable + fuera de 'Nuevo'"**: se agregó a `defaultFiles` (2026-08-20, `espacio65`, nombre "KPaint") para que un escritorio nuevo ya tenga un lienzo a mano, pero **sigue** siendo un archivo de usuario común — no se sumó a `filesUndeletable`, y el submenú "Nuevo" conserva su entrada para poder crear más. Es un lienzo en blanco precargado, no una app-herramienta singleton como Calculator/Config.
>
> **`ImgFile` (`img`) es una tercera categoría** (2026-08-19), distinta de las dos anteriores: no es un ícono fijo por defecto (no está en `defaultFiles`/`filesUndeletable` — una foto guardada se puede borrar como cualquier archivo de usuario) pero tampoco se crea a demanda desde el menú "Nuevo" como `txt`/`fld` (no tendría sentido "Nueva imagen en blanco"). La única forma de que exista un ícono `img` es que [[Camera]] lo cree al guardar una foto.
>
> **`KPaint` (`kp`, 2026-08-20) vuelve a la primera categoría** (como `txt`/`fld`): a diferencia de `img`, un lienzo en blanco sí tiene sentido, así que `kp` está en el menú "Nuevo" de `Folder`/`ContextMenuManager` y se crea a demanda — no está en `defaultFiles` (no aparece solo en un escritorio nuevo) ni en `filesUndeletable`.
