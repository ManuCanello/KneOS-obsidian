---
tags:
  - portfolio/kneos
  - apps
---

# KneOsBrain

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/KneOsBrain.js` — extiende [[File]]. Extensión `"brain"`, ícono propio `sources/appIcon/kneosbrain.svg` (grafo: 4 nodos + centro, `stroke="currentColor"` para las aristas + `fill="currentColor"` para los nodos, viewBox 24×24). Agregada 2026-08-25.

> [!abstract] Qué hace
> Expone **este mismo vault de Obsidian** (documentación de arquitectura/decisiones del proyecto) como una app más de KneOS — de solo lectura, el vault sigue siendo la fuente de verdad. Explorador de carpetas + búsqueda full-text + panel de tags a la izquierda; la nota renderizada o el grafo global de todas las notas (alternable con el toggle NOTA/GRAFO) al centro; outline + backlinks + grafo local (vecindario a 1 salto) de la nota abierta a la derecha. Detalle: esta misma nota es visible dentro de la propia app — el cerebro se documenta a sí mismo.

## De dónde salen las notas (build time, no runtime)

El vault vive fuera del repo y en producción el server no puede leer el disco de Manuel, así que las notas no se leen en vivo: **`scripts/buildVault.js`** (nuevo, Node puro, sin dependencias nuevas) corre como paso previo de `npm run build` (que a su vez corre en `postinstall`, o sea también en cada deploy) y:

1. Baja el tarball de `https://codeload.github.com/ManuCanello/KneOS-obsidian/tar.gz/refs/heads/main` (repo público) — o lee una carpeta local si `VAULT_LOCAL_PATH` está seteada (para ver cambios de docs sin pushear antes).
2. Lo descomprime con un parser de tar mínimo hecho a mano (headers de 512 bytes, soporta "long name" estilo GNU) — mismo criterio que `utils/dither.js`/`utils/pixelShapes.js`, resolver a mano en vez de sumar una dependencia para algo acotado.
3. Se queda solo con los `.md` fuera de `.obsidian/`, `.git/` y **`.claude/`** (el vault tiene su propia config/skills de Claude Code versionada — no son notas). Además excluye por nombre puntual `Deuda Técnica.md` (2026-08-25, a pedido explícito, `EXCLUDED_NOTES`): documenta hallazgos/decisiones internas para priorizar mejoras, no contenido pensado para navegarse desde la app. Los wikilinks de otras notas que apuntaban ahí (44, incluidos varios con `#Heading`) simplemente quedan apagados — mismo "sin cartel" que cualquier link a una nota que no existe, ver [[Reglas]].
4. Parsea cada nota: frontmatter (subset YAML), headings, tags (frontmatter + inline `#tag`), wikilinks `[[Nota|alias]]`/`[[Nota#Heading]]`, embeds `![[...]]` — resolviendo el target de cada uno por nombre de archivo (case-insensitive, primer match gana si hay ambigüedad, igual que Obsidian sin rutas únicas) e invirtiendo el grafo para calcular **backlinks**.
5. Emite un único `public/KneOS/sources/vault/vault.json` (gitignoreado, igual que `dist/`) con `{ generatedAt, notes: [...] }`.

**Resiliencia**: si el fetch/parseo falla y ya existe un `vault.json` de una corrida anterior, lo conserva (avisa por consola) — un deploy no debería romperse porque GitHub tosió. Si no hay ninguno previo, escribe uno vacío (la app se ve como un vault vacío, no como rota).

> [!info] Un solo JSON, no un archivo por nota
> 57 notas reales (después de sacar `.claude/` y `Deuda Técnica.md`): ~600KB crudos, bastante menos con el `compression()` que ya tiene `server.js`, y se paga recién cuando alguien abre la app (chunk propio vía el `import()` dinámico de `iconSrc.js`, igual que cualquier otra app pesada).

## Registro de la app

Mismo patrón de 4 lugares que cualquier app nueva (ver [[Knefy]]):
- `model/iconSrc.js` → `brain: { css: "...", load: () => import("../apps/KneOsBrain.js") }`
- `model/defaultFiles.js` → `{ desktop_place: "espacio68", ext: "brain", name: "KneOsBrain" }`
- `model/filesUndeletable.js` → `"brain"` agregado al Set
- `utils/formato.js` → `brain: "Aplicación"` en `TIPOS`

