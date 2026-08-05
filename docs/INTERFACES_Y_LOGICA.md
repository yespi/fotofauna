# Interfaces y Lógica de Aplicación — FotoFauna

**Última actualización:** 2026-05-07

---

## Dos clientes independientes

FotoFauna tiene **dos implementaciones completamente separadas**. Un bug o fix en una NO aplica a la otra automáticamente.

| | Desktop | Móvil |
|--|---------|-------|
| Ficheros | `index.html` + `PhotoGrid.js` + composables | `index-mobile.html` (fichero único ~3700 líneas) |
| URL PRO | `https://fotofauna.yespi.es/` | `https://fotofauna.yespi.es/index-mobile.html` |
| URL PRE | `https://fotofauna.yespi.es/pre/` | `https://fotofauna.yespi.es/pre/index-mobile.html` |
| Framework | Vue 3 CDN, multi-fichero | Vue 3 CDN, fichero único |
| Editar en | `webapp/public-pre/` | `webapp/public-pre/index-mobile.html` |

---

## Modelo de datos — foto (compartido)

```javascript
{
  id,           // UUID runtime
  name,         // nombre de fichero
  file,         // objeto File original
  thumbUrl,     // miniatura base (blob URL)
  cropThumbUrl, // miniatura recortada generada
  showCrop,     // boolean: mostrar recorte o original
  status,       // 'idle' | 'processing' | 'detected' | 'error' | 'discarded'
  species,      // especie asignada (IA o manual)
  organisms,    // array resultados detección YOLO [{bbox, label, confidence, ...}]
  hasOrganism,  // boolean: hay organismo detectado
  gps,          // { lat, lng } o null
  exif,         // metadata EXIF leída
  children,     // fotos apiladas bajo esta master
  disabled,     // excluida de export y acciones globales
  edit_params,  // estado editor: { exposure, brightness, contrast, saturation,
                //   temperature, shadows, highlights, vibrance, crop }
}
```

---

## Cliente Desktop (`PhotoGrid.js`)

### Topbar

**Dropzone / input file**
- Arrastra o haz click para cargar imágenes o carpetas.
- `addImageFiles()` → GPS EXIF extraído en lote de 20 (`_extractExifClientSide`) antes de mostrar las fotos.
- Sin diálogo bloqueante de ubicación: el panel OSM se sugiere si no hay GPS.

**Botón ☑ Todas** — `selectAll()` — selecciona fotos visibles no deshabilitadas.

**Botón ☐ Ninguna** — `clearSelection()`.

**Botón 💾 Guardar (N)** — `onExport()`:
1. Construye payload con fotos seleccionadas y sus stacks.
2. POST a `/vision/session/export`.
3. Descarga `fauna_export_YYYY-MM-DD.json`.
4. Limpieza de sesión (`/vision/session/cleanup` + revoke URL blobs).

### Rail izquierdo (pestañas verticales)

- 🔍 **Identificar** — `toggleLeftPanel('identify')` — panel taxonómico.
- 📍 **Ubicación** — `toggleLeftPanel('location')` — panel OSM + Leaflet.
- ✂ **Recortar** — modal full-screen Recorte2 (CropperJS + 10 sliders + presets ratio).

### Panel Ubicación

- **Búsqueda Nominatim** — `locDoSearch()` — resultados OSM.
- **Lista resultados** — `locSelectResult()` — elige lat/lon/nombre.
- **Mapa Leaflet** — pin azul = ubicación buscada; pins rojos = fotos con GPS aplicado.
- **Aplicar ubicación** — `applyLocation()` — solo escribe en fotos sin GPS. Nunca sobreescribe GPS existente.

### Panel Identificación

- **◁ ▷ Navegar fotos** — `idNavPrev/Next()`.
- **◁ ▷ Navegar grupo** — `idGroupPrev/Next()` — dentro de master+children.
- **Input buscar especie** — debounce 350ms → `GET /proxy/taxa/autocomplete`.
- **Click resultado** — `idPickResult()` → `confirmIdentification()` → propaga a children.
- **❓ Sin identificar** — `markUnknown()` — avanza a siguiente.

### Grid central

| Acción | Handler |
|--------|---------|
| Click celda | `selectPhoto()` — simple/rango/toggle según modificador |
| Doble click | Abre editor modal |
| Drag entre celdas | Apila fotos |
| Zona "Soltar para desagrupar" | `onDropUnstack()` |
| ↗ Eject child (sidebar) | `ejectChild()` — child → master en raíz |
| 👁/🚫 | `toggleDisable()` — excluye de export |
| Duplicar | `duplicatePhoto()` — clon con sufijo _N |
| X Eliminar | `removePhoto()` → status=discarded |
| Toggle recorte/original | `setShowCrop()` — solo visual |
| Rubber-band | `onGridMouseDown` + mousemove/up — selección rectangular |

### Editor de recorte (Recorte2)

Modal full-screen accesible desde rail izquierdo.

