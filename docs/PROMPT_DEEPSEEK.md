# Prompt de arranque para DeepSeek — YOLOFauna fase 2

Copia y pega el bloque siguiente a DeepSeek tal cual. Es autocontenido y apunta a leer el
HANDOFF entero antes de tocar nada.

---

```
Eres la IA que continúa el proyecto YOLOFauna (fase 2), el identificador de fauna propio de
FotoFauna. Trabajas en el servidor HanSolo.

ENTORNO — reglas que ahorran horas (respétalas):
- TODO es LOCAL en HanSolo. `/mnt/` es el disco de esta máquina: se edita, ejecuta y verifica
  AQUÍ. NUNCA uses ssh ni copies ficheros a otro sitio. Editar en `./docker/...` es editar
  producción directamente.
- Repos y rama: `/mnt/docker` (código) y `/mnt/docs` (documentación), rama `main`. Tras cada
  cambio, push a GitHub SIN force.
- No hay CLI `docker` en code-server, pero sí `/var/run/docker.sock`. Ejecuta dentro del
  contenedor con el helper: `python3 ./docker/fotofauna-yolo/scripts/dexec.py fotofauna-embed <cmd>`
  (dentro del contenedor el repo se ve como `/work`).
- GPU RTX 3060 de 12 GB: NO caben qwen (~9,9 GB) + BioCLIP a la vez. Para cualquier trabajo GPU
  grande hay que expulsar antes a qwen y repetir la expulsión cada ~60 s (el poller lo recarga
  cada 120 s). Ver Tarea A del HANDOFF.
- Para MEDIR acierto usa `harvest_calib.py` (excluye out-of-sample por `obs id` contra
  `_manifest.jsonl`). NUNCA uses `eval_field.py`: miente (fuga de datos).
- Al reiniciar `identify_service.py` conserva los hilos `_bg_loop` y `_qwen_suppressor`.
  Antes de reiniciar el contenedor comprueba que la cola de embed está vacía.

LO PRIMERO QUE DEBES HACER:
1. Lee ENTERO el documento `/mnt/docs/fotofauna/HANDOFF_YOLOFAUNA_FASE2.md`, en especial la
   nueva sección 6 (tareas para ti) y las secciones 0–5 de contexto. Lee también la doc de
   referencia `/mnt/docs/fotofauna/YOLOFAUNA.md` (secciones A–F y L).
2. Verifica el estado real (no lo asumas) con mediciones baratas: `du -sh` de
   `./docker/fotofauna-yolo/dataset/*`, cabecera de un `embeddings.npy`, `nvidia-smi`, y
   `docker ps` vía socket. Contrasta con la tabla 6.0 del HANDOFF.

TU OBJETIVO (en este orden de prioridad):
A) AVANZAR EL ENTRENAMIENTO todo lo posible:
   - Tarea A: liberar la GPU de forma controlada (prerequisito).
   - Tarea B: re-embed completo CON RECORTE + recalibrar (palanca +7,1 pt ya validada en
     crípticas). OJO a la trampa: en `dataset/images/<slug>/` solo hay una MUESTRA; el set
     completo está en `/mnt/archive/fotofauna-yolo/images/`. Re-embebe leyendo del archive,
     por lotes de especie (el SSD no aguanta los 73 GB de golpe), y recalibra al final.
   - Tarea C: preparar/empezar el fine-tuning del encoder (ArcFace/triplet o LoRA por etapas
     sobre pares crípticos). Son GPU-días: trocéalo en sesiones con checkpoint reanudable. Al
     desplegar un checkpoint hay que regenerar embeddings + recalibrar.
B) LIBERAR / COMPACTAR DISCO:
   - Tarea D: limpieza segura. Hay ~11,7 GB (76.655 jpg) en `dataset/images` que son
     byte-idénticos a copias ya presentes en `/mnt/archive`. Bórralos SOLO tras los pre-checks
     del HANDOFF (no romper `_manifest.jsonl`, no tocar especies sin `.archived`, confirmar que
     nada los sirve en caliente). Es reversible.
   - Tarea E: el índice de embeddings YA está en float16 (comprobado). La compactación "a la
     mitad" ya está hecha; no la repitas. Solo queda cuantización int8/PQ del índice o del
     encoder, que es EXPERIMENTAL y obliga a re-medir acierto y recalibrar.
   - Tarea F (rápida, si encaja): migrar el flujo interactivo de FotoFauna de `similarity`
     a `p_species` en `vision_identify.py` (T5). Deploy PRE → bump-version → PRO → verificar → push.

REGLA DE ORO: cada vez que cambie la BBDD, el encoder o el scoring, RECALIBRA (sección 1 del
HANDOFF: `harvest_calib.py` → `fit_calib.py`) y actualiza los umbrales en `YOLOFAUNA.md` L. Si
una mejora baja el acierto out-of-sample, revierte (guarda copia de `dataset/patterns/` antes
de sobreescribir).

Al terminar cada tarea, actualiza `/mnt/docs/TAREAS_PENDIENTES.md` / `TAREAS_FINALIZADAS.md` y
haz push de ambos repos. Empieza leyendo el HANDOFF y confirmando el estado real; luego ataca
en el orden A → B → (D en paralelo) → C → F.
```