`FileType.PRODUCTIVITY` (es una app de notas). `Window` propia (no `ViewWindow`: sí tiene taskbar y es redimensionable) con `tamano: { width: 940, height: 620 }`, mismo patrón que [[Knefy]].

## Archivos nuevos

| Archivo | Rol |
|---|---|
| `scripts/buildVault.js` | Lo de arriba. |
| `services/VaultServices.js` | Carga `vault.json` una vez (memoiza la promesa) + índices derivados: por path, por carpeta, por tag, `resolveTarget(target)` (nombre de archivo → path), `search(query)` (full-text case/acento-insensible con snippet). No usa `apiFetch.js` — es un asset estático propio, no una ruta autenticada de la API. |
| `utils/obsidianMarkdown.js` | Renderer del subset de markdown que este vault usa, a DOM (`createElement`, nada de `innerHTML`) — ver detalle abajo. |
| `utils/forceGraph.js` | Layout force-directed + dibujo en `<canvas>` 2D, compartido por el grafo global y el local — ver detalle abajo. |
| `apps/KneOsBrain.js` | La app: layout de 3 columnas, navegación con historial propio, filtro de tags. |
| `styles/apps/kneosbrain.css` | Estilos — monocromático (`--primary-*`), `container-type: inline-size` para el responsive (ver más abajo). |

## Renderer de markdown (`utils/obsidianMarkdown.js`)

Parser de bloques línea por línea (headings, párrafos, code fences, tablas, listas/tareas, blockquotes/callouts, `---`) + un tokenizador inline (código/wikilinks-embeds/links/negrita/cursiva/tachado) que consume `text` de izquierda a derecha. No es CommonMark completo — alcanza con lo que las notas reales producen (verificado renderizando las 58 originales sin excepciones: 119 callouts, 31 tablas, 992 wikilinks, 0 fallos; re-verificado tras excluir `Deuda Técnica.md` — 57 notas, 0 fallos, 44 wikilinks hacia esa nota resueltos como apagados sin romper nada).

- **Wikilinks/embeds** se resuelven vía `vaultIndex.resolveTarget()` (no vía el `resolved` que ya trae cada link en el JSON — hace falta resolver también targets que aparecen recién al transcluir un embed, cuyo texto no pasó por el parseo de build time de la nota que lo referencia). Un link a una nota inexistente se pinta apagado (`--primary-dim`) y no hace nada al clickear — sin cartel, regla del proyecto.
- **Embeds** (`![[Nota]]`/`![[Nota#Heading]]`) transcluyen el body completo o solo la sección (recortada por heading, mismo criterio que Obsidian: hasta el próximo heading de nivel igual o menor). Guarda contra ciclos con un `embedStack` (Set de paths en la cadena actual) que se hereda a cada recursión — un embed roto o cíclico no muestra nada (`display:none`), mismo "sin cartel" que un link muerto.
- **Callouts** (`> [!bug] Título`) se marcan con una etiqueta de texto (`[BUG]`) en vez de un color propio — paleta monocromática, ver [[Reglas]]. El tipo vive en la primera línea del bloque **con contenido real**, no necesariamente `lines[0]` literal (un separador `>` vacío antes del callout, como el `[!bug]` anidado de [[Knefy]], corría el marcador a la segunda línea).

> [!bug] Todos los nodos del grafo global arrancaban superpuestos en un solo punto (2026-08-25, resuelto)
> El wrapper+canvas del grafo global se construye e inicializa (`ForceGraph.setData`) **antes** de montarse en el DOM (para no recrear el `ResizeObserver` cada vez que se alterna NOTA/GRAFO). Con el canvas todavía sin layout real, `clientWidth`/`clientHeight` daban 0, así que el círculo de arranque de cada nodo colapsaba al mismo punto (0,0) para los 58 nodos. Repulsión entre dos nodos exactamente superpuestos da un vector de dirección nulo (`dx=dy=0`), así que la simulación nunca los separaba — se veía un único punto verde para siempre, sin aristas visibles. Fix: `forceGraph.js` usa un tamaño de fallback (400×300) para calcular el círculo de arranque cuando el canvas todavía no tiene layout real; el tamaño real se recalcula solo en el próximo resize (disparado por `KneOsBrain._renderCenter` al montar el wrapper). Encontrado renderizando la app real con Playwright, no en una prueba aislada del componente.

