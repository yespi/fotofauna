# Identificación en grupos — FotoFauna

**Fecha:** 2026-06-13  
**Estado:** implementado en PRE (`public-pre/`)

## Problema

Al agrupar fotos por drag & drop, la lógica antigua unificaba `species` en todas las fotos del grupo. Eso:

- Contagiaba identificaciones incorrectas entre fotos.
- Rompía identificaciones al desagrupar si se habían sobrescrito.

En cambio, al publicar en **Minka** o **iNaturalist**, un grupo debe ser **una sola observación** con **un taxón** para todas las fotos.

## Política (NO borrar)

| Fase | Comportamiento |
|------|----------------|
| **Edición** (agrupar, recortar, ubicar) | Cada foto conserva su `species` / `organisms`. **No** unificar al arrastrar. |
| **Identificación de grupo** | El master guarda `groupSpecies`: primera ID existente al agrupar, o la que el usuario confirme en el panel. |
| **UI rejilla** | Grupos muestran `groupSpecies` (taxón canónico), no la foto visible del carrusel ‹ ›. |
| **Publicación** Minka/iNat | Se usa `groupSpecies` (o fallback) para el payload y se aplica el mismo taxón a todas las fotos del batch. |
| **Desagrupar** | Cada foto sale con su `species` individual intacta. |

## Archivos clave

| Archivo | Rol |
|---------|-----|
| `composables/group-species.js` | Helpers centralizados + comentario de política |
| `composables/use-photo-actions.js` | `onDropOnCell`, `_cloneGroupMember`, `groupSpecies` al agrupar/desagrupar |
| `composables/use-identificacion.js` | Actualiza `groupSpecies` al confirmar ID en master con hijos |
| `composables/use-upload-platforms.js` | Payload unificado + `groupSpeciesPatchForPublish` |
| `PhotoGrid.js` | `getGroupCellDisplay()` para overlay/badge |

## Reglas para futuros cambios

1. **NO** reintroducir `winnerIdent` / `applyIdent` en `onDropOnCell`.
2. **SÍ** mantener clonado defensivo en `_cloneGroupMember` (evita referencias compartidas).
3. **SÍ** usar `resolveGroupSpeciesForPublish` / `groupSpeciesPatchForPublish` al publicar.
4. Si se añade persistencia nueva, incluir `groupSpecies` solo en el **master** (no en hijos).

## Bug sesión + identificación en grupo (2026-06)

**Síntoma:** nombre del grupo repetido en rejilla; al cambiar la ID, la identificación vieja aparecía ~N filas más abajo (N = tamaño del grupo).

**Causa:** al restaurar sesión, `flattenMeta` indexaba también los **hijos** del grupo; una foto suelta con el mismo nombre recibía metadatos del hijo. Referencias compartidas de `species`/`groupSpecies` amplificaban el efecto al editar.

**Fix (PRE):** `flattenMeta` solo masters; clonado defensivo en restore/`_applyToPhoto`; identificación aplicada al hijo visible (‹ ›) cuando corresponde; `groupSpecies` solo al confirmar en el master del grupo.

## Persistencia sesión — grupos y antipartículas (2026-06)

**Síntoma:** tras recargar y volver a cargar fotos, se pierden duplicados agrupados y ediciones de antipartículas.

**Causas:**
1. Agrupar por drag & drop **no marcaba la sesión como dirty** → el autosave no guardaba la estructura de grupos.
2. Cooldown de 5 min **bloqueaba** guardados aunque hubiera cambios recientes.
3. Antipartículas genera `file_filtered` en memoria pero **no se persistía** el JPEG editado (solo metadatos parciales).

**Fix (PRE):** `_afterStructuralChange` → `scheduleAutosave`; guardado forzado al cerrar pestaña (`flushAutosave`); blobs editados en IndexedDB (`edited_files`); serialización de `stamp_particles`, `active_version`, `hasEditedFile`.

### Fix definitivo duplicados + restore blobs (13-jun noche)

**Por qué seguía fallando tras arreglos anteriores:** se persistían blobs en IndexedDB, pero al recargar `_restoreEditedFileForPhoto` llamaba a `_applyToPhoto`, que busca la foto en `photos.value` **antes** de que el import las haya añadido → el blob existía pero nunca se aplicaba a la foto. Los duplicados de grid (`IMG_001_1.jpg`) tampoco se restauraban porque ese JPG no existe en disco.

**Fix adicional (PRE):**
1. Restore de blobs: `Object.assign` directo sobre la foto en el array de import (no `_applyToPhoto`).
2. `duplicatePhoto`: clona `file_filtered`/`file_original`, guarda `duplicateOf`, autosave urgente, blob inmediato.
3. Restore duplicados: reconstruye master sintético desde el original si falta el JPG `_N`.
4. E2E Playwright: 24 checks incl. duplicar + antipartículas + reload (timer cada 6 h).

Detalle completo (correo enviado 13-jun): en git history, `CORREO_PERSISTENCIA_DUPLICADOS_2026-06-13.md`.
