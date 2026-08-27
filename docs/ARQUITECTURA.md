# Arquitectura Actual — Ecosistema FotoFauna
**Última actualización:** 2026-05-23

> ⚠️ **Actualizado 2026-05-23.** Sigue siendo válido en lo estructural (contenedor `fauna_api` :3005,
> split PRE/PRO, clientes escritorio y móvil), pero no recoge YOLOFauna ni los cambios de julio–agosto.
> Para el estado reciente: [`CHANGELOG.md`](CHANGELOG.md) y [`YOLOFAUNA.md`](YOLOFAUNA.md).

---

## Resumen ejecutivo

Un único contenedor Python (`fauna_api`) sirve tanto el backend de visión IA como el frontend estático en el puerto 3005. Cloudflare Tunnel expone el servicio con HTTPS sin nginx intermedio. Hay dos entornos frontend: PRO (`/`) y PRE (`/pre/`). La app tiene **dos clientes separados**: escritorio (`index.html` + `PhotoGrid.js`) y móvil (`index-mobile.html`).

---

## ⚠️ REGLAS ABSOLUTAS DE TRABAJO

1. **Ruta canónica única:** `./docker/ecosistema-fauna/` — no existe ninguna ruta alternativa en `/home/yespi/...`
2. **Editar siempre en PRE** (`webapp/public-pre/`), nunca en `public/` directamente.
3. **Deploy PRE→PRO:** usar exclusivamente `bash deploy-to-pro.sh`. Nunca `cp` manual.
4. **Deploy a PRO solo con permiso explícito del usuario.**
5. **Claves API:** siempre en `(configure via environment)`. Nunca en código.

---

## Diagrama de servicios

```
Internet (HTTPS)
  │
  └── https://fotofauna.yespi.es  (Cloudflare Tunnel)
                  │
                  ▼ localhost:3005
        ┌─────────────────────────────────────────────────┐
        │  fauna_api  (Docker, python:3.11-slim)           │
        │                                                  │
        │  FastAPI                                         │
        │  ├── GET  /          → /app/web/      (PRO)      │
        │  ├── GET  /pre/      → /app/web-pre/  (PRE)      │
        │  ├── POST /vision/detect                         │
        │  ├── POST /vision/dehaze                         │
        │  ├── POST /vision/dehaze/score                   │
        │  ├── POST /vision/marine-correct                 │
        │  ├── GET  /vision/health                         │
        │  ├── POST /vision/session/export                 │
        │  ├── POST /vision/session/cleanup                │
        │  ├── GET  /proxy/taxa/autocomplete               │
        │  ├── GET  /proxy/minka/obs                       │
        │  ├── POST /proxy/minka/publish                   │
        │  ├── GET  /admin/errors                          │
        │  └── DELETE /admin/errors                        │
        └─────────────────────────────────────────────────┘
                  │
        ┌─────────┴──────────┐
        │  postgres-global    │  (compartido con otros proyectos)
        │  DB: fauna          │
        │  user: admin_yespi  │
        └────────────────────┘
```

---

## Entornos frontend (PRO / PRE)

| | PRO | PRE |
|--|-----|-----|
| URL | `https://fotofauna.yespi.es/` | `https://fotofauna.yespi.es/pre/` |
| Ficheros host | `webapp/public/` | `webapp/public-pre/` |
| Ficheros contenedor | `/app/web` | `/app/web-pre` |

---

## Dos clientes: Escritorio y Móvil

### Cliente Escritorio
- **Ficheros:** `index.html`, `PhotoGrid.js`, `FaunaApp.js`, `composables/`
- **URL PRO:** `https://fotofauna.yespi.es/`
- **URL PRE:** `https://fotofauna.yespi.es/pre/`
- Framework: Vue 3 CDN, sin build step, múltiples ficheros JS.
- Editor de recorte: `use-recortar.js` con CropperJS, Before/After canvas deslizante.
- Cache-busting: `?v=HASH` en todos los imports JS/CSS; hash calculado automáticamente por `bump-version.sh`.

### Cliente Móvil
- **Fichero:** `index-mobile.html` (todo en un único fichero, ~4800 líneas)
- **URL PRO:** `https://fotofauna.yespi.es/index-mobile.html`
- **URL PRE:** `https://fotofauna.yespi.es/pre/index-mobile.html`
- Framework: Vue 3 CDN, sin build step, fichero único auto-contenido.
- Diseñado para pantallas táctiles, orientación vertical.

> ⚠️ **Son implementaciones completamente independientes.** Un bug/fix en una versión NO aplica a la otra automáticamente. Siempre verificar en qué versión se trabaja.

---

## Ficheros clave