> [!bug] Ciclo de embeds (A embebe a B, B embebe de vuelta a A) tiraba stack overflow (2026-08-25, resuelto)
> El guard contra ciclos (`embedStack`) se pasaba correctamente entre `renderEmbed`/`renderWikiOrEmbed`/`renderInlineToken`, pero las funciones de **bloque** (`renderParagraph`, `renderHeading`, `renderList`, `renderTable`) llamaban a `renderInlineInto` sin reenviarlo — el parámetro por default (`embedStack = new Set()`) creaba un Set nuevo y vacío en cada párrafo/heading/celda, perdiendo todo el historial acumulado. Un embed cíclico entraba en recursión infinita real (`RangeError: Maximum call stack size exceeded`), no solo un renderizado incorrecto. El vault real no usa `![[...]]` en ninguna nota (0 embeds en las 58), así que esto no se hubiera visto nunca en uso normal — encontrado con una prueba sintética armada a propósito para forzar el ciclo, después de notar que el guard nunca podía dispararse dado cómo estaban threaded los parámetros. Fix: las 4 funciones de bloque ahora reciben y reenvían `embedStack` explícitamente.

## Grafo (`utils/forceGraph.js`)

Look **ASCII** (2026-08-25, a pedido explícito, mismo lenguaje que el mapa de [[KneMap]]): nada de arcos/líneas vectoriales lisas. Cada nodo es un glifo monoespaciado según su cantidad de links — `.` (0-1), `o` (2-4), `O` (5-9), `@` (10+) — y cada arista es una fila de caracteres `-`/`|`/`\`/`/` elegidos según la pendiente del segmento (`edgeChar`), muestreados a pasos fijos entre los dos nodos. La nota abierta se marca envolviendo su glifo entre `<` `>` en vez de un aro vectorial. Fuente monoespaciada (`ui-monospace, Consolas, monospace`) a 24px los nodos / 16px las aristas (subida desde 16px/12px, a pedido explícito — "más grande"), mismo criterio que `.kneMapGrid`.

> [!info] Nodos más separados (2026-08-25, a pedido explícito)
> `REPULSION` 2600→7000 y `SPRING_LENGTH` 70→150 (distancia de reposo de cada arista) -- el resto de la física (gravedad, temperatura/cooling) no cambió, así que sigue asentándose rápido y estable, solo que en un radio bastante más amplio.

> [!info] Aristas atenuadas por default (2026-08-25, a pedido explícito)
> Con ~990 wikilinks entre 58 notas, mostrar todas las aristas a la vez a opacidad plena era una maraña ilegible al zoomear (confirmado en vivo con Playwright antes de este cambio). `_edgeAlpha(a, b)` (nuevo, separado de `_alphaFor` que sigue siendo solo para nodos) las deja casi invisibles por default (`0.08`) y solo ilumina (`0.85`) las que tocan el nodo bajo el mouse o la nota abierta (grafo local) — el resto de las aristas, aun con algo resaltado, quedan en `0.05`. Los nodos en sí no cambiaron: siempre visibles, son "la lista" del grafo; el ruido que molestaba era específicamente de las aristas.

Layout tipo **Fruchterman-Reingold** (repulsión O(n²) entre todo par de nodos — trivial para ~58 nodos, no hace falta Barnes-Hut — + resortes en las aristas + gravedad al centro), dibujado a mano en `<canvas>` 2D. El desplazamiento de cada nodo por frame está topeado por una "temperatura" que decae de forma monótona cada frame (`COOLING_RATE = 0.985`) — a diferencia de un sistema de velocidad+damping real, esto garantiza que la simulación converge sí o sí, nunca puede oscilar para siempre. Se congela sola cuando la temperatura cae bajo un umbral (nada de un `requestAnimationFrame` corriendo para siempre en una escena ya asentada) y se recalienta solo con una interacción que deba mover nodos (arrastrar uno, datos nuevos) — **pan/zoom no recalientan**, son transformaciones de la vista.

Gestos de mouse, mismo lenguaje que [[KneMap]]: rueda zoomea hacia el puntero, arrastrar el fondo paneá, arrastrar un nodo lo fija (`pinned`, dejar de moverse con la simulación). Opacidad base por carpeta (variación de brillo vía un hash chiquito, no un color nuevo — monocromático); hover resalta el nodo + sus vecinos directos y atenúa el resto; el filtro de tags (panel TAGS de la sidebar izquierda) atenúa el grafo a las notas con ese tag.

Misma clase para el grafo global (todas las notas + todos los links resueltos, deduplicados sin importar dirección) y el local (vecindario a 1 salto de la nota abierta, con `focusPath` marcado por los `<>`) — `KneOsBrain._refreshLocalGraphData` recalcula el vecindario en cada navegación, `_graphNodeFor`/`buildGraphEdges`/`buildDegreeMap` son helpers de módulo compartidos por ambos.

> [!bug] "El grafo se vuelve loco" al zoomear (2026-08-25, reportado por Manuel, resuelto)
> Dos problemas superpuestos. **(1)** La rueda del mouse (`onWheel`, zoom hacia el puntero) llamaba a `_wake()` en cada tick de scroll — cada scroll de zoom recalentaba y reiniciaba la simulación entera, así que el layout se reacomodaba solo cada vez que el usuario intentaba zoomear (pan/zoom son transformaciones de la vista, no deberían tocar el layout para nada). **(2)** El sistema de física original (velocidad + `DAMPING = 0.82`, sin tope de desplazamiento) no garantizaba convergencia: si dos nodos quedaban muy cerca (el círculo de arranque con 58 nodos en un radio chico los deja bastante juntos), la repulsión entre ellos podía dar una fuerza enorme en un solo frame, y sin ningún tope esa velocidad se propagaba y podía amplificarse en vez de absorberse, produciendo un jitter visible que nunca bajaba del umbral de "energía" para dormir la simulación — se veía como si el grafo temblara/rebotara sin parar. Fix: se reemplazó el sistema de velocidad/damping por un layout Fruchterman-Reingold clásico, donde el desplazamiento de cada nodo está topeado por una "temperatura" que decae en forma monótona cada frame — sea cual sea la fuerza instantánea, el desplazamiento nunca puede superar ese tope, y el tope mismo tiende a 0, así que la simulación converge siempre, sin excepción. Además se sacó el `_wake()` de `onWheel` — zoomear ya no toca el layout en absoluto. Verificado con Playwright: 15 scrolls de zoom in seguidos + 15 de zoom out no alteran la disposición de los nodos, solo la vista.

## Responsive (ventana redimensionable hasta 300×200)

`container-type: inline-size` en `.brainApp` (no un media query: lo que importa es el ancho de **la ventana**, no el del viewport del navegador — mismo motivo por el que la portada grande de [[Knefy]] se había roto con `vw`). Por debajo de 700px los paneles laterales se ocultan solos; por debajo de 460px también se ocultan la navegación atrás/adelante y la ruta del header. Verificado con Playwright arrastrando la ventana real a su mínimo 300×200: sin scroll horizontal, sin overlap.

## Verificación

Renderizadas las 57 notas reales del vault sin excepciones (script standalone contra `vault.json` real, sin pasar por la UI). Probado en navegador con Playwright: grafo global asentándose (rápido y estable, sin jitter) con glifos+aristas ASCII visibles, 15 scrolls de zoom in + 15 de zoom out sin alterar el layout, navegación por búsqueda/árbol/wikilink/backlink, [[Knefy]] con sus 7 callouts (1 abstract + 6 bug) y su tabla de estados de pantalla, grafo local con vecindario correcto y la nota abierta marcada `<@>`, repintado al cambiar el color del sistema desde [[Config]], y el caso 300×200 de arriba.
