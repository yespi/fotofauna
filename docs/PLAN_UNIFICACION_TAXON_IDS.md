# Plan de unificación de Taxon IDs (FF-TAXON-00)

**Estado:** F0 en curso (2026-06-20) · **Prioridad:** P0 (junto a identificación FF)

## Regla canónica

- **`inat_id` = clave primaria** en todo el sistema.
- **`minka_taxon_id` = campo derivado**; nunca intercambiable con `inat_id`.
- **Resolución por nombre: solo match binomial exacto.** Prohibido el fallback al primer hit de autocomplete (`candidates[0]`).
- **Obs Minka sin equivalente iNat resoluble:** se conserva con su `minka_taxon_id` y flag **`inat_unresolved`** (no se pierde dato). El nombre Minka se usa en UI mientras tanto; se resuelve en F3. *(Decisión Gustavo 2026-06-20.)*

## Áreas afectadas (40+ puntos)

| Área | Archivos clave | Riesgo |
|------|----------------|--------|
| Subida Minka/iNat | `use-minka-payload.js`, `group-species.js`, `routes_minka.py`, `routes_inat.py` | Alto (resolve Minka) |
| UI identificación | `use-identificacion.js`, `routes_files.py` (proxy taxa) | Medio |
| Auto-ID UI principal | `use-vision-pipeline.js`, `vision_routes.py`, `vision_identify.py` | Bajo–Medio |
| Auto-ID admin Minka | `autoid.html`, `routes_minka_autoid.py` | Alto |
| Descargas Atlas | `bq_download_inat.py` ✓, `bq_download_minka.py`, `bq_obs_taxon_audit.py` | Bajo (patrón gold en iNat) |
| Alta especies Atlas | `catalog_extra.py`, `species_registry.py`, `catalog_meta.py` | Bajo–Medio |
| FaunaDex sync | `routes_sync.py`, `routes_pokedex.py` | Medio |
| Mapa «También en esta zona» | `pokedex.html`, `_zone_species_local` | **Alto — iNat ID enviado a API Minka** |
| Extra | `bq_precache_all_thumbs.py`, `calc_badges.py` | Alto — tratan IDs Minka como iNat |
| Extra | `scripts/bq-download-observations.py` (legacy) | Alto — deprecar |
| Extra | `academy.html` (subset TARGET_SPECIES duplicado) | Medio — puede desincronizar |

## Fases

| Fase | Contenido | Estimación |
|------|-----------|-----------|
| **F0** | Contrato `TaxonRef` + tests homónimos + bloquear script legacy | 1–2 días |
| **F1** | Fixes alto riesgo: FaunaDex zona, resolve Minka, precache, autoid | 3–5 días |
| **F2** | Catálogo 100% en DB (sin núcleo estático en Python) | 1–2 sem |
| **F3** | Mapear `user_observations` Minka→iNat + política 1900 taxones | 2–3 sem |
| **F4** | Audit nightly + tests CI continuos | continuo |

**Primera entrega esta semana:** F0 + FF-TAXON-01 (FaunaDex zona) + FF-TAXON-02 (sin fallback en resolve Minka).

### Diagnóstico F3 / FF-MINKA-01 (2026-06-22)

Estado real del mapeo Minka→iNat medido en DB (`minka_taxa_registry`: 10.394 taxones, solo 352 con `inat_id` = 3,4%). **De los 1.385 taxones que aparecen en observaciones de usuario:**

| Situación | Nº | Acción F3 |
|-----------|----|-----------|
| Ya mapeados a iNat | 287 | — |
| Sin mapear, **en** registry (tienen nombre científico) | 680 | resolver binomial contra **API iNat** (no hay match local: 0/680 comparten name_norm con filas ya resueltas → **obligatorio llamar a iNat**) |
| Ni en registry | 418 | requieren **sync Minka** del taxón antes de mapear |

⚠ **F3 NO es local**: el mapeo exige llamar a la API de iNat por cada binomial (rate-limited → oleadas, patrón `bq-conservation-import`). Hay que decidir política antes de lanzarlo (¿confianza mínima de match?, ¿qué hacer con los 418 fuera de registry?, los "1900 taxones"). La infraestructura ya existe: `academy/minka_taxa_registry.py::map_inat_id()` escribe el mapeo con `inat_match_source`/`inat_match_confidence`. **Pendiente de decisión de Gustavo** (proceso lento + cuota API).

## Progreso

