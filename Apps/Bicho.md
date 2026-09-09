---
tags:
  - portfolio/kneos
  - apps
---

# Bicho

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Pet.js` (clase `Pet`) — extiende [[File]]. Extensión `"pet"`, ícono propio `sources/appIcon/pet.svg` (glifo de ojo, `fill="currentColor"`, viewBox 24×24). Agregada 2026-09-09.

> [!abstract] Qué hace
> Una mascota de escritorio tipo Clippy: un ojo glitcheado hecho de scanlines que deambula libremente sobre `#aplicaciones`, se puede arrastrar, y responde preguntas sobre Manuel y el proyecto vía [[Módulo Groq]], alimentado con **todas** las fuentes de conocimiento del repo a la vez (archivo curado + README/ARCHITECTURE/PRODUCT.md + este mismo vault + el CV + el estado vivo del escritorio). Doble propósito: cara del sistema para un visitante casual, y guía activa para un reclutador que no sabe por dónde arrancar.

## No vive en una ventana (el problema de diseño real de esta app)

Toda app extiende `File`, y `File` compone un [[Window y Taskbar|Window]] en su constructor — pero Bicho es un elemento libre, no algo que se abre en una ventana. La solución no fue anular `this.window`: un `Window` que nunca se abre es 100% inerte (todos sus métodos hacen early-return sobre `this._ventanaEl`, que solo se puebla en `abrir()`), así que dejarlo muerto y sin usar es gratis, y anularlo hubiera obligado a auditar cada lugar de `File` que lo desreferencia (`renombrar()`, `fijarEnTaskbar()`).

En cambio, se sobreescribe el contrato de ventana para montar/desmontar el sprite libre:

```js
toggleVentana() { this._mounted ? this._unmount() : this._mount(); }
abrirVentana()  { this._mount(); }
cerrarVentana() { this._unmount(); }
minimizar()     { this._unmount(); }
restaurar()     { this._mount(); }
```

`abrirVentana()` tiene que overridearse aparte de `toggleVentana()` porque `ContextMenuManager._setIconMenu` (opción "Abrir" del menú contextual del ícono) la llama **directo**, sin pasar por `toggleVentana` — sin este segundo override, "Abrir" crearía una `Window` real vacía con botón en la taskbar (mismo motivo por el que [[KPaint]] overridea los dos métodos). Doble click en el ícono prende/apaga el bicho; nunca aparece en la taskbar ni tiene barra de título.

Singleton estático (`Pet._active`, mismo idioma que `ContextMenu._menusAbiertos`/`Clock._calendariosAbiertos`): montar uno desmonta cualquier otro vivo.

## Dónde vive y en qué capa

`#petSprite` (canvas) y `#petBubble` (bocadillo) son **hermanos** de `#aplicaciones`, hijos directos de `#escritorio` — no hijos de `#aplicaciones`, porque ese contenedor tiene `enableMultiSelect` (un mousedown ahí dispara la goma de selección) y `DesktopGrid.buscarVacio()` recorre sus hijos como celdas de la grilla. Los bounds del deambular sí salen de `#aplicaciones.offsetWidth/offsetHeight` (inmune a transforms — KneOS corre en un iframe con `matrix3d` en la página padre), así nunca pisa la barra de tareas aunque el sprite sea hijo de `#escritorio`.

`z-index: 9996` (sprite) / `9997` (bocadillo): por encima de toda `.ventana` (arrancan en 1000, suben con `Window.traerAlFrente()`), **por debajo** de los overlays CRT del escritorio (`::before` scanlines 9998, `::after` flicker 9999) — deliberado: un bicho "hecho de scanlines" tiene que recibir el CRT encima, no flotar limpio por arriba. Riesgo teórico documentado en el código: tras ~9000 enfoques de ventana un `.ventana` superaría los 9000, pero Bicho no tiene esa clase y no contamina ese `max()` — no se le puso techo a `traerAlFrente()` por un caso imposible en la práctica.

## Render: canvas 2D, no SVG

