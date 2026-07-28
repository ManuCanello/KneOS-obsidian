---
tags:
  - portfolio/kneos
  - apps
---

# TxtFile

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/TxtFile.js` — extiende [[File]]. Extensión `"txt"`, ícono `sources/icon/txtIcon.png`.

> [!abstract] Qué hace
> Editor de texto tipo bloc de notas con formato enriquecido (negrita, cursiva, subrayado, tachado, H1), contador de caracteres, indicador de estado de guardado, y persistencia manual (botón 💾 o Ctrl+G).

## Constructor(nombre)

`super()`; inicializa `_texto=""`, `_editorEl=null`, `_txtServices = new TxtServices()`, `_cargado=false`.

## Funciones

- **`_crearContenido()`**: construye `.txtContainer` con menú de acciones, un `div` `contentEditable` (`.txtTextField`) como área de edición, y un footer. Escucha `input` (sincroniza `_texto`, actualiza footer) y `keydown` (Ctrl+G → `_guardar()`). Dispara `_cargarContenido()` si `!_cargado`.
- **`async _cargarContenido()`**: marca `_cargado=true`; si `id==null` no hace nada; si no, pide el contenido a `TxtServices.getContent(id)` y lo carga con `setText()`.
- **`_crearMenuAccions()`**: botones H1/B/I/U/S (cada uno llama `_setStyle(tag)`) + botón guardar.
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

[[Frontend Model Services Utils#Services|TxtServices]] → [[Módulo Txt]].
