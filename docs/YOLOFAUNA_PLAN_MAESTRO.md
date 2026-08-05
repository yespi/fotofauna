# YOLOFauna — PLAN DE ACCIÓN MAESTRO

> **Este es el documento único de referencia para seguir subiendo el acierto de YOLOFauna.**
> Sustituye como fuente de verdad operativa a las secciones "PLAN DE MEJORA" / "1bis" / "1ter"
> de `YOLOFAUNA.md` (que quedan como el *research log* que sustenta las decisiones de aquí —
> no se borran, pero para saber "qué hago ahora" se lee **este** fichero).
> Última actualización: **2026-08-04**. Nada de lo de aquí abajo está ejecutado todavía salvo
> lo marcado explícitamente como ✅.

---

## 0. Cómo usar este documento

1. Lee la **sección 1** (estado verificado) para no redescubrir nada.
2. Lee la **sección 2** (bloqueadores confirmados) — cambia el orden de prioridad de lo
   anotado ayer: el fine-tuning del encoder está bloqueado por **hardware**, no solo por
   una librería. Eso hace que la **sección 4 (inferencia de mínimo riesgo)** pase a ser lo
   primero a ejecutar, porque no toca el encoder ni necesita más VRAM.
3. Cada tarea tiene: objetivo, pasos, ficheros a tocar, criterio de aceptación y qué hacer
   si falla. Ejecuta en el **orden de la sección 9**, no salgas de orden salvo que algo se
   demuestre imposible (documentarlo aquí mismo si pasa).
4. Regla de oro de todo el proyecto, repetida porque es la que más veces se ha olvidado:
   **cualquier cambio en BBDD, encoder o scoring exige recalibrar** (`harvest_calib.py` →
   `fit_calib.py` → reiniciar) y **medir con `harvest_calib.py`, nunca con `eval_field.py`**
   (mide con fuga de datos, ver `YOLOFAUNA.md` sección L.0).

---

## 1. ESTADO VERIFICADO (no redescubrir)

| Cosa | Valor |
|------|-------|
| Especies con imágenes | **975** (antes 955) |
| Fotos totales | **520.594**, en SSD `/mnt/gpu/fotofauna-images/` |
| Técnica ganadora de embeddings | **Triplet loss sobre embeddings ya extraídos** (89 épocas, loss 0.025). Mejora el 99% de las especies. **No** toca los pesos de BioCLIP — es una transformación aprendida *encima* del embedding congelado |
| Acierto especie (out-of-sample, `harvest_calib.py`) | **63,9%** |
| Acierto género | 69,7% |
| Acierto familia | 74,8% |
| Muestra de calibración | 2.427 fotos, 972 especies |
| Precisión a p≥0,98 | 95,4% (23,9% cobertura) |
| Precisión a p≥0,90 | 94,5% (47,7% cobertura) |
| Crop (bbox) + triplet vs original + triplet | **Idénticos** en 1369 spp — el crop por caja NO aporta sobre el triplet solo |
| ProjHead (2,6M params) | 50,9% val — peor que triplet |
| Wave de auto-publicación en Minka | **Pausado** (publicaba erróneas en crípticas) |
| GPU | RTX 3060, 12 GB VRAM — única GPU disponible en local |

---

## 2. BLOQUEADORES CONFIRMADOS — leer antes de tocar el encoder

### 2.1. VRAM insuficiente para fine-tuning local del encoder (confirmado, no especulación)
- BioCLIP ViT-L (428M params) ocupa **9,6 GB** de VRAM solo con los pesos cargados.
- Con 12 GB totales quedan **2,4 GB libres** — insuficiente para activaciones + gradientes
  + estado del optimizador, **incluso con BATCH=2 y gradient checkpointing activado**
  (ambos probados, ambos insuficientes).
