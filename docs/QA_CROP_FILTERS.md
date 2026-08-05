# QA FotoFauna — E2E Playwright (recorte, filtros, marathon)

Herramientas de QA automatizado de FotoFauna (y sondas BQ asociadas). Los scripts viven en
`./docker/ecosistema-fauna/backend/tests/` y `/mnt/scripts/fauna/`.

## Auditoría recorte + filtros

Carga fotos, recorte 1:1, todos los filtros IA, antipartículas. Captura errores de consola, `pageerror` y HTTP ≥400.

```bash
# FF desktop PRE + PRO (hooks QA + consola)
cd ./docker/ecosistema-fauna/backend/tests
source .venv-e2e/bin/activate   # pip install -r requirements-e2e.txt Pillow httpx
python3 e2e_crop_filters_full.py

# Solo PRE, clics reales en botones del panel recorte
FF_TARGETS=pre python3 e2e_crop_filters_full.py --ui-filters

# Todas las interfaces FF + BQ
/mnt/scripts/fauna/qa-all-interfaces-audit.sh
```

### Filtros cubiertos

| key | Etiqueta UI |
|-----|-------------|
| enhance | Auto ✨ |
| sharpen | Nitidez |
| deblur | Enfocar |
| dehaze | Bruma |
| underexp | Subexp. |
| overexp | Sobreexp. |
| contrast | Contraste |
| marine | Marina |
| reds | Rojos |
| (stamp) | Antipartículas 💨 |

### Hooks QA (`?qa=1`, desktop `?desktop=1&qa=1`)

`window.__ffQa` en PhotoGrid.js — ver código para API completa. Destacados:

- `applyCropByName(name, ratioLabel)` — recorte programático
- `clickFilterInCrop(name, filterKey)` — clic en botón filtro real
- `applyFilterToName(name, key)` — toggle filtro vía API/hook

Flags: `__ffQaSkipQuality` (no auto-papelera en import QA).

### Ruido ignorado (no es bug)

- Google Analytics / AdGuard (`ERR_SSL_UNRECOGNIZED_NAME_ALERT`)
- `auth/refresh` guest 200
- `ERR_FILE_NOT_FOUND` en blob URLs revocados al cerrar editor

### Interfaces

| App | Entornos | Script |
|-----|----------|--------|
| FotoFauna desktop | PRE `/pre/?desktop=1&qa=1`, PRO `/?desktop=1&qa=1` | `e2e_crop_filters_full.py` |
| FotoFauna móvil | PRE/PRO `index-mobile.html` | `ff-full-ui-audit.mjs` |
| BioQuest | PRE/PRO Atlas + Academy | `bq-full-ui-audit.mjs`, `bq-boot-probe.mjs` |

Credenciales: `QA_FF_EMAIL` / `QA_FF_PASSWORD` en `(configure via environment)`.
Salida JSON: `backend/tests/crop-filter-runs/<run_id>/report.json`.

## Marathon QA (campaña larga)

Campaña automatizada ~6 h: flujo completo Playwright + informe HTML por correo.

```bash
bash ./docker/ecosistema-fauna/backend/tests/run-marathon-6h.sh
```

| Variable | Default | Descripción |
|----------|---------|-------------|
| `FF_MARATHON_HOURS` | 6 | Duración campaña |
| `FF_MARATHON_INTERVAL_SEC` | 180 | Pausa entre iteraciones (~3 min) |
| `FF_MARATHON_TO` | contact@yespi.es | Destinatario informe |
| `FF_MARATHON_SEND_EMAIL` | 1 | 0 = solo HTML local |

Una iteración manual:

```bash
cd ./docker/ecosistema-fauna/backend/tests
FF_MARATHON_RUN_ID=test1 .venv/bin/python e2e_fotofauna_marathon.py
```

Qué prueba cada iteración: arranque PRE + login OAuth simulado, limpiar sesión, cargar 8 JPEG
sintéticos, ubicación a todas, filtros IA + ajustes, identificar/quitar ID, duplicar/eliminar/agrupar,
antipartículas + guardar sesión, dry-run Minka, F5 + recargar mismos JPG → comparar manifiesto, logs.

Informes: `backend/tests/marathon-runs/<run_id>/report.json` + capturas;
campaña: `backend/tests/marathon-runs/campaign_<ts>/marathon_report.html`.

Variante continua multi-app (FF+BQ): `/mnt/scripts/fauna/qa-marathon-continuous.sh`
(env `QA_MARATHON_HOURS`, `QA_MARATHON_INTERVAL_SEC`). Los artefactos de la campaña de
2026-06-19 y su registro de bugs se retiraron de este repo (git history:
`QA_MARATHON_BUGS.md`, `qa-marathon-runs/`).

## Selectores canónicos (FF-B05, `ff-selectors.mjs`)

Referencia para los scripts Playwright (`/mnt/scripts/fauna/ff-selectors.mjs`, auditado con `ff-pg-grid-audit.mjs`):

| Clave | Selector |
|-------|----------|
| `grid` / `gridCell` | `.pg-grid` / `.pg-cell` (no `.pg-card`) |
| `gridEmpty` | `.pg-empty-hero` |
| `leftRail` / `sidebar` / `leftPanel` | `.pg-left-rail` / `.pg-sidebar` / `.pg-lpanel` |
| `locationMap` | `.pg-loc__map, .leaflet-container` (Leaflet inyecta `.leaflet-container` en runtime) |
| `sidebarMap` | `.pg-map-frame, .pg-map-locality` |
| `cropPanel` | `.c2-panel` (crop2) |
| `identifyResults` | `.pg-id__results` |
| `minkaBody` / `minkaTabs` | `.pg-minka__body` / `.pg-minka__tabs` |
| Móvil | `.m-gallery` / `.m-photo-grid` / `.m-bottom-nav` |

Lista completa (23 claves) en `ff-selectors.mjs`.
