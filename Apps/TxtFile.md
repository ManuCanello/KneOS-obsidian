---
tags:
  - portfolio/kneos
  - apps
---

# TxtFile

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/TxtFile.js` — extiende [[File]]. Extensión `"txt"`, ícono `sources/appIcon/txtIcon.png`.

> [!abstract] Qué hace
> Editor de texto tipo bloc de notas con formato enriquecido (negrita, cursiva, subrayado, tachado, H1), contador de caracteres, indicador de estado de guardado, persistencia manual (botón de guardar o Ctrl+G), y desde 2026-07-30 dos flujos de IA: un botón fijo en la toolbar que muestra la respuesta en un preview (Aceptar/Volver/Cancelar) antes de agregarla al final de la nota, y un menú contextual (click derecho) — solo aparece con texto seleccionado — con la opción "Preguntarle a KneAI" para reescribir esa selección in place.

## Constructor(nombre)

`super()`; inicializa `_texto=""`, `_editorEl=null`, `_txtServices = new TxtServices()`, `_cargado=false`, `_contextMenu = new ContextMenu()` (instancia propia, mismo patrón que [[Folder]]), `_selectionAiPopup=null`. Desde 2026-07-31, `super()` pasa `FileType.PRODUCTIVITY` (ver [[File]]) en el lugar del viejo parámetro `direction`.

## Funciones

- **`_crearContenido()`**: construye `.txtContainer` (`position:relative`, ancla de `.txtAiSelectionPopup`) con menú de acciones, un `div` `contentEditable` (`.txtTextField`) como área de edición y un footer. Escucha `input` (sincroniza `_texto`, actualiza footer), `keydown` (Ctrl+G → `_guardar()`) y `contextmenu` del editor → si hay selección no colapsada, `preventDefault()` + `_openSelectionContextMenu`; si no, deja pasar el menú nativo del navegador (copiar/pegar/sugerencias de ortografía). Dispara `_cargarContenido()` si `!_cargado`.
- **`async _cargarContenido()`**: marca `_cargado=true`; si `id==null` no hace nada; si no, pide el contenido a `TxtServices.getContent(id)` y lo carga con `setText()`.
- **`_crearMenuAccions()`**: botón de IA (primero del todo) + botones H1/B/I/U/S (cada uno llama `_setStyle(tag)`) + botón guardar. El botón guardar (`.txtIconSave`, 2026-07-29) usa `sources/accions/save.svg` vía el mismo patrón `::before` + `mask-image` + `currentColor` que los botones de [[Window y Taskbar|Window]] (hereda el hover-invert de `.txtIcon:hover` sin reglas propias) — antes mostraba el emoji 💾 como texto. De paso se sacó el atributo `title` (violaba la convención del proyecto de no usar tooltips nativos) y se reemplazó por `aria-label`.
- **Botón de IA general (`.txtIconAi`, 2026-07-30)**: ícono `sources/appIcon/kneAi.png` aplicado con `aplicarIconoImagen` (PNG ya coloreado, sin mask). A diferencia de los demás `.txtIcon`, anula el hover-invert de fondo (el ícono es verde y quedaría invisible contra el fondo verde del invert) y usa `filter: brightness(1.3)` como señal de hover en su lugar.
  - **`_toggleAiMenu(menu, btnAi)` / `_openAiMenu(menu, btnAi)`**: abre/cierra un popup (`.txtAiPopup`, 420×420) anclado a `.txtMenuAcciones` (`position:relative`). Se cierra al clickear afuera (listener en `document`, guardado en `popup._closeOnOutsideClick` para poder sacarlo al togglear/aceptar/cancelar vía `_closeAiPopup`).
  - **Flujo de dos vistas dentro del mismo popup** (no toca la nota hasta que el usuario confirma):
    - **`_renderAskView(popup, previousQuestion)`**: `<textarea>` (`.txtAiInput`, Enter sin Shift envía) + botón "Enviar". Recibe la pregunta anterior para poder restaurarla al volver desde la vista de respuesta.
    - **`async _askAi(popup, textarea)`**: arma el mismo tipo de prompt de sistema que [[KneAI]] (fuerza HTML puro, nada de Markdown) y llama `window.groq.ask(context, question, [])` — sin historial, cada pregunta es independiente. Si falla (`null`), deja la pregunta puesta para reintentar (mismo criterio que KneAI).
    - **`_renderAnswerView(popup, question, answer)`**: preview del HTML devuelto (`.txtAiPreview`) + fila de acciones (`.txtAiActions`) con **Aceptar** (inserta la respuesta y cierra el popup), **Volver** (vuelve a `_renderAskView` con la pregunta restaurada, sin descartar nada) y **Cancelar** (cierra el popup sin tocar la nota). Los tres `click` hacen `stopPropagation` — necesario porque mutan `popup.innerHTML` sincrónicamente, y si el click siguiera burbujeando hasta el listener de "click afuera" de `document`, el botón ya estaría desconectado del popup y el listener lo interpretaría como un click afuera, cerrando todo de golpe (bug real, encontrado y arreglado en la vuelta "Volver").
  - **`_appendAiAnswer(html)`**: `insertAdjacentHTML("beforeend", html)` directo sobre `_editorEl` (sin wrapper ni clase propia — se sacó `.txtAiResponse`/el separador visual a pedido), sincroniza `_texto` y refresca el contador de letras. Se persiste recién cuando el usuario guarda (botón o Ctrl+G), no hay auto-guardado.
- **IA de selección — menú contextual (2026-07-30)**. Pasó por dos diseños antes de este: primero un botón fijo bajo la toolbar, después un botón flotante que seguía la selección (`.txtAiSelectionFloater`, con `getBoundingClientRect()` para no depender de que las ventanas se arrastran con CSS `transform`, ver [[Window y Taskbar|Window]]) — descartado por el usuario ("no funciona correctamente") en favor de un menú contextual nativo del proyecto:
  - **`_openSelectionContextMenu(container, clientX, clientY, range)`**: abre un `ContextMenu` (`AI_SELECTION_CONTEXT_MENU_ID`) en `(clientX, clientY)` con un único ítem, "Preguntarle a KneAI" (ícono `kneAi.png`, `closeMenuId` para que el menú se cierre solo al elegirlo). No es un submenú porque el `ContextMenu` genérico ([[Menús Contextuales]]) no soporta inputs de texto dentro de un ítem — la opción abre un popup aparte.
  - **`_openSelectionAiPrompt(container, clientX, clientY, range)`**: crea `.txtAiSelectionPopup` — `<input type="text">` (`.txtAiSelectionInput`, reusa `.txtAiInput` para el estilo) + botón "Enviar" — posicionado en `(clientX, clientY)` relativo a `container` (mismo cálculo de rects que el resto del proyecto). Guarda la instancia en `this._selectionAiPopup` (una sola a la vez).
    > [!bug] Mismo bug que "Volver", en otro disparador — encontrado y arreglado acá
    > El listener de "click afuera" no se puede registrar de forma síncrona: el click que abre este popup (el ítem del menú contextual) sigue burbujeando hacia `document` en ese mismo evento, y `ContextMenu.addItem` no le hace `stopPropagation()` (es genérico, no se le puede pedir que sí sin afectar a todos los demás menús del proyecto). Si el listener quedara activo ya, se dispararía con ese mismo click y cerraría el popup apenas se abriera. Se registra con `setTimeout(..., 0)` para que quede activo recién en el próximo tick.
  - **`async _askSelectionAi(input, range)`**: arma un prompt distinto al general (reescribe un fragmento dado + una instrucción, en vez de responder libremente) con el mensaje `Texto seleccionado: "..." / Instrucción: ...`. Igual criterio de fallo silencioso que el resto (deja la instrucción puesta para reintentar).
  - **`_replaceSelectionWithAi(range, html)`**: `range.deleteContents()` + `range.insertNode(range.createContextualFragment(html))` — reemplaza el fragmento seleccionado directamente, sin wrapper. Sincroniza `_texto` y el contador de letras.
  - **`_closeSelectionAiPrompt()`**: saca el listener de "click afuera" de `this._selectionAiPopup`, lo saca del DOM y limpia la referencia.
- **`async _guardar()`**: si `id==null` no hace nada; si no, `TxtServices.saveContent(id, texto)`, muestra estado ("Guardado"/"Error"), y si tuvo éxito llama `_actualizarColumnaLista()`.
- **`_actualizarColumnaLista()`**: recalcula tamaño real en bytes UTF-8 (`new Blob([texto]).size`), actualiza `size`/`updatedAt`, refresca columnas ([[File|_refrescarColumnas]]) y propaga el delta de tamaño a las carpetas ancestro (`_propagarTamano`).
- **`_getFooter()` / `_changueFooterInfo()`**: contador de letras.
- **`_mostrarEstadoGuardado(texto)`**: mensaje temporal (2s).
- **`_getSelected()`**: `Range` de la selección actual dentro del editor.
- **`_setStyle(element)`**: aplica/quita una etiqueta HTML (`STRONG`, `EM`, `U`, `S`, `H1`) sobre la selección.
- **`_getCharLength()`**: cantidad de caracteres (trim).
- **`getText()` / `setText(texto)`**: getter/setter público del contenido.
- **`contarPalabras()`**: cantidad de palabras — método público utilitario, no parece usarse en la UI directamente.

## Persistencia

[[Frontend Model Services Utils#Services|TxtServices]] → [[Módulo Txt]]. El botón de IA pasa por [[Frontend Model Services Utils#Services|Groq.js]] (`window.groq`) → [[Módulo Groq]] — mismo servicio global que usa [[KneAI]], pero acá sin historial de chat.