```
./docker/ecosistema-fauna/
├── CLAUDE.md                          ← instrucciones para Claude (leer siempre)
├── docker-compose.yml
├── deploy-to-pro.sh                   ← ÚNICO método de deploy PRE→PRO
├── backend/
│   ├── main.py                        ← FastAPI: orquestador thin (329 líneas)
│   ├── db.py                          ← asyncpg + migraciones automáticas
│   ├── auth.py                        ← JWT, login, reset password
│   ├── admin.py                       ← panel admin: users, error_log
│   ├── routes_files.py                ← /upload /files /list /proxy/taxa
│   ├── routes_minka.py                ← /proxy/minka/* (publish, obs, species...)
│   ├── routes_inat.py                 ← /proxy/inat/* (publish, validate)
│   ├── vision_engine.py               ← thin re-exporter (79 líneas)
│   ├── vision_routes.py               ← endpoints /vision/* (locate/identify/detect...)
│   ├── vision_image.py                ← corrección/mejora de imágenes + endpoints
│   ├── vision_identify.py             ← pipeline iNat+Gemini+Groq+OpenRouter
│   ├── vision_detect.py               ← YOLO + fallback Groq Vision
│   ├── vision_exif.py                 ← extracción EXIF (GPS, fecha, modelo)
│   └── vision_http.py                 ← httpx singleton + JWT iNat + semáforos
├── webapp/
│   ├── public/                        ← Frontend PRO (NO editar directamente)
│   └── public-pre/                    ← Frontend PRE (TRABAJAR AQUÍ SIEMPRE)
│       ├── index.html                 ← Escritorio: CSS global + mount Vue
│       ├── PhotoGrid.js               ← Escritorio: componente Vue principal (~2631 líneas, refactorizado Fase 2)
│       ├── FaunaApp.js                ← Escritorio: shell de la app
│       ├── index-mobile.html          ← Móvil: app completa en un fichero (~3700 líneas)
│       ├── styles.css                 ← Estilos compartidos desktop
│       ├── recortar-utils.js          ← Utilidades de recorte compartidas
│       ├── PhotoGrid.js               ← Grid de fotos desktop
│       └── composables/
│           ├── use-recortar.js        ← Editor recorte desktop (CropperJS, B/A)
│           ├── use-before-after.js    ← Slider before/after (canvas, drag, keep/revert) [Fase 2]
│           ├── use-file-loader.js     ← Carga de archivos (ZIP, EXIF, calidad) [Fase 2]
│           ├── use-minka.js           ← Panel Minka (obs, species, projects)
│           ├── use-ubicacion.js       ← Leaflet + puntos de ubicación
│           ├── use-identificacion.js  ← Identificación de especie
│           ├── use-photo-actions.js   ← Acciones sobre fotos
│           ├── use-filtros.js         ← Filtros de imagen (CSS + canvas)
│           └── use-batch-filtros.js   ← Filtros en lote
├── docs/
│   ├── arquitectura_actual.md         ← este fichero
│   ├── interfaces_y_logica_aplicacion.md
│   └── backlog_mejoras_ui_ux.md
├── admin/                             ← Panel admin web
├── fauna-tmp/                         ← recortes YOLO por sesión (purga 24h)
└── temp/                              ← thumbnails efímeros
```

---

## Pipeline de visión IA

```
Fase 1 — Localizar  (POST /vision/detect, batch 2)
   YOLOv8n (CPU local)
   Detecta organismos → bbox, label, EXIF (GPS + fecha)

Fase 2 — Recortar   (canvas local, sin red)
   Recorte del bbox con padding, mejora contraste/saturación

Fase 3 — Identificar (incluido en /vision/detect)
   Motor 1: iNaturalist CV (score_image) — principal
            Requiere JWT renovado cada ~30 días
            GPS por defecto: lat=41.82 lng=3.06 si no hay EXIF
   Motor 2: Google Gemini — fallback si iNat score bajo
   Motor 3: Groq Llama — validación taxonómica

   Estrategia adaptativa:
     • confianza ≥ 0.50 → devuelve inmediatamente
     • confianza < 0.50 → espera hasta 8s más
     • Timeout global: 30s

Fase 4 (cliente) — GPS EXIF client-side
   Móvil: parseado en JS puro antes de enviar al servidor
   Desktop: parseado en _extractExifClientSide() en lote
   Si GPS detectado en cliente: no se muestra diálogo de ubicación
```

### Filtros IA acumulativos — Pipeline completo

Los filtros se aplican siempre **sobre `file_original`** en cadena (orden definido en `FILTER_DEFS`). Activar/desactivar cualquiera recalcula desde el original. El frontend cancela pipelines en curso via `AbortController`.

