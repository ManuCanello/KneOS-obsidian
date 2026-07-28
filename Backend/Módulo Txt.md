---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Txt

⬅️ Volver a [[Backend]]

Persistencia del contenido de archivos de texto ([[TxtFile]]), acoplada al sistema de íconos ([[Módulo Icon]]).

## Endpoints (`routes/txtRoutes.js`, montado en `/txtRoutes`, `requireAuth` en todo el router desde 2026-07-28)

| Método | Path | Controller |
|---|---|---|
| PATCH | `/content` | `saveContent` |
| GET | `/content/:id_icon` | `getContent` |

## Controllers (`controllers/txtController.js`)

- **`saveContent`**: recibe `id_icon, txtcontent` del body + `req.pcId` del token. Valida (2026-07-27, `utils/validation.js`) `id_icon` como id válido y `txtcontent` como string (**puede ser vacío** — un `.txt` vacío es válido, a diferencia de otros campos de texto del proyecto). Llama `saveTxtContent(...)`; si el modelo devuelve `null` (ícono no existe o no pertenece al `pc_id`), responde `404 { error: "Icono no encontrado" }`; si tuvo éxito, `{ success: true }`. Es el único controlador "de escritura" del proyecto que maneja explícitamente un caso 404 además del 500 genérico.
- **`getContent`**: `req.params.id_icon` + `req.pcId` → `getTxtContent(id_icon, req.pcId)`, devuelve `{ txtcontent }` (string vacío si no hay contenido o el ownership no coincide — siempre `200`, no distingue "no encontrado" de "vacío").

## Modelo (`models/txtModel.js`)

- **`saveTxtContent(id_icon, pc_id, txtcontent)`**:
  1. Verifica ownership (`findFirst` en `icons` por `id_icon` + `pc_id`); si no existe, devuelve `null`.
  2. Calcula `size = Buffer.byteLength(txtcontent, 'utf8')`.
  3. Transacción Prisma (`$transaction`) con dos operaciones atómicas: `txt.upsert` (contenido) + `icons.update` (nuevo `size`).
  4. Devuelve la fila `txt` resultante.
- **`getTxtContent(id_icon, pc_id)`**: `findFirst` en `txt` filtrando por `id_icon` y relación anidada `icons: {pc_id}` (asegura que solo el dueño lea el contenido), `select: {txtcontent: true}`.

> [!info] Acoplamiento con Icon
> Cada guardado recalcula el tamaño del archivo en bytes UTF-8 y lo persiste junto con el contenido — este `size` alimenta el cálculo recursivo de tamaños de carpetas (`getTamanosAgregados`, ver [[Módulo Icon]]). No se puede crear un `txt` sin un `icon` preexistente, y borrar un icono borra en cascada manual su `txt`.

## Consumido por

Servicio frontend [[Frontend Model Services Utils#Services|TxtServices]], usado por [[TxtFile]].
