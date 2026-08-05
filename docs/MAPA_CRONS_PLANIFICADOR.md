# Mapa de crons FF/BQ ↔ Planificador de tareas (SCHED)

Estado de los crons de `/etc/cron.d/hansolo-fauna` respecto al planificador de tareas
(`scheduler/` en el backend). Actualizado 2026-06-24.

**Regla:** un cron migrado se **comenta** en `/etc/cron.d/hansolo-fauna` (no corre en
dos sitios). Los del **sistema** (backup, DNS, mantenimiento) NO se migran — no son de
FF/BQ. Los que **tocan el host** (`.env`, docker compose, Playwright, root) NO son
migrables in-process y se quedan como cron a propósito.

## ✅ Migrados al planificador (cron comentado)
| Cron original | Handler | Frecuencia |
|---|---|---|
| `calc_badges_cron.sh` | `calc_badges` | 1h |
| `autoid-wave.sh` | `autoid_wave` | 1h |
| `ff_sync_and_badges_daily.sh` | `sync_and_badges` | diario |
| `bq-obs-taxon-audit.sh` | `obs_taxon_audit` | diario |
| (nuevo) | `download_observations` | diario |

## ✅ Pipeline nocturno unificado (2026-07-02)

Handler **`academy_sync_nightly`** (#18) absorbe en un solo paso (orden):

1. `research_vessels_sync` (antes #20)
2. `bq_sync_nightly` — obs iNat+Minka
3. `refresh_common_names` + `prefill_species_docs`
4. **`edu_favicon_refresh`** — favicons pendientes (voluntariado + edu_sites)
5. **`edu_papers_feed`** — buffer circular artículos (+5/día, máx. 30, traducción ES)
6. `realm_snap_backfill` — lote nocturno (antes #29)
7. `recompute_academy_cache` (antes #19)

Tareas #19, #20, #29 se archivan automáticamente (`academy/academy_nightly.py`).

**`volunteer_events_refresh`** (#27, cada 12 h): crawl + hasta 25 favicons nuevos por pasada.

Favicon shell `edu-volunteer-crawl.sh` también repesca favicons al terminar (compatibilidad cron host).

## ✅ Migrados SCHED-04 (tarea recurring activa; `bq-academy-nightly` comentado + paso quitado de `fauna-nightly`)
| Cron / paso | Handler | Frecuencia | Notas |
|---|---|---|---|
| `bq-academy-sync-nightly.sh` (`bq_sync_nightly.py`) | `academy_sync_nightly` | diario | sync nocturno catálogo |
| `bq-recompute-academy-cache.sh` | `recompute_academy_cache` | diario | recalcula caché Atlas |
| `bq-research-vessels-sync.sh` | `research_vessels_sync` | diario | buques oceanográficos |
| `bq-iucn-update.sh` | `iucn_update` | semanal | estado IUCN + amenazas (probado: 222s) |

> Los 4 eran los únicos pasos de `bq-academy-nightly.sh` → ese cron se comentó entero, y se desactivó su invocación dentro de `fauna-nightly.sh` (paso `academy-data`) para evitar duplicación.

## ✅ Migrados SCHED-04 (2ª tanda — redundantes, comentados)
- `fauna-users-daily.sh` — sus 2 pasos (`ff-sync-badges`, `calc-badges`) ya estaban en el planificador (`sync_and_badges` + `calc_badges`) → cron redundante, comentado entero.
- `ff_sync_and_badges_midweek.sh` — es un **alias** de `fauna-users-daily.sh` → también redundante, comentado.

**BALANCE:** migrados los crons de **valor real** (sincronización de datos: medallas, obs, sync, audit, academy-sync, recompute, vessels, iucn). Lo que queda son tareas que **no conviene migrar** (ver abajo): host/root/node/Playwright (no posible in-process) o descargas de media masivas (mejor cron nocturno, bajo valor de observabilidad). Migrar más sería completismo, no valor.

## ⏳ Migrables pero de BAJO valor (dejados como cron por ahora)
- `bq-academy-media-nightly.sh` — `docker exec python3 /app/bq_download_academy_thumbs.py` + precache + rotate (3 scripts). Descargas masivas de thumbs; migrable pero mejor cron nocturno.
- `edu-volunteer-crawl.sh` — python del **host** (venv `/mnt/utils/venvs/`); absorbido por `volunteer_events_refresh` #27 + favicons en #18.
- `bq-papers-precache-morning.sh` — **solo manual/bootstrap**; producción en `academy_sync_nightly` #18 paso 5b (cron host 4×/día comentado 2026-07-02).
- pasos sueltos de `fauna-weekly`/`bq-academy-weekly` (minka-taxa-sync, WDPA, invasive) — entremezclados con pasos node/host.

## ❌ NO migrables (se quedan como cron a propósito)
- `renew-inat-token.sh` — toca `.env` del host + `docker compose` (orquestación host)
- `bq-ai-warm-cron.sh` / `bq-playwright-smoke-hourly.sh` — Playwright en el host
- `cleanup-fauna-tmp.sh` — corre como **root**
- `img-gemini-autogen.sh` — sondea cuota Gemini en host
- `scheduler-watchdog.sh` — **debe** ser cron: es la contingencia que revive el planificador si el tick muere

## 🖥️ Crons del SISTEMA (NO son FF/BQ — fuera del planificador)
`hansolo-backup`, `hansolo-dns`, `hansolo-maintenance`, `hansolo-sync-suite`,
`anacron`, `e2scrub_all`. Infraestructura del servidor.