#### Definición (`use-filtros.js` / `FILTER_DEFS`)

| key | Icono | Endpoint | Algoritmo backend | Exclusión |
| --- | --- | --- | --- | --- |
| `sharpen` | 🔍 Nitidez | `POST /vision/sharpen` | Unsharp Mask sobre canal L (LAB). Parámetro `strength` 0-1. | — |
| `particles` | 💨 Partículas | `POST /vision/dehaze` | UDCP (Underwater Dark Channel Prior) + CLAHE LAB. | — |
| `underexp` | 🌙 Subexp. | `POST /vision/auto-enhance?mode=ai` | Fusión Mertens (exposure fusion multi-exposición sintética) + CLAHE LAB clip=2.5. Para subexposición severa (dark ratio >20% y avgLum <90). | `exposure` |
| `overexp` | ☀️ Sobreexp. | `POST /vision/auto-enhance?mode=fast` | CLAHE LAB clip=2.0-2.5 + boost de saturación (entorno marino). Para quemados >6%. | `exposure` |
| `contrast` | ◑ Contraste | `POST /vision/auto-enhance?mode=contrast` | Estiramiento de histograma percentil 1-99% en canal L (LAB) + CLAHE leve clip=1.5. Para histogramas comprimidos (stdLum <38, imagen ni oscura ni quemada). | `exposure` |
| `marine` | 🌊 Marina | `POST /vision/marine-correct` | Gray-world adaptativo (R `strength×0,82`, G `strength×0,3`, B↓ `strength×0,4`) + CLAHE L + unsharp suave. | — |
| `reds` | 🔴 Rojos | `POST /vision/reduce-reds` | Reducción de dominante roja (refracción de agua / luz artificial). | — |

**Grupo `exposure`**: `underexp`, `overexp` y `contrast` son **mutuamente excluyentes** — activar uno desactiva los demás automáticamente (campo `exclusionGroup` en `FILTER_DEFS`).

#### Detección automática (análisis de píxeles client-side, sin backend)

`_crop2AnalyzeHints()` en `use-recortar.js` muestrea 120×120 px de la imagen cargada y calcula:

- `darkRatio` — fracción de píxeles con luminancia < 40
- `burnedRatio` — fracción con luminancia > 230
- `avgLum` — luminancia media (Rec. 601)
- `stdLum` — desviación estándar de luminancia (indicador de contraste global)

Umbrales de detección:

- **Subexp**: `darkRatio > 0.20` y `avgLum < 90` → auto-aplica `underexp`
- **Sobreexp**: `burnedRatio > 0.06` → auto-aplica `overexp`
- **Bajo contraste**: `stdLum < 38` y `avgLum` en rango 50-200 y sin exceso de oscuros/quemados → **sugiere** `contrast` (ámbar pulsante, no aplica solo)
- **Nitidez**: calculada con `_crop2AnalyzePixels` (Laplaciano) → auto-aplica `sharpen`

Los filtros marcados como `AUTO_APPLY_KEYS` (`sharpen`, `underexp`, `overexp`) se aplican directamente. El resto se **sugiere** visualmente (botón ámbar con animación `filtro-suggest`) para que el usuario decida.

#### Endpoints de visión adicionales

- `POST /vision/dehaze` — eliminación de partículas/bruma (UDCP)
- `POST /vision/dehaze/score` — puntuación de bruma 0-1
- `POST /vision/marine-correct` — corrección dominante azul/verde marino
- `POST /vision/auto-enhance` — mejora automática con `mode`: `fast` | `ai` | `contrast`
- `POST /vision/sharpen` — nitidez con `strength` 0-1
- `POST /vision/reduce-reds` — reducción dominante roja con `strength` 0-1

---

## GPS EXIF — Parsing client-side

**Móvil** (`index-mobile.html`, función `readGpsFromFile`):
- Parser TIFF/EXIF en JS puro, sin librería externa.
- Lee tags GPS (lat/lon en DMS rational), convierte a DD.
- Llamado en `addFiles()` **antes** de decidir mostrar el diálogo de ubicación.
- Si alguna foto tiene GPS → se cargan sin preguntar ubicación.
- Si ninguna tiene GPS → se muestra el diálogo `locGateVisible`.

**Desktop** (`PhotoGrid.js`, función `_extractExifClientSide`):
- No muestra diálogo bloqueante, solo sugiere abrir panel de ubicación.
- GPS se extrae en lote de 20 antes de mostrar las fotos.

**Bug conocido corregido (2026-05-07):** El parser usaba `t.valOff - tiffStart` como base para valores inline, causando doble resta del offset. Fix: usar `t.valOff` directamente.