- **Navegación:** flechas ◁▷ o teclado — guarda antes de mover.
- **Sliders (10):** Brillo, Contraste, Saturación, Exposición, Temperatura, Nitidez, Sombras, Luces, Vibrance, Viñeta.
- **Presets ratio:** 1:1, 4:3, 3:2, 16:9, 5:7, 2:3, 3:4, Libre.
- **Ajustar a IA** — `resetCropToBbox()` — reaplica bbox de `organisms[0]` con padding.
- **Restablecer** — `resetAll()` — sliders a default, crop a bbox si existe.
- **Guardar automático** al navegar o cerrar (no hay botón manual de Guardar).
- **Criterio recorte pequeño:** si cubre <10% de ancho/alto → se normaliza a imagen completa.

### Atajos de teclado (Desktop)

**Grid:**

| Atajo | Acción |
|-------|--------|
| Ctrl/Cmd+A | Seleccionar todas |
| Delete | Descartar foto enfocada |
| E | Abrir editor |
| U | Desagrupar a raíz |
| Flechas | Mover foco |
| Shift+Flechas | Ampliar selección |
| PageUp/Down/Home/End | Navegación rápida |

**Editor:**

| Atajo | Acción |
|-------|--------|
| Escape | Guardar y cerrar |
| ← → | Guardar y navegar |

---

## Cliente Móvil (`index-mobile.html`)

### Galería principal

**Input file (área tap / zona de carga)**
- `addFiles(files)` — async, pre-parsea GPS EXIF de todas las fotos antes de decidir mostrar el diálogo de ubicación.
- Si alguna foto tiene GPS → carga sin preguntar.
- Si ninguna tiene GPS → muestra `locGateVisible` (diálogo de ubicación).

**Long press en foto** — inicia selección múltiple.

**Tap en foto** → abre sheet de detalle.

### Sheet de detalle (foto individual)

- Especie identificada (nombre científico + común).
- Confianza de identificación.
- GPS / ubicación.
- Botones: **Recortar**, **Publicar en Minka**, controles de grupo.

### Editor de recorte (móvil)

> ⚠️ El guardado es automático al navegar o cerrar. No existe botón manual de Guardar.

**Fila principal de botones (5):**

| Botón | Función |
|-------|---------|
| ✨ Auto | `autoEnhance()` — mejora automática |
| ↶ -90° | `cropRotateLeft()` |
| ↷ +90° | `cropRotateRight()` |
| ◑ B/A / 👁 Original | `cropToggleBA()` — muestra imagen sin filtros / restaura |
| ⋮ Más | `cropMoreOpen = true` — expande menú |

**Menú ⋮ expandido:**

| Botón | Función |
|-------|---------|
| 🌫 Limpiar | `cropApplyDehaze()` — dehaze vía `/vision/dehaze` |
| 🔍 Nitidez | `cropApplyUnblur()` — unblur |
| ☀ Exposición | `cropApplyOverexposure()` — corrección sobreexposición |
| 🎯 IA bbox | `cropApplyBbox()` — encuadra al bbox de `organisms[0]`; deshabilitado si sin organismo |
| 🌊 Marino | Corrección dominante azul/verde marino (`/vision/marine-correct`) |

**Sliders (8):**

| Slider | Rango | Nota |
|--------|-------|------|
| Exposición | -100 a +100 | |
| Brillo | 0.5 a 1.5 | |
| Contraste | 0.5 a 1.5 | |
| Saturación | 0 a 2 | |
| Temperatura | -100 a +100 | cálido/frío |
| 🌑 Sombras | -100 a +100 | añadido 2026-05-07 |
| ☀ Luces | -100 a +100 | añadido 2026-05-07 |
| 🎨 Vibrance | -100 a +100 | añadido 2026-05-07 |

**Before/After (B/A):**
- Pulsar B/A → quita CSS filters → muestra original. Botón cambia a "👁 Original".
- Pulsar Original → restaura filters. Client-side, sin llamada al servidor.

**Navegación en editor:**
- Botones ◁ Anterior / Siguiente ▷ — autoguardan la foto actual antes de cargar la siguiente.
- Contador `N / Total` visible.

**`_buildFilterStr(a)` — filtro CSS compartido:**
```javascript
function _buildFilterStr(a) {
  const b = Math.max(0, a.brightness + a.exposure * 0.25);
  const shadowF    = 1 + ((a.shadows    ?? 0) / 100) * 0.3;
  const highlightF = 1 + ((a.highlights ?? 0) / 100) * 0.3;
  const vibranceS  = Math.max(0, (a.saturation ?? 1) + ((a.vibrance ?? 0) / 100) * 0.5);
  return [
    `brightness(${(b * highlightF).toFixed(3)})`,
    `contrast(${a.contrast.toFixed(3)})`,
    `saturate(${vibranceS.toFixed(3)})`,
    a.temperature > 0
      ? `sepia(${(a.temperature / 100 * 0.4).toFixed(3)})`
      : `hue-rotate(${(-a.temperature * 0.3).toFixed(1)}deg)`,
    shadowF !== 1 ? `brightness(${shadowF.toFixed(3)})` : '',
  ].filter(Boolean).join(' ');
}
```