- ✅ **FF-TAXON-02** — `routes_minka.py::_minka_resolve_taxon_id`: eliminado el fallback `pick = exact or candidates[0]`. Ahora sin match binomial exacto → `None` + log `[FF-TAXON-02]` para auditoría. PRE+PRO. (2026-06-20)
- ✅ **FF-TAXON-01** — `routes_pokedex.py::_photo_inat_id`: ya NUNCA usa el id Minka como iNat. Se añadió `id_source` ('minka'|'inat') en `_taxon_card` (Minka) y `_zone_species_local` (local). Cubre también `species/mine`. PRE+PRO. (2026-06-20)
- ✅ **F0** — contrato `academy/taxon_ref.py::TaxonRef` (inmutable, `inat_id` primario, `minka_taxon_id` derivado, `inat_unresolved`, `photo_inat_id()` que nunca devuelve Minka). Tests `tests/test_taxon_ref.py` (14 checks, homónimos incluidos) — pasan dentro del contenedor. `_photo_inat_id` refactorizado para usar `TaxonRef` (equivalencia verificada). (2026-06-20)
  - Ejecutar tests: `docker exec fauna_api_pre python tests/test_taxon_ref.py`
- ✅ **F1 — COMPLETA (2026-06-22)**. Revisados los 3 cabos de alto riesgo:
  - `bq_precache_all_thumbs.py` → ya estaba corregido (`_user_taxon_ids` mapea minka_taxon_id→inat_id vía `academy_catalog_meta`, descarta no resolubles). Verificado.
  - `calc_badges.py` (`/mnt/scripts/fauna/`) → ya estaba corregido (traduce IDs Minka→iNat antes de resolver ancestros). Verificado.
  - `routes_minka_autoid.py::_resolve_minka_taxon` → **corregido ahora**: eliminado el fallback `items[0]` (mismo patrón prohibido que FF-TAXON-02 en `routes_minka.py`). Sin match binomial exacto → `None` + log `[FF-TAXON-02]`. Importa `_norm_taxon_name` de `routes_minka` (normaliza acentos/caja). Este `taxon_id` se publica directo en Minka, así que era el más sensible. **Solo PRE** (api-pre reiniciado, healthy, import runtime OK). ⚠ PENDIENTE PRO con permiso explícito.
  - Barrido completo del backend: no quedan otros `candidates[0]`/`items[0]` que resuelvan taxon_id para publicación (los de `vision_identify.py:445` = solo thumbnail, y `routes_pokedex.py:803` = autocompletado de UI, son inocuos).
- 🔄 **F2 — PASO 1 hecho (2026-06-22, solo PRE): núcleo del catálogo en DB.**
  - Nueva tabla `academy_catalog_core` (185 sp) con el mismo modelo de datos que `TARGET_SPECIES` (inat_id PK, name, common, group, emoji, bbox, iucn_status, miteco_marine, invasive, base_group). Sembrada idempotente desde el estático (`ON CONFLICT DO NOTHING` → nunca pisa ediciones manuales en DB).
  - Módulo `academy/catalog_core.py`: `ensure_table` / `seed_from_static` / `load_core`. Enganchado en `main.py` startup ANTES de `load_extras`.
  - **Punto de unión mínimo**: `academy/common.py::merged_target_species()` ahora usa `_core_base()` = DB si cargada, **fallback a `TARGET_SPECIES` estático** si la tabla está vacía o la DB no responde. Los 28 ficheros que consumen `merged_target_species()`/`INAT_ID_TO_SP` heredan el cambio sin tocarse.
  - **Verificado**: round-trip estático↔DB↔reconstruido = 0 mismatches en 185 sp; arranque real loguea "185 cargadas desde DB"; edición en DB (emoji) se refleja tras reinicio; subprocesos sin pool caen al fallback estático sin romper. La lista Python sigue siendo SEMILLA + red de seguridad (NO se borra).
  - ✅ **F2 PASO 2 — backend del CRUD (2026-06-22, solo PRE)**. `catalog_core.update_species()` edita campos del núcleo y **recarga el catálogo en caliente** (`load_core` → `INAT_ID_TO_SP` consistente sin reinicio). Endpoints admin: `GET /admin/academy/catalog/core` (lista) y `POST /admin/academy/catalog/core/edit` (editar). Protegidos por `require_admin` (verificado: responden 401 sin auth). `update_species` probado end-to-end: editó `common` del lince y revirtió en memoria sin reiniciar. Campos editables: name/common/group/emoji/bbox/iucn_status/miteco_marine/invasive/base_group (NO inat_id = PK canónica).
  - ✅ **Coherencia dashboard**: `pipeline_stats.py` ahora calcula `catalog_core`/`miteco`/`catalog_inat_ids` desde `common._core_base()` (DB con fallback), no desde `len(TARGET_SPECIES)`. Verificado 185.
  - **Pendiente F2 PASO 3 (UI)**: añadir edición inline a la tabla "Gestión del catálogo" del panel admin (`webapp/public-pre/admin/index.html`, sección `academy-catalog-tbody`) usando los endpoints ya existentes. No hecho a ciegas: requiere validación visual/UX con Gustavo. El backend ya está listo y probado.
  - **Pendiente F2 PASO 4**: decidir si el estático pasa a solo-semilla-de-bootstrap (hoy es semilla + fallback de seguridad; mantenerlo así es lo más robusto). ⚠ Todo en PRE; PRO con permiso explícito.
- ⏳ **F4** — integrar `test_taxon_ref.py` en CI (ahora ejecutable standalone).