`utils/petSprite.js` — `drawEye(ctx, state, colors)`, toda la forma en una sola función (cambiar de criatura es reemplazarla, no tocar `Pet.js`). Grilla 24×16 celdas de 3px → sprite 72×48. El contorno y el iris son elipses vía `celdasElipse` (reusa [[Frontend Model Services Utils#Utils|utils/pixelShapes.js]], sin rasterizador propio). El relleno del interior solo pinta filas pares con una fase que corre (`_scanPhase`, mismo lenguaje que `@keyframes scanlineShift` de `desktop.css`) en `--primary-dim`; contorno e iris van sólidos en `--primary-color` — así la forma se sigue leyendo por encima del rayado.

Se eligió canvas sobre SVG-con-`currentColor` porque la expresión cambia cada frame (parpadeo, mirada, glitch) — mutar el DOM 60 veces por segundo sería peor que repintar un canvas ya escalado a DPR. Colores cacheados vía `leerColorCSS()` ([[Frontend Model Services Utils|model/themeColors.js]], la misma que usa [[Camera]]) y refrescados **solo** en el evento `kneos:theme-color`, nunca por frame (`getComputedStyle` a 60fps fuerza recálculo de estilo). Nada de `text-shadow` en ningún lado — el glow sale del contraste entre el relleno dim y el contorno sólido.

## Movimiento

Drag vía `interact.js` (`window.interact`, ya global desde `KNEOS.js`, mismo idioma que `Window._hacerMovible`) — mouse y touch gratis, sin pointer events a mano. Un solo dueño del `transform`: `interact` solo acumula sobre `this._x/_y` en el listener `move`, el loop rAF es quien escribe el DOM.

Wander: máquina de dos estados (mover hacia un punto al azar / pausar 1.5–4s) a 35px/s, con la dirección del paso alimentando la mirada del ojo (`_gazeX/_gazeY`) y un bob senoidal aplicado solo en el render (nunca acumulado en `_x/_y`, para no hacer drift). El bocadillo abierto congela el wander — texto que se mueve es ilegible, y ahorra reposicionar el bocadillo por frame.

Loop con las guardas del patrón [[Maxwell]]: `if (!this._el?.isConnected)` corta el `requestAnimationFrame` en vez de seguir pidiendo frames para un elemento fuera del documento; `if (document.hidden)` pausa y resetea el reloj (si no, el primer `dt` al volver de background sería de varios segundos y el bicho pegaría un salto). `_unmount()` hace `cancelAnimationFrame` + `interact(el).unset()` (igual que `Window.cerrar()`) + saca los listeners de tema/documento — nada queda corriendo tras cerrarlo.

> [!bug] El glitch del bocadillo pisaba su propio posicionamiento (encontrado y resuelto en la verificación con Playwright)
> `.petBubble--glitchIn` reusaba `@keyframes pixelGlitch` (ya definido en `desktop.css`, huérfano hasta esta app) para que el bocadillo se materialice con el mismo lenguaje visual que el sprite — pero ese keyframe anima `transform`, la misma propiedad que `Pet._positionBubble()` escribe inline para posicionar el bocadillo, y en CSS una animación gana sobre un estilo inline mientras corre. El bocadillo se abría teletransportado a `translate(0,0)` durante los 0.4s de la animación. Fix: el glitch se aplica a un `<div class="petBubbleInner">` hijo (que solo lleva el fondo/borde/padding/contenido), dejando `.petBubble` — el elemento que el JS posiciona — sin ninguna animación de `transform` propia. Mismo principio que ya se había documentado para el sprite (el glitch del ojo tampoco se aplica al elemento posicionado), pero se pasó por alto al escribir el bocadillo la primera vez.

## Conocimiento: las 5 fuentes combinadas

`utils/petContext.js` — `buildContext(question, { vaultIndex, docsIndex })` ensambla el system prompt por pregunta, en capas:

| Capa | Fuente | Presupuesto |
|---|---|---|
| Persona + guardrails + protocolo `#run` | `model/petKnowledge.js` (curado a mano) | ~250 tok, fijo |
| Estado vivo del escritorio | `utils/desktopSnapshot.js` | ~120 tok, fijo |
| Identidad (CV + links de contacto) | `model/curriculumData.js` + `CONTACT_LINKS` (exportada desde [[User]]) | ~200 tok, fijo |
| Notas del cerebro + docs del repo | `vault.json` + `docs.json` vía [[VaultServices]] | ≤700 tok, retrieval top 3 |

**README/ARCHITECTURE/PRODUCT.md → `docs.json`, sin tocar `vault.json`**: `scripts/buildVault.js` reusa entera su función `buildNotes()` (un `.md` de la raíz del repo es una nota más para ese pipeline) y emite un segundo archivo con la misma forma de nota. Se decidió NO meter estos tres docs en `vault.json` porque ese JSON es el modelo de datos de punta a punta de [[KneOsBrain]] (árbol de carpetas, chips de tags, el grafo global, backlinks) — tres notas sintéticas ahí le agregarían nodos huérfanos al grafo y ~90KB a cada apertura de esa app, para algo que solo usa la mascota. `VaultServices` ahora acepta la URL como parámetro opcional del constructor (`sourceUrl = "sources/vault/vault.json"`, dos líneas de cambio) — KneOsBrain sigue construyendo sin argumentos, cero impacto; Bicho hace `new VaultServices("sources/vault/docs.json")` y hereda índices/memoización/`search()` gratis. `docs.json` gitignoreado igual que `vault.json` (mismo prefijo `public/KneOS/sources/vault/`). Carga perezosa de ambos índices, recién en la primera pregunta — juntos pesan ~900KB y no deben tocar el arranque de KneOS.

**Retrieval sin reescribir el buscador**: `VaultServices.search(query)` busca la query como **un único substring**, no tokeniza — así que `petContext.js` tokeniza la pregunta (términos de ≥4 chars sin stopwords, top 4 por longitud como proxy pobre de IDF), llama `search(term)` una vez por término sobre los dos índices, y rankea cada nota por cantidad de **términos distintos** que matchearon (no por cantidad total de matches). El snippet de 40 chars que ya da `search()` es muy corto para un LLM (pensado para la sidebar de KneOsBrain) — se recorta una ventana propia de hasta 800 chars alrededor del primer match, sin tocar esa constante compartida.

**Veracidad estricta** (decisión explícita de Manuel, alineada con `PRODUCT.md` — "solo evidencia real, cero contenido fabricado"): el prompt ordena que si el dato no está en el contexto recuperado, el bicho diga que no sabe y sugiera dónde mirar. Sin tabla de respuestas fijas para preguntas frecuentes — cada pregunta pasa por el modelo con el contexto que se le arma. Máximo dos oraciones, texto plano sin markdown.

## Acciones (`#run`)

Mismo protocolo que [[Kmd]] (`_cmdKneAi`): si el pedido se resuelve con una acción, el modelo contesta en una sola línea (`#run abrir <nombre>` / `#run nota <titulo>`) y el cliente la intercepta con una regex antes de mostrarla, en vez de mostrar texto.

- `abrir <nombre>` — busca por nombre entre los archivos raíz del escritorio; si no matchea nada, carga el contenido de la carpeta "Juegos" (que no está en `archivosAbiertos` hasta que se abre una vez) y reintenta ahí — mismo fallback que usa Kmd para nombres que no son raíz.
- `nota <titulo>` — resuelve el título contra el índice del vault (`resolveTarget`) y llama a `KneOsBrain.openNote(path)` (nuevo método público, agregado junto con esta app: antes `_loadVault()` era fire-and-forget al abrir la ventana, ahora guarda la promesa en `this._loadVaultPromise` para que `openNote` pueda esperarla — abre la ventana si hace falta, espera a que el vault esté cargado, y navega directo).

Sin tabla de acciones nueva por comando: el vocabulario es chico a propósito (cada verbo es una superficie de falla).

## Presupuesto de llamadas a Groq

`groqLimiter` = 10 req/min por `pc_id`, **compartido** con [[KneAI]], [[Kmd]] y [[TxtFile]] (ver `middlewares/rateLimiters.js`). Bicho nunca toca ese límite compartido — cede siempre:

- Deambular, saludo inicial, parpadeo, glitch, click y drag: **cero llamadas**, todo local (`model/petKnowledge.js`).
- Solo una pregunta tipeada explícitamente en el bocadillo dispara una llamada.
- Auto-throttle en el cliente, más estricto que el server: máximo 3 llamadas por minuto y 15s entre llamadas, dejando ≥7/min para el resto del sistema.
- `groq.ask()` nunca lanza, devuelve `null` en cualquier error (ver [[Módulo Groq]]) — eso se traduce en cerrar el bocadillo sin ningún indicador, la regla de UX del proyecto (ver [[Reglas]]): sin cartel, sin borde rojo, sin shake.

## Presencia y apagado

App con ícono (`espacio70`, última celda libre de la grilla — `espacio69` era [[Curriculum]], la última ocupada). Se apaga con click derecho sobre el sprite → "Ocultar" (`ContextMenu` propio de la instancia, mismo componente que usa el resto del sistema) — **no** desde [[Config]], que hoy es un panel de un solo propósito (el selector de color) y agregarle un toggle hubiera sido más código del que la feature vale. Cadencia elegida a propósito: callado por default, un único saludo a los ~20s del montaje que se auto-cierra a los 6s, y después solo habla si se le habla — el modo de falla clásico de un Clippy es el chatter no solicitado.

## Registro de la app

Mismo patrón de 4 lugares que cualquier app nueva (ver [[Knefy]]):
- `model/iconSrc.js` → `pet: { css: "url(sources/appIcon/pet.svg)", load: () => import("../apps/Pet.js") }`
- `model/defaultFiles.js` → `{ desktop_place: "espacio70", ext: "pet", name: "Bicho" }`
- `model/filesUndeletable.js` → `"pet"` agregado al Set
- `utils/formato.js` → `pet: "Aplicación"` en `TIPOS`

`FileType.AI`. Sin `Window` real (ver más arriba). Sin backend nuevo: memoria de conversación en memoria (`this._history`, tope 6, igual que `Kmd._aiHistory`), sin persistir nada — se pierde al recargar la página, igual que la posición y el on/off.

## Archivos nuevos

| Archivo | Rol |
|---|---|
| `apps/Pet.js` | La app: montaje/desmontaje, sprite, wander, drag, bocadillo, llamada a Groq, dispatch de `#run`. |
| `model/petKnowledge.js` | Curado a mano: persona/tono/reglas del prompt, saludo, frases idle. |
| `utils/petSprite.js` | `drawEye()`, el rasterizado puro del ojo. |
| `utils/desktopSnapshot.js` | `describeDesktop()`, serializa el estado vivo (hora, tema, ventanas abiertas, catálogo de apps) para el prompt. |
| `utils/petContext.js` | Tokenizado, retrieval sobre `VaultServices.search()`, ensamblado del prompt. |
| `styles/apps/pet.css` | Sprite + bocadillo. |
| `sources/appIcon/pet.svg` | Ícono, un solo `<path fill="currentColor">`. |
| `public/KneOS/sources/vault/docs.json` | Generado por `buildVault.js` (gitignoreado). |

## Archivos modificados

`services/VaultServices.js` (URL parametrizable + `export` a `foldText`), `apps/User.js` (`CONTACT_LINKS` exportada), `apps/KneOsBrain.js` (`openNote()` + `_loadVaultPromise`), `scripts/buildVault.js` (emite `docs.json`), `styles/main.css` (`@import`), los 4 archivos de registro de arriba, `.gitignore` ya cubría `docs.json` (el prefijo `public/KneOS/sources/vault/` completo ya estaba ignorado).

## Testing / Verificado

Con Playwright contra la app real (no una prueba aislada): doble... mejor dicho, **click simple** prende/apaga el bicho (las apps de KneOS abren con un solo click en el ícono, no doble click — confirmado leyendo `DesktopManager._crearContenedorIcono`, el listener está en `"click"`); "Abrir" del menú contextual del ícono prende el bicho sin crear una ventana vacía ni un botón de taskbar (el test real del override de `abrirVentana`); deambula solo dentro del área de íconos y nunca pisa la barra de tareas (confirmado con posiciones tomadas antes/después de un wait ≥6s, ya que el primer wander tiene una pausa aleatoria de 1.5–4s antes de arrancar); se arrastra; el canvas repinta al instante al cambiar el color del tema desde Config (verificado leyendo el primer píxel opaco del canvas antes/después de `applyThemeColor('rojo')`); click derecho abre "Ocultar"; click simple abre el bocadillo con input, sin salirse del escritorio. Pipeline de conocimiento verificado end-to-end sin gastar cuota de Groq: `vault.json` (62 notas) y `docs.json` (3 docs) cargan y se indexan, y `buildContext()` con preguntas reales recupera la nota correcta ("qué es KneOsBrain" trae la nota de KneOsBrain; una pregunta que solo responde ARCHITECTURE.md trae el fragmento correcto de ahí) y siempre incluye los datos fijos del CV. Acciones `#run` probadas invocando `_runOpen`/`_runNote`/`_maybeRunAction` directo (sin pasar por Groq): abre Kmd por nombre, navega KneOsBrain a una nota real por título, y un texto sin el prefijo `#run` no dispara ninguna acción. Cero errores de consola en todo el flujo.