- **Se necesita ≥16 GB VRAM** para fine-tuning convencional (full o "descongelar último
  bloque") de un modelo de este tamaño con las técnicas ya probadas.

### 2.2. LoRA vía PEFT genérico: incompatible, confirmado con código (no solo teoría)
- `peft` (HuggingFace) se instaló y se probó: **el wrapper rompe el `forward()`** de
  BioCLIP/open_clip. No es un problema de "falta probar", es un intento real que falló.
- Esto **coincide** con lo encontrado en la investigación externa: `peft` tiene un *issue*
  abierto sin resolver desde 2023 pidiendo soporte nativo para OpenCLIP — el fallo no es
  casualidad de esta instalación, es una limitación conocida de la librería.

### 2.3. Lo que esto significa para el plan
Las dos vías "baratas" que se iban a intentar (LoRA genérico, descongelar el último bloque)
**están las dos cerradas** por motivos distintos (una por librería, otra por VRAM). El plan
de abajo (sección 3) da **tres vías nuevas no probadas todavía** que sí atacan la causa raíz
(memoria), en vez de repetir variantes de lo ya fallido.

---

## 3. ÁRBOL DE DECISIÓN — fine-tuning del encoder (la palanca de mayor techo)

No ejecutar esto antes que la sección 4 (que no requiere GPU extra ni resuelve nada de
memoria). Cuando se llegue aquí, seguir el árbol en orden — cada rama es más cara que la
anterior:

### 3.A. 🥇 QLoRA (LoRA + cuantización 4-bit del backbone) — NO PROBADO, probar primero
**Por qué no se probó ya:** lo que falló fue LoRA "normal" (adaptadores de bajo rango sobre
pesos en fp16/fp32 sin cuantizar) vía el wrapper de `peft`, que además rompía el forward.
QLoRA es distinto en dos ejes a la vez:
1. **Cuantiza el backbone congelado a 4-bit** (bitsandbytes NF4) → los 9,6 GB de pesos caen
   a **~2,5-3 GB**, liberando ~7 GB para activaciones/gradientes/optimizador de los
   adaptadores LoRA. Esto ataca directamente el bloqueo de la sección 2.1.
2. **No depende del wrapper de `peft` que rompe el forward** — se puede implementar LoRA a
   mano (insertar matrices de bajo rango en las proyecciones `q_proj`/`v_proj`/`out_proj`
   de los bloques de atención de open_clip directamente, sin pasar por el `AutoModel` de
   HF que es lo que rompía) o usar `clipora` (sección 3.B) como base y añadirle
   cuantización.
- **Pasos:**
  1. `pip install bitsandbytes` en el contenedor `fotofauna-embed` (verificar que la imagen
     base tiene CUDA compatible con bitsandbytes — comprobar versión de CUDA del contenedor
     antes: `nvidia-smi` / `torch.version.cuda`).
  2. Cargar el modelo open_clip de BioCLIP-2 con `load_in_4bit=True` (o cuantizar a mano
     los `nn.Linear` de los bloques de atención con `bitsandbytes.nn.Linear4bit`).
  3. Insertar adaptadores LoRA (rango 8-16 para empezar) **solo en los últimos 2-4 bloques**
     de la ViT-L — no hace falta LoRA en todo el backbone para un fine-tuning enfocado en
     pares crípticos.
  4. Entrenar con el dataset de pares crípticos (ver 3.D más abajo) con triplet/ArcFace.
  5. Medir VRAM real con `nvidia-smi` durante el entrenamiento — objetivo: quedar por
     debajo de 11 GB con margen para el resto del sistema.
- **Riesgo:** la cuantización a 4-bit introduce ruido en el forward que puede degradar la
  calidad de los embeddings del propio backbone (no solo de los adaptadores). Mitigar
  validando contra `harvest_calib.py` **antes de desplegar**, comparando con el baseline
  triplet actual (63,9%). Si baja, no desplegar — es una señal de que este backbone
  concreto no tolera 4-bit bien (pasar a 3.C).

### 3.B. 🥈 `clipora` — toolkit dedicado a LoRA sobre OpenCLIP (no genérico HF) — NO PROBADO
- Repo: `github.com/awilliamson10/clipora`. A diferencia de `peft`, está hecho
  específicamente para el formato de checkpoint de OpenCLIP (que es exactamente lo que usa
  BioCLIP-2: `open_clip_model.safetensors`), así que **no debería chocar con el mismo
  wrapper que rompió `peft`**.
- Estado del proyecto: activo pero incompleto (le faltan features de *merge* de adaptadores
  e inferencia productizada) — para desplegar en producción probablemente haga falta
  escribir a mano el paso de "fusionar LoRA con los pesos base" tras entrenar, o mantener
  los adaptadores como *hook* separado en `identify_service.py`.
- **No resuelve por sí solo el problema de VRAM** (sección 2.1) — LoRA reduce el estado del
  optimizador (Adam solo sobre los adaptadores, no sobre 428M params), pero el forward
  sigue necesitando los 9,6 GB del backbone en fp16/fp32. Si `clipora` sin cuantizar sigue
  dando OOM, combinar con 3.A (cuantizar el backbone que carga `clipora`).
- **Pasos:** clonar, adaptar su script de carga al checkpoint local de BioCLIP-2 (no
  descargar de HF de nuevo, usar el `.safetensors` ya cacheado en `hf_cache/`), probar
  primero con rango LoRA bajo (4-8) y batch=1 para ver si cabe sin cuantizar.

### 3.C. 🥉 Alquilar GPU cloud (Lambda Labs / RunPod) — fallback ya identificado, el más caro pero el más seguro
Si 3.A y 3.B no caben o degradan calidad, esta es la vía sin incertidumbre técnica:
- **RunPod** o **Lambda Labs**: instancia con GPU ≥24 GB (RTX 4090/A5000/A6000) por horas.
  Coste orientativo: ~0,3-0,8 $/h según proveedor y GPU — un fine-tuning de "GPU-días" bien
  troceado en checkpoints puede hacerse en pocas horas de alquiler puntual, no días
  continuos, si se prepara bien el dataset y el script **en local primero**.
- **Preparación antes de pagar por GPU cloud** (para no desperdiciar horas de alquiler):
  1. Tener el dataset de pares crípticos ya construido (sección 3.D) y subido a algún sitio
     accesible desde la instancia cloud (S3/HF datasets/rsync directo).
  2. Tener el script de entrenamiento **ya probado localmente** con un subconjunto pequeño
     (aunque sea con OOM en el modelo completo, se puede validar la lógica con un modelo
     dummy más pequeño o con `--max_steps 5` en CPU) para no depurar bugs de código con el
     medidor de coste corriendo.
  3. Checkpoints reanudables (`dataset/ft_ckpts/`) para poder parar/reanudar sesiones de
     alquiler cortas en vez de una sola sesión maratón.
  4. Al terminar: descargar el checkpoint final, **regenerar embeddings en local** (eso sí
     es barato, es solo inferencia) y recalibrar como siempre.
- **Decisión pendiente del usuario:** esto implica gasto real (no es gratis como el resto
  del plan). Antes de ejecutar esta rama, confirmarlo explícitamente — no es una acción
  reversible de coste cero como las demás.

### 3.D. Dataset de pares crípticos (prerequisito común a 3.A/3.B/3.C, hacerlo YA independientemente de qué rama se elija)
Gratis, no requiere GPU, se puede hacer mientras se decide/prueba 3.A-3.C:
- Sacar pares de `identifications[].category=="maverick"` / `identifications_most_disagree`
  de las respuestas de Minka (ya se consultan para el enriquecimiento taxonómico).
- Complementar con la matriz de confusión de `scripts/analyze_acc.py` (géneros como *Doto*,
  *Haminoea*, *Favorinus*, *Elysia*, *Aplysia* ya identificados como conflictivos).
- Guardar en `dataset/crypto_pairs.json`: lista de `(especie_a, especie_b, nº_confusiones)`.
- Este dataset es el que alimenta el triplet/ArcFace de cualquiera de las 3 ramas — no se
  tira trabajo si cambia la rama elegida.

### 3.E. Pista de investigación en paralelo — encoder alternativo más pequeño (DINOv3 ViT-S/B)
No es parte del árbol principal (cambiar de encoder es una decisión mayor, no un parche),
pero es la única vía que **evita el problema de VRAM desde la raíz** en vez de rodearlo:
- DINOv3 ViT-S (21M) / ViT-B (86M) son 5-20x más pequeños que BioCLIP ViT-L (428M) →
  fine-tuning completo (no solo LoRA) cabría de sobra en la 3060.
- **Trade-off honesto:** DINOv3 es self-supervised generalista (entrenado en 1,7B imágenes
  sin etiquetas), no tiene el conocimiento taxonómico específico que BioCLIP-2 adquirió de
  TreeOfLife-200M. Partiría de una base zero-shot peor, pero al ser mucho más barato de
  fine-tunear completo (no solo LoRA), podría alcanzar o superar a BioCLIP+triplet tras
  fine-tuning específico en los datos de FotoFauna.
- **Cómo probarlo sin comprometer el pipeline en producción:** experimento paralelo, no
  sustituye a BioCLIP-2 mientras no esté medido. Extraer embeddings DINOv3 de las mismas
  ~2.400 fotos de calibración, entrenar un fine-tune rápido (unas pocas horas, cabe en la
  3060), medir con la misma metodología de `harvest_calib.py` y comparar acierto contra el
  63,9% actual **antes** de considerar migrar el pipeline completo (que sería un cambio de
  520k embeddings, no trivial).
- Prioridad: **después** de intentar 3.A/3.B, como plan B de fondo si el fine-tuning de
  BioCLIP sigue sin caber ni con cuantización.

### Qué NO repetir en esta sección (ya falló, con evidencia)
- LoRA vía `peft` genérico sin cuantizar → rompe el forward.
- Descongelar el último bloque transformer sin cuantizar, BATCH=2 + gradient checkpointing
  → OOM.
- ProjHead (cabeza de proyección de 2,6M params) → 50,9% val, peor que triplet.
- MLP/cabeza lineal a escala completa (1257+ especies) → no bate al kNN.

---

## 4. TAREA INMEDIATA (hacerla YA, no depende de GPU ni de la sección 3) — inferencia de mínimo riesgo taxonómico

**Por qué va primero:** no toca el encoder, no necesita más VRAM, no depende de que 3.A/B/C
funcionen. Es un cambio en la **regla de decisión** sobre las salidas del kNN que ya existen
hoy. Puede ejecutarse esta misma sesión.

### 4.1. Qué es
Hoy la abstención especie→género→familia decide por un **margen fijo**
(`YOLOFAUNA_FAMILY_MARGIN=0.06`): si el top-1 y el top-2 del kNN son congéneres y su
diferencia de score es menor que el margen, se baja a género; si solo comparten familia, se
baja a familia. Es una heurística razonable pero arbitraria.

La alternativa (de un paper de clasificación taxonómica marina, arxiv 2606.25989) es una
**regla de decisión bayesiana de mínimo riesgo**: en vez de comparar solo el top-1 vs top-2,
mira **todos** los vecinos del kNN con su probabilidad calibrada (`p_species`), calcula para
cada posible nivel de respuesta (especie concreta / género / familia) el **coste esperado de
equivocarse** usando una matriz de distancia taxonómica, y elige el nivel que minimiza ese
coste esperado — no el de mayor score puntual.

### 4.2. Por qué debería ser mejor que el margen fijo
- El margen fijo solo mira 2 candidatos (top-1, top-2). La regla de mínimo riesgo usa **toda
  la distribución de vecinos del kNN** (k=25 ya calculado, no cuesta nada extra sacarlo).
- Un margen de 0,06 es igual de estricto para un caso con 3 especies empatadas que para uno
  con 25 vecinos todos de la misma familia pero especies distintas — la regla de riesgo
  distingue estos casos porque pesa por la distribución completa, no por 2 números.
- Se integra en el pipeline **sin regenerar embeddings ni recalibrar el modelo base** (la
  calibración de `p_species`/`p_genus`/`p_family` ya existe y se sigue usando tal cual, solo
  cambia cómo se combinan para decidir el nivel de respuesta).

### 4.3. Pasos concretos
1. **Matriz/función de distancia taxonómica.** No hace falta una matriz completa N×N: basta
   una función `dist(taxon_a, taxon_b)` con las reglas ya usadas en la abstención actual:
   `0` si misma especie, `1` si mismo género distinta especie, `2` si misma familia distinto
   género, `3` en cualquier otro caso. Los datos de género/familia ya están en
   `target_species.json` desde `enrich_taxonomy.py` (sección F de `YOLOFAUNA.md`) — no hay
   que volver a llamar a Minka.
2. **Riesgo esperado por nivel de respuesta**, usando los k=25 vecinos y sus pesos/votos
   (ya calculados por el kNN existente) como aproximación de la distribución:
   - Riesgo de responder "especie X" = Σ (peso_i × dist(X, especie_del_vecino_i))
   - Riesgo de responder "género G" = Σ (peso_i × dist(G, género_del_vecino_i)) — donde
     `dist` a nivel género colapsa la distancia especie→género a 0 si coincide el género.
   - Análogo para familia.
3. **Decisión:** elegir el nivel (especie del top-1 / género mayoritario / familia
   mayoritaria) que minimice el riesgo esperado, en vez de comparar el margen top-1/top-2.
   Mantener como salvaguarda un **suelo de confianza calibrada** (no responder nada por
   debajo de cierto `p` aunque el riesgo sea mínimo — para no regalar "familia" en fotos
   donde ni siquiera la familia es fiable).
4. **Dónde tocar:** `identify_service.py`, en la función que hoy implementa la abstención
   por `YOLOFAUNA_FAMILY_MARGIN` (buscar esa constante en el fichero). Sustituir/complementar
   esa lógica por la de riesgo esperado. **No tocar** `load_calib()`/`_calibrate()` — la
   calibración de `p_species` etc. se mantiene igual, solo cambia cómo se usa después.
5. **Medir antes de desplegar:** correr `harvest_calib.py` con la nueva regla de decisión
   (puede hacerse offline sobre las muestras ya cosechadas en `dataset/calib_raw.jsonl`, sin
   necesidad de descargar fotos nuevas) y comparar:
   - % de veces que la nueva regla da especie/género/familia vs la regla de margen actual.
   - Precisión real de cada nivel con la nueva regla vs la actual (66,7%/72,6%/77,3% o los
     números vigentes de la sección 1).
   - Buscar específicamente si sube la utilidad combinada (más respuestas útiles a género/
     familia sin bajar la precisión de especie).
6. **Desplegar solo si mejora medible.** Si no mejora, documentarlo aquí y quedarse con el
   margen fijo — no es una apuesta segura, es una hipótesis a validar.

### 4.4. Criterio de aceptación
- Con la misma muestra de calibración (2.427 fotos), la regla de mínimo riesgo iguala o
  mejora la precisión por nivel actual (63,9%/69,7%/74,8%) **y/o** reduce el % de "no
  respuesta"/abstención total sin bajar precisión.
- Si empeora cualquiera de los tres niveles, no desplegar.

---

## 5. TAREA — completar geo priors (GPS + fecha)

- **Estado:** 1088/1369 especies con coordenadas (dato de sección M de `YOLOFAUNA.md`,
  puede haber cambiado ligeramente con las 975→ especies actuales — revisar cifra real antes
  de empezar). Faltan ~281 especies, pendientes de Minka-first.
- **Bloqueo conocido:** `scripts/build_geo_priors.py` se corrompe al escribir el fichero de
  salida. **Depurar ese bug concreto** (probablemente escritura no atómica / concurrencia —
  revisar si escribe con `open(..., 'w')` directo en vez de escribir a un temporal y hacer
  `rename` atómico al final, que es la causa típica de corrupción si el proceso se
  interrumpe a mitad).
- **Pasos:**
  1. Reproducir el fallo con un subconjunto pequeño de especies para depurar rápido.
  2. Corregir la escritura (patrón: escribir a `geo_priors.json.tmp`, `os.replace()` al
     final — atómico, no dependas de rollback a mano).
  3. Completar las 281 especies restantes.
  4. **Recosechar la calibración completa** (`harvest_calib.py` + `fit_calib.py`) con los
     geo priors activos al 100% del catálogo — hoy solo están parcialmente activos, así que
     la calibración vigente no refleja el efecto completo de esta señal.
- **Criterio de aceptación:** `geo_priors.json` con las 1369 (o el total vigente) especies
  representadas; recalibración muestra mejora o al menos no empeora frente al 63,9%/69,7%/
  74,8% actual.
- **Esfuerzo:** bajo (es depurar un bug + terminar algo ya empezado, no una técnica nueva).

---

## 6. TAREA — fusión de sinónimos, barrido exhaustivo

- Solo hay **un caso confirmado**: `ambigolimax_valentianus` ≡ `lehmannia_valentiana`.
- **Pasos:**
  1. Correr `scripts/analyze_acc.py` mirando específicamente pares con **confusión mutua
     alta** (ambas especies se confunden la una con la otra en proporción similar, no solo
     una absorbe a la otra) — ese patrón es la firma de un sinónimo taxonómico, distinto de
     una confusión genuina entre especies distintas pero parecidas.
  2. Para cada candidato, verificar en Minka cuál es el nombre canónico vigente (Minka es el
     árbitro, sección C de `YOLOFAUNA.md`) — no fusionar por intuición, confirmar con la
     taxonomía real.
  3. Fusionar: mover los embeddings del slug perdedor al ganador (o remapear el `slug` en
     `target_species.json` y volver a generar `prototype.npy` combinando ambos conjuntos).
- **Criterio de aceptación:** cada fusión confirmada sube el acierto de esa pareja de
  especies sin afectar a otras (medir antes/después con `analyze_acc.py` en ese par
  concreto).
- **Esfuerzo/impacto:** bajo esfuerzo, impacto pequeño (puñado de casos esperables) — hacer
  en paralelo a las tareas 4/5, no como bloque dedicado de una sesión completa.

---

## 7. TAREA — VLM re-ranker sobre el top-3 crítico

- **Cuándo:** después de la tarea 4 (mínimo riesgo), cuando ya haya menos pares realmente
  indistinguibles por embedding puro — así el VLM se usa donde más aporta.
- **Qué:** para fotos en zona de incertidumbre (top-1/top-2 con score muy próximo, mismo
  género), pasar el top-3 candidato a un VLM local (qwen, ya disponible en HanSolo) pidiendo
  que compare rasgos diagnósticos visibles en la foto contra las 2-3 especies candidatas
  (patrón de color, forma de rinóforos/branquias en heterobranquios, etc.).
- **Restricción de recursos conocida:** GPU compartida con qwen (memoria
  `yolofauna-gpu-qwen-contention`, ver memoria persistente) — el re-ranker solo se invoca en
  la minoría de fotos dudosas, no en todo el flujo, así que el coste de contención GPU es
  acotado, pero **coordinar con el patrón ya existente de expulsar/recargar qwen** que usa
  el resto del pipeline.
- **Pasos:**
  1. Definir el prompt: dar la foto + lista de 2-3 especies candidatas con sus rasgos
     diagnósticos conocidos (si existe una ficha descriptiva por especie; si no, generarla a
     partir de la taxonomía/descripciones de Minka).
  2. Solo activar cuando la tarea 4 ya haya decidido "abstenerse en el margen" (es decir,
     el re-ranker es un intento adicional antes de resignarse a género/familia, no un
     reemplazo del kNN).
  3. Medir con `harvest_calib.py` sobre el subconjunto de fotos donde se invoca: ¿el VLM
     acierta más que la abstención a género/familia en esos casos concretos?
- **Criterio de aceptación:** en el subconjunto de fotos dudosas, el VLM re-ranker acierta
  la especie con más frecuencia que quedarse en género/familia, sin introducir falsos
  positivos de alta confianza declarada.

---

## 8. TAREA — reactivar el wave de auto-publicación en Minka

- **Condición explícita: NO reactivar hasta que las tareas 4 (mínimo riesgo) y/o 5 (geo
  priors completos) muestren una mejora medible sobre el 63,9%/69,7%/74,8% actual.** Ya se
  reactivó una vez y se pausó por publicar errores en especies crípticas — no repetir el
  mismo error sin haber cambiado algo material primero.
- **Pasos al reactivar:**
  1. Filtrar por `prediction.p_species` (probabilidad calibrada), **nunca** por
     `confidence` (similitud sin calibrar).
  2. Empezar conservador: `p_species ≥ 0.90` (~96% precisión medida en la calibración
     vigente, ~47,7% de cobertura con los números actuales — revisar cifra vigente en el
     momento de reactivar).
  3. Mantener la salvaguarda `WAVE_YOLOFAUNA_REQUIRE_INAT=1`.
  4. `UPDATE autoid_schedules SET enabled=true WHERE id=1;`
  5. Considerar publicar **a familia** con umbral alto (p≥0.93 o el vigente en ese momento)
     en los casos donde la especie no llegue al umbral — es una ID válida en Minka y casi
     nunca se equivoca a ese nivel.
  6. Vigilar activamente los primeros días — no dar por bueno solo con el número offline,
     confirmar con observaciones reales publicadas.

---

## 9. ORDEN DE EJECUCIÓN RECOMENDADO

```
Sesión 1 (sin GPU extra, sin gasto):
  1. Tarea 4 — inferencia de mínimo riesgo taxonómico (sección 4)
  2. Tarea 3.D — construir dataset de pares crípticos (prerequisito, en paralelo)
  3. Tarea 6 — fusión de sinónimos, barrido con analyze_acc.py (en paralelo, bajo esfuerzo)

Sesión 2 (depuración + datos):
  4. Tarea 5 — depurar build_geo_priors.py, completar 281 especies, recalibrar

Sesión 3 (decisión de inversión de tiempo/dinero):
  5. Sección 3 — árbol de decisión del fine-tuning del encoder:
     3.A (QLoRA) → si falla o degrada → 3.B (clipora) → si sigue sin caber → 3.C (GPU
     cloud, requiere confirmación explícita de gasto) — en paralelo, considerar 3.E
     (DINOv3) como pista de fondo si A/B no dan fruto en un par de sesiones

Sesión 4 (con menos pares crípticos gracias a lo anterior):
  6. Tarea 7 — VLM re-ranker sobre el top-3 crítico

Cuando 4/5 (y opcionalmente el fine-tuning) muestren mejora medible:
  7. Tarea 8 — reactivar el wave, conservador, con vigilancia activa
```

**Regla transversal:** cualquier tarea que cambie BBDD/encoder/scoring termina con
`harvest_calib.py` + `fit_calib.py` + actualizar los umbrales vigentes en este documento y en
`YOLOFAUNA.md` sección L. No dar una tarea por cerrada sin ese paso.

---

## 9bis. ESTIMACIÓN DE TIEMPO Y RESULTADO ESPERADO POR TAREA (2026-08-04)

> **Aviso de honestidad:** las columnas de "resultado esperado" para las tareas 4-7 son
> extrapolaciones razonadas (de la literatura citada o de la lógica del propio pipeline), no
> mediciones propias — nadie ha completado todavía ninguna de ellas en este proyecto. La
> única forma de saber el número real es ejecutar y medir con `harvest_calib.py`. Tratar
> estos rangos como expectativa a priori para decidir si vale la pena invertir tiempo, no
> como una promesa.

| Tarea | Tiempo hasta tener un número medible | Resultado esperado (especie top-1) | Certeza |
|-------|--------------------------------------|-------------------------------------|---------|
| **4. Mínimo riesgo taxonómico** | **Mismo día** — no hace falta GPU nueva ni recosechar datos, se mide offline sobre `calib_raw.jsonl` ya existente | Especie top-1 probablemente ~igual (63,9%); la ganancia real es en **utilidad combinada**: menos fotos abstenidas del todo, más recuperadas a género/familia correcto. El componente equivalente en el paper de referencia aportaba +4,8% dentro de su ensemble | Alta certeza de que no empeora nada (es solo cambiar una regla de decisión); incierto cuánto sube la utilidad combinada hasta medirlo |
| **3.D + 6. Pares crípticos + fusión de sinónimos** | Horas, en paralelo a lo anterior | Sinónimos: **<1 punto global** (solo 1 caso confirmado, quizá 2-5 más tras el barrido — afecta a un puñado de las 975 especies, no al conjunto). Pares crípticos no da resultado por sí solo, es preparar el dataset que alimenta la tarea 3 | Alta (impacto pequeño pero seguro) |
| **5. Geo priors completos** | 1 sesión para depurar `build_geo_priors.py` + descarga en background (a un ritmo similar a otros scripts del proyecto, decenas/hora) → **probablemente 1-3 días de calendario** hasta tener las 281 especies + recalibrar | No hay medición aislada previa del efecto del geo prior ya activo parcialmente. Estimación conservadora: **+1 a 3 puntos** al completarlo al 100% (el 79% ya está activo y su contribución marginal no se ha medido por separado) | Baja-media — es la tarea con menos base empírica propia para estimar |
| **7. VLM re-ranker top-3** | 1-2 sesiones de prompt engineering + medición sobre el subconjunto dudoso | Solo afecta al **subconjunto "dudoso"** (top-1/top-2 muy próximos, quizá 10-20% de las fotos). Si el VLM acierta ahí mejor que quedarse en género/familia, el efecto global sería **+1 a 3 puntos** de especie top-1 (proporcional al tamaño de ese subconjunto) | Especulativo — no hay baseline propio, aunque `duel_inat.py` da precedente de que YOLOFauna+ayuda externa ya gana a iNat en crípticas |
| **3. Fine-tuning del encoder (QLoRA / clipora)** | 1-2 sesiones para dejar el training loop funcionando (resolver bitsandbytes/cuantización) + **entrenamiento en "GPU-días" troceados** — con la 3060 compartida con qwen, en la práctica esto es **1-3 semanas de calendario** troceando sesiones cortas, no una cifra de GPU-horas puras | Es "la palanca que rompe el techo" según el propio análisis del proyecto y la revisión externa. En literatura de fine-tuning de encoders para dominios de grano fino muy específicos, las ganancias típicas sobre un baseline de embeddings congelados+triplet son del orden de **+5 a +15 puntos**, concentradas sobre todo en las especies crípticas (heterobranquios). **Pero ojo**: nunca se ha completado en este proyecto — hay riesgo real de **0 ganancia o incluso degradación** (la cuantización a 4-bit puede introducir ruido; ya se han probado 3 vías que fallaron por VRAM/librería) | La estimación con más incertidumbre de toda la tabla — rango amplio a propósito |
| **3.C Alquilar GPU cloud** | Si se aprueba el gasto: 1 sesión de preparación + **6-24 h de alquiler real** (no semanas) | Mismo rango de resultado que la fila anterior (misma técnica, solo cambia dónde corre) | Misma incertidumbre en el *resultado*, pero mucha menos incertidumbre en el *tiempo* — es la vía para comprimir "semanas" en "días" si el presupuesto lo permite |
| **8. Reactivar el wave** | Inmediato tras validar 4 y/o 5 | No es una mejora de acierto en sí — es empezar a **capturar en producción** la mejora ya lograda por las tareas anteriores | - |

### Dos escenarios de conjunto (orden de magnitud, no cifra prometida)

- **Escenario conservador** (solo tareas 4+5+6, las "baratas", sin tocar el encoder):
  de **63,9% actual** a un rango estimado de **~65-68% especie top-1**, con una mejora más
  visible en cobertura útil a género/familia que en el número de especie en sí. Alcanzable
  en **1-2 semanas de calendario**, sin gasto y sin depender de que la GPU tenga más VRAM.
- **Escenario con fine-tuning del encoder exitoso** (tareas 4+5+6 + sección 3 con QLoRA/
  clipora/cloud funcionando): rango especulativo de **~70-78% especie top-1**, concentrado
  en la mejora de las especies crípticas (heterobranquios). Alcanzable en **3-5 semanas si
  se hace en local** troceado, o **1-2 semanas si se aprueba GPU cloud** para esa parte
  concreta. **Este escenario tiene una probabilidad real de no cumplirse** (ya han fallado
  3 intentos previos por motivos técnicos distintos) — no planificar como si fuera seguro.

---

## 10. REFERENCIAS

| Qué | Dónde |
|-----|-------|
| Research log completo (por qué se descartó cada cosa) | `YOLOFAUNA.md` — secciones "PLAN DE MEJORA", "1bis", "1ter" |
| Estado de tareas / historial de sesión | `/mnt/docs/TAREAS_PENDIENTES.md` §"YOLOFauna" |
| Servicio de identificación | `./docker/fotofauna-yolo/scripts/identify_service.py` |
| Cosecha del set de calibración (honesto, sin fuga) | `scripts/harvest_calib.py` |
| Ajuste de la calibración | `scripts/fit_calib.py` |
| Análisis de confusión / sinónimos / pares crípticos | `scripts/analyze_acc.py` |
| Helper de exec por socket (code-server no tiene CLI docker) | `scripts/dexec.py` |
| Memoria persistente: contención GPU qwen/BioCLIP | memoria `yolofauna-gpu-qwen-contention` |
| Paper: inferencia de mínimo riesgo taxonómico + LLRD | arxiv 2606.25989 |
| Paper: MATANet (crop+contexto multi-escala) | arxiv 2601.03729 |
| `clipora` (LoRA sobre OpenCLIP) | github.com/awilliamson10/clipora |
| Handoff histórico (Tareas A-F, ya completadas/supersedidas por este documento) | `webs/fotofauna/HANDOFF_YOLOFAUNA_FASE2.md` |
