---
tags:
  - portfolio/kneos
  - apps
---

# Curriculum

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Curriculum.js` — extiende [[File]]. Extensión `"cv"`, ícono propio `sources/appIcon/curriculum.svg` (documento con esquina doblada + un mini retrato/hombros, `stroke="currentColor"`, viewBox 24×24 — evoca "CV" y no un `.txt` genérico). Agregada 2026-08-25.

> [!abstract] Qué hace
> El CV de Manuel — estudios, experiencia laboral, habilidades e idiomas — como una app más de KneOS, a pedido explícito tras evaluar qué le faltaba al portfolio como pieza para conseguir trabajo (el resto del proyecto no tenía nada de estudios/experiencia laboral en ningún lado, solo la ficha de contacto de [[User]]). `ViewWindow` de tamaño fijo (760×860, scroll propio), sin el gimmick neón/glitch de [[User]] — acá el contenido tiene que leerse rápido, no verse "corrupto".

## Contenido separado de la vista

Todo el texto vive en `model/curriculumData.js` (`PROFILE`, `EXPERIENCE`, `EDUCATION`, `SKILLS`, `LANGUAGES`, `CERTIFICATIONS`), mismo criterio que `defaultFiles.js`/`asciiEmojis.js` — actualizar el CV no toca el código de render. Dos campos quedan vacíos/null a propósito hasta tener el dato real: `PROFILE.email` (`null`) y `CERTIFICATIONS` (`[]`) — `Curriculum.js` no renderiza esas secciones si no hay contenido, nada de placeholders vacíos ("Certificaciones: próximamente" ni similares).

Secciones, en orden: header (nombre + tagline), Sobre mí, Experiencia (Xiara desde ene. 2026, Sailo desde jun. 2025 — dos empleos simultáneos, cada uno con sus bullets), Estudios (IES Santa Fe, DaVinci, UNL — con su estado terminado/no terminado), Habilidades (chips), Idiomas (español nativo, inglés C1), y Contacto/Certificaciones solo si hay datos.

## Registro de la app

Mismo patrón de 4 lugares que cualquier app nueva (ver [[Knefy]]/[[KneOsBrain]]):
- `model/iconSrc.js` → `cv: { css: "...", load: () => import("../apps/Curriculum.js") }`
- `model/defaultFiles.js` → `{ desktop_place: "espacio69", ext: "cv", name: "Curriculum" }`
- `model/filesUndeletable.js` → `"cv"` agregado al Set
- `utils/formato.js` → `cv: "Aplicación"` en `TIPOS`

`FileType.OTHER` (info personal, igual que [[User]]) — no `PRODUCTIVITY`/`UTILITY`, es una ficha, no una herramienta.

## Pendiente

`CERTIFICATIONS` sigue vacío — Manuel las va a subir más adelante (la sección "Certificaciones" no aparece hasta entonces). `PROFILE.email` ya está confirmado (`canellomanuel2@gmail.com`, 2026-08-25) — distinto del email de la cuenta de Manuel usada para esta sesión de Claude Code, elegido a propósito para el contacto público del CV.
