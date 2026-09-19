# FotoFauna — `fotofauna.yespi.es`

Subes fotos de fauna, la IA recorta e identifica la especie y las publica como observaciones en Minka.

| | |
|---|---|
| **Web** | https://fotofauna.yespi.es/ (PRO) · `/pre/` (PRE) |
| **Código** | `./docker/ecosistema-fauna/` |
| **Identificador propio** | `./docker/fotofauna-yolo/` (YOLOFauna) |
| **Backend** | `fauna_api` :3005 — sirve API y frontend estático |
| **Tareas** | [`TAREAS_PENDIENTES.md`](../../TAREAS_PENDIENTES.md) · [`TAREAS_FINALIZADAS.md`](../../TAREAS_FINALIZADAS.md) |

**Flujo:** editar en `webapp/public-pre/` → `bump-version.sh` → `deploy-to-pro.sh` → verificar (`docker ps` / `curl`) → push a GitHub. Nunca editar `public/` a mano.

## Índice

### Arquitectura y referencia
| Doc | Qué es |
|-----|--------|
| [`ARQUITECTURA.md`](ARQUITECTURA.md) | Contenedor único, PRE/PRO, clientes escritorio y móvil, diagrama de servicios |
| [`INTERFACES_Y_LOGICA.md`](INTERFACES_Y_LOGICA.md) | Pantallas, composables y lógica de la aplicación |
| [`BACKEND_PRE_PRO_SPLIT.md`](BACKEND_PRE_PRO_SPLIT.md) | Separación de backend PRE/PRO (compartida con BioQuest) |
| [`PLAN_UNIFICACION_TAXON_IDS.md`](PLAN_UNIFICACION_TAXON_IDS.md) | Regla canónica de taxon IDs entre iNat y Minka |
| [`MAPA_CRONS_PLANIFICADOR.md`](MAPA_CRONS_PLANIFICADOR.md) | Crons `/etc/cron.d/hansolo-fauna` ↔ planificador de tareas |

### YOLOFauna (identificador propio)
| Doc | Qué es |
|-----|--------|
| [`YOLOFAUNA.md`](YOLOFAUNA.md) | Arquitectura, dataset, calibración y roadmap |
| [`HANDOFF_YOLOFAUNA_FASE2.md`](HANDOFF_YOLOFAUNA_FASE2.md) | Traspaso de fase 2 para otra IA. §6: tareas de arranque |
| [`PROMPT_DEEPSEEK.md`](PROMPT_DEEPSEEK.md) | Prompt copiable de arranque para DeepSeek |

### Producto y calidad
| Doc | Qué es |
|-----|--------|
| [`CHANGELOG.md`](CHANGELOG.md) | Changelog rolling por sesión (jun-2026 →) |
| [`BIOFAUNA_FOTOS.md`](BIOFAUNA_FOTOS.md) | Admin bulk download of the BioFauna photo catalog |
| [`FOTOFAUNA_ANALISIS.md`](FOTOFAUNA_ANALISIS.md) | Auditoría técnica 2026-08-02. **Abiertos: T9 y T11**; el resto (T1–T8, T10, T12) ya está aplicado — leer la cabecera antes de tocar nada |
| [`FOTOFAUNA_ANALISIS_YESPI.md`](FOTOFAUNA_ANALISIS_YESPI.md) | Análisis UX/producto: 20 mejoras priorizadas |
| [`GRUPOS_IDENTIFICACION.md`](GRUPOS_IDENTIFICACION.md) | Política de identificación en grupos + reglas «NO reintroducir» |
| [`FILTROS_ORGANIZACION_2026-06-13.md`](FILTROS_ORGANIZACION_2026-06-13.md) | Barra de filtros IA del recorte |
| [`QA_CROP_FILTERS.md`](QA_CROP_FILTERS.md) | QA E2E Playwright: recorte + filtros, selectores canónicos |

## Notas

- `/mnt/docs/fotofauna` y `/mnt/docs/ecosistema-fauna` son **symlinks** a esta carpeta: los crons de QA escriben ahí con rutas fijas. Sus resultados (`qa_*.json`, `audit/`, …) están en `.gitignore`.
- Backlogs de auditorías de mayo–junio 2026 (móvil, escritorio, regresiones): [`_archive/2026-05-06-fotofauna-backlogs/`](../../_archive/2026-05-06-fotofauna-backlogs/).
- **FMINKA** (predecesor orquestado con n8n): [`_archive/historico/FMINKA_DOCUMENTACION.md`](../../_archive/historico/FMINKA_DOCUMENTACION.md).