### Agrupación de fotos (móvil)

- Long press en una foto activa selección múltiple.
- Barra de selección muestra botón **Agrupar**.
- Fotos agrupadas aparecen como stack con thumbnail del master.

### Identificación manual (móvil)

- Input en sheet de detalle: escribir nombre científico o común.
- Autocomplete contra `/proxy/taxa/autocomplete`.
- Confirmar → guarda especie + propaga al grupo.

---

## Pipeline IA (compartido)

```
Fase 1 — Localizar   POST /vision/detect  (batch 2)
   YOLOv8n (CPU local) → bbox, label, confianza

Fase 2 — Recortar    canvas local, sin red
   Recorte bbox + padding, mejora contraste/saturación

Fase 3 — Identificar (incluido en /vision/detect)
   Motor 1: iNaturalist CV  (principal, JWT renovable)
   Motor 2: Google Gemini   (fallback si score < 0.50)
   Motor 3: Groq Llama      (validación taxonómica)

   Estrategia adaptativa:
     confianza ≥ 0.50 → devuelve inmediatamente
     confianza < 0.50 → espera hasta 8s más
     Timeout global: 30s

Fase 4 — GPS EXIF client-side
   Móvil:   readGpsFromFile()  — parser TIFF/EXIF JS puro
   Desktop: _extractExifClientSide()  — batch de 20
   Si GPS detectado en cliente → no se muestra diálogo de ubicación
```

---

## GPS EXIF — Parsing client-side

### Móvil (`readGpsFromFile`)

Parser TIFF/EXIF en JS puro. Convierte tags GPS (DMS rational) a decimal (DD).

Llamado en `addFiles()` **antes** de decidir mostrar `locGateVisible`:

```javascript
async function addFiles(files) {
  // ...
  const gpsResults = await Promise.all(imgs.map(f => readGpsFromFile(f).catch(() => null)));
  const anyHasGps = gpsResults.some(g => g != null);
  if (!anyHasGps) {
    _pendingFiles = imgs;
    locGateVisible.value = true;
    return;
  }
  _pendingFilesGps = gpsResults; // reutilizados en _doAddFiles
  _doAddFiles(imgs);
}
```

**Bug corregido 2026-05-07:** `readRational` y `readAscii` usaban `t.valOff - tiffStart` causando doble resta del offset. Fix: usar `t.valOff` directamente.

### Desktop (`_extractExifClientSide`)

- Extrae GPS en lote de 20 antes de mostrar las fotos.
- No muestra diálogo bloqueante; sugiere abrir panel de ubicación.

---

## Endpoints consumidos desde frontend

```
POST /vision/detect
POST /vision/detect?fallback=1
POST /vision/dehaze
POST /vision/dehaze/score
POST /vision/marine-correct
POST /vision/session/export
POST /vision/session/cleanup
POST /admin/errors           ← errores del cliente móvil
GET  /proxy/taxa/autocomplete
GET  /proxy/minka/obs
POST /proxy/minka/publish
GET  Nominatim search/reverse (OSM, externo)
```

---

## Reglas funcionales críticas

1. **No sobreescribir GPS existente** al aplicar ubicación manual.
2. **Fotos disabled** quedan fuera de export y acciones globales.
3. **El editor guarda automáticamente** al navegar o cerrar (no hay botón manual de Guardar). Esto aplica a ambos clientes.
4. **Recorte pequeño** (<10% de ancho/alto) se normaliza a imagen completa.
5. **Selección múltiple** usa Set reactivo reemplazando referencia completa (Vue 3 reactivity).
6. **GPS pre-check móvil** — siempre antes de mostrar `locGateVisible`.
7. **Dos clientes independientes** — verificar siempre en cuál se trabaja antes de editar.

---

## Publicación Minka — Desktop

- Fuente: fotos master seleccionadas (`selectedIds`).
- Cada master → observación independiente en Minka (con sus children).
- Sin mezcla entre masters distintos.
- Fecha de observación: EXIF normalizado `YYYY-MM-DD` → `observed_on`.
- Sin ubicación → abortado con error explícito.
- Feedback: resumen por lote con URL de observación creada.

---

## Guía para pedir cambios a la IA

```
1. Objetivo UX:        qué debe hacer el usuario de forma diferente
2. Cliente afectado:   Desktop / Móvil / Ambos
3. Fichero exacto:     PhotoGrid.js / index-mobile.html / composable específico
4. Función(es):        nombre exacto del handler a modificar
5. Cambio funcional:   condición actual → condición deseada
6. Casos de prueba:    escenarios A/B/C con resultado esperado
7. No regresiones:     GPS no sobreescrito, guardado automático, reactividad Vue
```