---

## iNaturalist — Autenticación JWT

```
INAT_ENABLED=1
INAT_EMAIL=contact@yespi.es
INAT_PASSWORD=<ver (configure via environment)>
INAT_API_TOKEN=<JWT actual, ver (configure via environment)>
```

- JWT manual caduca cada ~30 días.
- Renovación automática: `/mnt/scripts/fauna/renew-inat-token.sh` (cron día 10 de cada mes, 04:00h).

---

## Base de datos (PostgreSQL)

Contenedor: `postgres-global` (compartido). DB: `fauna`.

Tablas principales:
- `users` — usuarios registrados, flag `is_admin`
- `error_log` — errores registrados desde el frontend (tipo, mensaje, contexto JSON)
- `location_points` — ubicaciones permanentes guardadas por usuario

Acceso directo:
```bash
docker exec postgres-global psql -U admin_yespi -d fauna -c "SELECT ..."
```

---

## Panel de administración

URL: `https://fotofauna.yespi.es/admin/`
- Login con cuenta de usuario `is_admin=true` (contact@yespi.es)
- Secciones: usuarios, errores registrados (`error_log`), estadísticas
- Los errores del cliente mobile se registran vía `logError()` → `POST /admin/errors`

---

## Cloudflare Tunnel

Config: `/mnt/cloudflare/config.yml` | Servicio: `cloudflared.service` (systemd)

```yaml
- hostname: fotofauna.yespi.es
  service: http://127.0.0.1:3005
```

---

## Comandos de operación

```bash
# Estado y logs
docker ps | grep fauna
docker logs -f fauna_api --tail 50

# Verificar endpoints
curl http://localhost:3005/vision/health

# Reiniciar (cambios Python, sin cambiar compose)
docker restart fauna_api

# Recrear (si cambia docker-compose.yml)
cd ./docker/ecosistema-fauna && docker compose up -d --force-recreate

# Deploy PRE → PRO (ejecutar desde ./docker/ecosistema-fauna)
bash deploy-to-pro.sh

# Verificar sintaxis JS antes de guardar
node --input-type=module --check < webapp/public-pre/PhotoGrid.js

# Ver errores registrados en BD
docker exec postgres-global psql -U admin_yespi -d fauna \
  -c "SELECT error_type, message, created_at FROM error_log ORDER BY created_at DESC LIMIT 20;"
```

---

## Changelog técnico

### 2026-05-07 — Sesión móvil (bugs GPS + editor)
- **GPS parsing fix:** `readAscii` y `readRational` usaban `t.valOff - tiffStart` causando doble resta. Fix: usar `t.valOff` directamente.
- **GPS reactividad:** `photo.gps = gps` no activaba Vue 3 reactivity. Fix: reemplazo inmutable `photos.value[idx] = { ...photo, gps }` en `_doAddFiles`, `applyLocation`, `applyLocName`, `applyStatus`.
- **GPS antes del diálogo:** `addFiles()` ahora parsea GPS de todas las fotos antes de decidir mostrar `locGateVisible`. Si alguna tiene GPS, se cargan sin preguntar.
- **Editor crop `dragMode`:** Cambiado de `'move'` a `'crop'` — la imagen ya no se desplaza al arrastrar, sino que se dibuja el recuadro de recorte.
- **`cropGroupNav` y `cropNavPrev/Next`:** Esperan `img.onload` antes de llamar `initCropper()`, evitando inicialización sobre imagen incompleta.
- **`_closeSheetInternal`:** `_cropper.destroy()` movido a `.finally()` del autoguardado async, evitando destruir el cropper antes de que termine el guardado.
- **Botón Guardar eliminado del editor móvil:** El guardado es automático al navegar o cerrar. No existe botón manual de guardar en el editor de recorte.
- **Nuevos botones editor móvil:** ◑B/A (Before/After toggle), 🔍Nitidez, ☀Exposición, 🎯IA bbox, menú ⋮ expandible.
- **Nuevos sliders móvil:** Shadows, Highlights, Vibrance (antes solo: Exp, Bril, Cont, Sat, Temp).
- **`cropMoreOpen` ref:** Declarado, expuesto y reseteado en `initCropper()`.

### 2026-04-29 — Seguridad endpoints
- `POST /upload`, `GET /files/{filename}`, `GET /list`: añadido `_current_user` dependency.

### 2026-04-08 — Sesión 5 PRE
- Deduplicación al arrastrar imágenes.
- Mapa EXIF sidebar con Leaflet.
- Editor recorte: presets de proporción, indicador de ratio, mover con botón derecho.
- Recorte2: nueva interfaz modal full-screen en desktop.
