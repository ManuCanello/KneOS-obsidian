---
tags:
  - portfolio/kneos
  - backend
---

# Módulo KPaint

⬅️ Volver a [[Backend]]

Persistencia del contenido de un dibujo de [[KPaint]]: un array de 4096 (64×64) índices de paleta (0-7), uno por píxel — **sin color final**, el color se resuelve siempre client-side al dibujar, mismo criterio que [[Módulo CameraPhoto]]. Acoplada al sistema de íconos ([[Módulo Icon]]) con el mismo molde que `camera_photos`/`txt` (tabla 1:1 con `files` por `id_icon`, upsert + actualización de `size` en una transacción). Agregada 2026-08-20 al crear [[KPaint]].

## Endpoints (`routes/kpaintRoutes.js`, montado en `/kpaintRoutes`, `requireAuth` en todo el router)

| Método | Path | Controller |
|---|---|---|
| PATCH | `/drawing` | `saveDrawing` |
| GET | `/drawing/:id_icon` | `getDrawing` |

## Controllers (`controllers/kpaintController.js`)

- **`saveDrawing`**: recibe `id_icon, pixels` del body + `req.pcId` del token. Valida `id_icon` (`isValidId`) y `pixels` con `isIndexArray(value, length, max)` (`utils/validation.js`, 2026-08-20 — generalización de `isGrayscaleArray` a un tope arbitrario en vez de 255 fijo) — array de exactamente `KPAINT_PIXEL_COUNT = 64*64 = 4096` enteros `0 ≤ v ≤ 7` (`KPAINT_PALETTE_MAX_INDEX`, ambas constantes duplicadas del frontend — `KPAINT_SIZE`/`KPAINT_PALETTE_SIZE` en `public/KneOS/js/model/kpaintPalette.js` — porque el backend no comparte módulos con el frontend). Llama `saveKPaintDrawing(...)`; `null` (ícono no existe o no es del `pc_id`) → `404`; éxito → `{ success: true }`. Mismo shape que `savePhoto` de [[Módulo CameraPhoto]].
- **`getDrawing`**: `req.params.id_icon` + `req.pcId` → `getKPaintDrawing(id_icon, req.pcId)`, devuelve `{ pixels }` (array vacío si no hay fila o el ownership no coincide — siempre `200`).

## Modelo (`models/kpaintModel.js`)

- **`saveKPaintDrawing(id_icon, pc_id, pixels)`**: verifica ownership (`findFirst` en `files`), calcula `size = Buffer.byteLength(JSON.stringify(pixels), 'utf8')`, y en una `$transaction` hace `kpaint_files.upsert` (contenido) + `files.update` (nuevo `size`). Devuelve la fila `kpaint_files` resultante o `null` si el ícono no existía.
- **`getKPaintDrawing(id_icon, pc_id)`**: `findFirst` en `kpaint_files` filtrando por `id_icon` y relación anidada `files: {pc_id}`, `select: {pixels: true}`.

## Tabla `kpaint_files` (`prisma/schema.prisma`, 2026-08-20)

```prisma
model kpaint_files {
  id_icon Int   @id
  pixels  Json
  files   files @relation(fields: [id_icon], references: [id_icon], onDelete: NoAction, onUpdate: NoAction)
}
```

`pixels` es `Json` — un array plano de índices de paleta (0-7), no el color final. `prisma db push` (sin migración con nombre — este proyecto no versiona `prisma/migrations/`, ver [[Deuda Técnica]] si aplica) sincronizó la tabla directo contra la BD de desarrollo.

> [!info] Acoplamiento con Icon
> Igual que `camera_photos`/`txt`: no se puede crear una fila en `kpaint_files` sin un ícono `kp` preexistente (a diferencia de Camera, acá el ícono nace directo del menú "Nuevo" — `DesktopManager.crearIcono`/`crearIconoEnCarpeta` — no de un flujo de guardado de otra app), y borrar el icono borra en cascada manual su `kpaint_files` (`deleteIconRecursivo`, ver [[Módulo Icon]]).

## Consumido por

Servicio frontend `KPaintServices` (`public/KneOS/js/services/KPaintServices.js`), usado por [[KPaint]] tanto para `getDrawing` (al abrir) como `saveDrawing` (al guardar).
