# Especies invasoras de alto impacto y seguimiento prioritario

*Actualizado 15-oct-2026. Sustituye a «Especies invasoras mediáticas / conflictivas» (13-oct). El flag de código `media_conflictive` se mantiene por compatibilidad y es **alias de `alto_impacto`**; los ficheros `especies_mediaticas_conflictivas_20261013.json` pasan a `especies_invasoras_alto_impacto.json`.*

**Alcance.** Ya no es una lista de tres ejemplos. *Pterois miles*, *Callinectes sapidus* y *Rugulopteryx okamurae* eran solo casos de partida (siguen con umbral 0,8467 y alertas curadas, ver «Tratamiento especial»). Esta ficha cubre **149 taxones**: los 114 de la lista UE de nuestro fichero (`target_species.json`), las 29 marinas de la lista provisional de Weitzmann y los ejemplos terrestres, de agua dulce y marinos indicados por Gustavo que no estaban en ninguna de ellas. Sin promote FAISS; AutoID global intacto (p ≥ 0,83, 1 worker).

## Método

- **Galería:** vectores por especie en el índice p15 en producción (`species_ids.npy`, 1.289.637 vectores).
- **Eval, acierto y confusores:** `dataset/calib_raw_t05.jsonl` (decisiones del panel de campo, top-1 de especie); «confusores» = especie más frecuente del top-1 cuando falla. Con n pequeño (10–30) los porcentajes son indicativos.
- **Umbral:** `calibration_hierarchical.json` (por especie; si no hay, por género; si no, 0,83 global).
- **Presencia, observaciones y ratio:** `artifacts/invasoras/presencia_cat_es.csv` (solo para la lista UE; iNaturalist+BioQuest ya descargados, sin llamadas nuevas). «ratio esf. 5 a» = media 2020–24 / 2015–19 de obs por esfuerzo BioQuest, **sin** normalizar por panel de controles: no leer como tendencia ecológica (§Método del paper).
- Nada de esto llama a iNat ni a Minka.

## Tabla resumen (149 taxones)

| Especie | Lista | UE 1143/2014 | CEEA | Presencia CAT | Obs CAT | ratio esf. 5 a | Vectores p15 | Eval n / acierto | Confusores (fallos top-1) | Umbral |
|---|---|---|---|---|---:|---|---:|---|---|---|
| *Acridotheres cristatellus* | UE | Reg. 2025/1422 | sí [género] | ausente | 0 | — | 467 | 10 / 80,0 % | sturnus_vulgaris 1, columba_livia 1 | 0,8223 (género) |
| *Acridotheres tristis* | UE | Reg. 2019/1262 | sí [género] | ausente | 0 | — | 434 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Alopochen aegyptiaca* | UE | Reg. 2017/1263 | sí | establecida | 167 | 0,349 | 911 | 30 / 90,0 % | anas_platyrhynchos 2, fulica_atra 1 | 0,8300 (n=30) |
| *Corvus splendens* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 200 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Oxyura jamaicensis* | UE | Reg. 2016/1141 | sí | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Pycnonotus cafer* | UE | Reg. 2022/1203 | sí | ausente | 0 | — | 180 | 10 / 90,0 % | galerida_cristata 1 | 0,83 (global) |
| *Pycnonotus jocosus* | UE | Reg. 2025/1422 | sí | ausente | 0 | — | 230 | 10 / 100,0 % | — | 0,83 (global) |
| *Threskiornis aethiopicus* | UE | Reg. 2016/1141 | sí | establecida | 16 | 0,135 | 229 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Axis axis* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 182 | 10 / 80,0 % | capra_hircus 1, tragelaphus_strepsiceros 1 | 0,8300 (n=10) |
| *Callosciurus erythraeus* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 230 | 10 / 70,0 % | felis_catus 1, sciurus_niger 1 | 0,8235 (género) |
| *Callosciurus finlaysonii* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 227 | 10 / 90,0 % | callosciurus_erythraeus 1 | 0,8300 (n=10) |
| *Castor canadensis* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 180 | 10 / 90,0 % | quercus_robur 1 | 0,8300 (n=10) |
| *Cervus nippon* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 183 | 10 / 70,0 % | cervus_elaphus 3 | 0,8317 (género) |
| *Herpestes javanicus* | UE | Reg. 2016/1141 | sí | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Muntiacus reevesi* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 180 | 10 / 60,0 % | tragelaphus_scriptus 2, sus_scrofa 1 | 0,83 (global) |
| *Myocastor coypus* | UE | Reg. 2016/1141 | sí | establecida | 498 | 0,902 | 214 | 30 / 96,7 % | bos_taurus 1 | 0,8300 (n=30) |
| *Nasua nasua* | UE | Reg. 2016/1141 | sí [género] | ausente | 0 | — | 180 | 10 / 90,0 % | helogale_parvula 1 | 0,8300 (n=10) |
| *Neogale vison* | UE | Reg. 2025/1422 | — | establecida | 109 | 0,282 | 180 | 10 / 70,0 % | sus_scrofa 1, felis_catus 1 | 0,8300 (n=10) |
| *Nyctereutes procyonoides* | UE | Reg. 2017/1263 | sí | ausente | 0 | — | 200 | 10 / 80,0 % | crocuta_crocuta 1, procyon_lotor 1 | 0,8300 (n=10) |
| *Ondatra zibethicus* | UE | Reg. 2017/1263 | sí | ausente | 0 | — | 228 | 10 / 90,0 % | cyprinus_carpio 1 | 0,8300 (n=10) |
| *Procyon lotor* | UE | Reg. 2016/1141 | sí | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Sciurus carolinensis* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 229 | 10 / 100,0 % | — | 0,83 (global) |
| *Sciurus niger* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 180 | 10 / 90,0 % | sciurus_carolinensis 1 | 0,83 (global) |
| *Tamias sibiricus* | UE | Reg. 2016/1141 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Ameiurus melas* | UE | Reg. 2022/1203 | sí | ocasional_o_puntual | 7 | 0,506 | 227 | 10 / 70,0 % | cyprinus_carpio 2, micropterus_salmoides 1 | 0,83 (global) |
| *Channa argus* | UE | Reg. 2022/1203 | sí [género] | ausente | 0 | — | 180 | 10 / 90,0 % | zosterisessor_ophiocephalus 1 | 0,8300 (n=10) |
| *Fundulus heteroclitus* | UE | Reg. 2022/1203 | sí | ausente | 0 | — | **sin galería en p15** | 12 / 0,0 % | gambusia_holbrooki 9, diplodus_cadenati 1 | 0,83 (global) |
| *Gambusia affinis* | UE | Reg. 2022/1203 | — | ocasional_o_puntual | 1 | — | 450 | 10 / 50,0 % | gambusia_holbrooki 4, crangon_crangon 1 | 0,8300 (n=10) |
| *Gambusia holbrooki* | UE | Reg. 2022/1203 | sí | en_expansion | 128 | 1,619 | 973 | 30 / 83,3 % | gambusia_affinis 2, aphia_minuta 1 | 0,8300 (n=30) |
| *Lepomis gibbosus* | UE | Reg. 2019/1262 | sí | establecida | 36 | 0,731 | 419 | 30 / 96,7 % | gambusia_holbrooki 1 | 0,8300 (n=30) |
| *Misgurnus anguillicaudatus* | UE | Reg. 2025/1422 | sí | ocasional_o_puntual | 2 | 0,084 | 230 | 10 / 80,0 % | dasybranchus_gajolae 1, nerophis_ophidion 1 | 0,8300 (n=10) |
| *Misgurnus bipartitus* | UE | Reg. 2025/1422 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,8173 (género) |
| *Morone americana* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 180 | 10 / 80,0 % | pomadasys_incisus 2 | 0,8300 (n=10) |
| *Perccottus glenii* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 186 | 10 / 80,0 % | zosterisessor_ophiocephalus 1, pomatoschistus_minutus 1 | 0,8300 (n=10) |
| *Plotosus lineatus* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 258 | 30 / 100,0 % | — | 0,8300 (n=30) |
| *Pseudorasbora parva* | UE | Reg. 2016/1141 | sí | ocasional_o_puntual | 8 | 0,126 | 229 | 10 / 60,0 % | gambusia_holbrooki 2, cyprinus_carpio 1 | 0,83 (global) |
| *Cherax destructor* | UE | Reg. 2025/1422 | sí | ausente | 0 | — | 180 | 10 / 60,0 % | procambarus_clarkii 3, homarus_gammarus 1 | 0,83 (global) |
| *Eriocheir sinensis* | UE | Reg. 2016/1141 | sí | ausente | 0 | — | 210 | 10 / 70,0 % | necora_puber 2, carcinus_aestuarii 1 | 0,83 (global) |
| *Faxonius immunis* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 180 | 10 / 70,0 % | procambarus_clarkii 3 | 0,8300 (n=10) |
| *Faxonius rusticus* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 180 | 10 / 50,0 % | pacifastacus_leniusculus 2, upogebia_pusilla 1 | 0,8300 (n=10) |
| *Orconectes limosus* | UE | Reg. 2016/1141 | sí | ocasional_o_puntual | 1 | 99,0 | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Orconectes virilis* | UE | Reg. 2016/1141 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Pacifastacus leniusculus* | UE | Reg. 2016/1141 | sí | establecida | 35 | 0,618 | 227 | 10 / 70,0 % | procambarus_clarkii 3 | 0,8300 (n=10) |
| *Procambarus clarkii* | UE+ej | Reg. 2016/1141 | sí | establecida | 646 | 0,453 | 483 | 30 / 93,3 % | channa_argus 1, procambarus_virginalis 1 | 0,8300 (n=30) |
| *Procambarus virginalis* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 150 | 10 / 50,0 % | procambarus_clarkii 5 | 0,8317 (género) |
| *Lampropeltis getula* | UE | Reg. 2022/1203 | — | ausente | 0 | — | **sin galería en p15** | 12 / 0,0 % | myrichthys_maculosus 4, notospermus_geniculatus 2 | 0,83 (global) |
| *Trachemys scripta elegans* | UE | Reg. 2016/1141 | sí | establecida | 263 | 0,234 | 269 | 30 / 86,7 % | mauremys_leprosa 1, felis_catus 1 | 0,8300 (n=30) |
| *Lithobates catesbeianus* | UE | Reg. 2016/1141 | sí | ausente | 0 | — | 230 | 10 / 90,0 % | pelophylax_perezi 1 | 0,8300 (n=10) |
| *Xenopus laevis* | UE | Reg. 2022/1203 | sí | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Brachyponera chinensis* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 186 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Solenopsis geminata* | UE | Reg. 2022/1203 | — | ausente | 0 | — | **sin galería en p15** | 12 / 0,0 % | solenopsis_invicta 8, solenopsis_richteri 2 | 0,8235 (género) |
| *Solenopsis invicta* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 193 | 10 / 60,0 % | linepithema_humile 2, messor_barbarus 1 | 0,8300 (n=10) |
| *Solenopsis richteri* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 105 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Vespa mandarinia* | UE | Reg. 2025/1422 | sí [género] | ausente | 0 | — | 180 | 10 / 80,0 % | vespa_velutina 2 | 0,8317 (género) |
| *Vespa velutina nigrithorax* | UE | Reg. 2016/1141 | sí [género] | en_expansion | 156 | 1,626 | 200 | 30 / 93,3 % | polistes_gallicus 1, malus_domestica 1 | 0,8300 (n=30) |
| *Wasmannia auropunctata* | UE | Reg. 2022/1203 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Cipangopaludina chinensis* | UE | Reg. 2025/1422 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Limnoperna fortunei* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 180 | 10 / 80,0 % | mytilus_edulis 1, lithophaga_lithophaga 1 | 0,8300 (n=10) |
| *Marisa cornuarietis* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 180 | 10 / 90,0 % | planorbella_duryi 1 | 0,8300 (n=10) |
| *Mulinia lateralis* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 384 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Arthurdendyus triangulatus* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 348 | 10 / 90,0 % | lumbricus_terrestris 1 | 0,8300 (n=10) |
| *Bipalium kewense* | UE | Reg. 2025/1422 | — | ocasional_o_puntual | 3 | 0,042 | 218 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Obama nungara* | UE | Reg. 2025/1422 | — | establecida | 20 | 0,211 | 227 | 10 / 90,0 % | milax_gagates 1 | 0,8300 (n=10) |
| *Platydemus manokwari* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 180 | 10 / 70,0 % | obama_nungara 2, milax_gagates 1 | 0,8300 (n=10) |
| *Asterias amurensis* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 180 | 10 / 70,0 % | marthasterias_glacialis 2, xantho_hydrophilus 1 | 0,8300 (n=10) |
| *Acacia mearnsii* | UE | Reg. 2025/1422 | — | ausente | 0 | — | 460 | 10 / 100,0 % | — | 0,8235 (género) |
| *Acacia saligna* | UE | Reg. 2019/1262 | — | en_expansion | 65 | 1,897 | 451 | 10 / 80,0 % | eucalyptus_camaldulensis 1, prunus_amygdalus 1 | 0,8300 (n=10) |
| *Ailanthus altissima* | UE+ej | Reg. 2019/1262 | sí | en_expansion | 1540 | 1,63 | 945 | 30 / 100,0 % | — | 0,8300 (n=30) |
| *Alternanthera philoxeroides* | UE | Reg. 2017/1263 | sí | ocasional_o_puntual | 2 | 99,0 | 314 | 10 / 90,0 % | spoladea_recurvalis 1 | 0,83 (global) |
| *Andropogon virginicus* | UE | Reg. 2019/1262 | — | ocasional_o_puntual | 1 | 99,0 | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Asclepias syriaca* | UE | Reg. 2017/1263 | — | ausente | 0 | — | 230 | 10 / 100,0 % | — | 0,83 (global) |
| *Baccharis halimifolia* | UE | Reg. 2016/1141 | sí | ausente | 0 | — | 214 | 10 / 100,0 % | — | 0,83 (global) |
| *Broussonetia papyrifera* | UE | Reg. 2025/1422 | — | establecida | 175 | 1,065 | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Cabomba caroliniana* | UE+ej | Reg. 2016/1141 | sí | ausente | 0 | — | 451 | 10 / 90,0 % | elodea_densa 1 | 0,8300 (n=10) |
| *Cardiospermum grandiflorum* | UE | Reg. 2019/1262 | — | ocasional_o_puntual | 3 | 99,0 | 211 | 10 / 80,0 % | araujia_sericifera 1, cucurbita_pepo 1 | 0,83 (global) |
| *Celastrus orbiculatus* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 180 | 10 / 90,0 % | prunus_domestica 1 | 0,8300 (n=10) |
| *Cortaderia jubata* | UE | Reg. 2019/1262 | sí (excepto Canarias) [género] | ocasional_o_puntual | 2 | — | 453 | 10 / 90,0 % | cortaderia_selloana 1 | 0,8300 (n=10) |
| *Crassula helmsii* | UE | Reg. 2025/1422 | sí | ausente | 0 | — | 184 | 10 / 90,0 % | petrosedum_rupestre 1 | 0,83 (global) |
| *Delairea odorata* | UE | Reg. 2025/1422 | — | en_expansion | 107 | 1,551 | 229 | 10 / 100,0 % | — | 0,83 (global) |
| *Ehrharta calycina* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 185 | 10 / 80,0 % | oenothera_lindheimeri 1, melica_ciliata 1 | 0,83 (global) |
| *Eichhornia crassipes* | UE | Reg. 2016/1141 | sí | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Elodea nuttallii* | UE | Reg. 2017/1263 | sí | ocasional_o_puntual | 2 | 99,0 | 218 | 10 / 70,0 % | elodea_densa 3 | 0,8300 (n=10) |
| *Gymnocoronis spilanthoides* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 183 | 10 / 90,0 % | bidens_aurea 1 | 0,83 (global) |
| *Gunnera tinctoria* | UE | Reg. 2017/1263 | — | ausente | 0 | — | 188 | 10 / 100,0 % | — | 0,83 (global) |
| *Hakea sericea* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Heracleum mantegazzianum* | UE | Reg. 2017/1263 | sí | ocasional_o_puntual | 2 | — | **sin galería en p15** | 12 / 0,0 % | heracleum_sosnowskyi 12 | 0,83 (global) |
| *Heracleum persicum* | UE | Reg. 2016/1141 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Heracleum sosnowskyi* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 384 | 10 / 100,0 % | — | 0,83 (global) |
| *Humulus scandens* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 229 | 10 / 100,0 % | — | 0,8317 (género) |
| *Hydrocotyle ranunculoides* | UE | Reg. 2016/1141 | sí | ausente | 0 | — | 224 | 10 / 90,0 % | lemna_minor 1 | 0,8300 (n=10) |
| *Impatiens glandulifera* | UE | Reg. 2017/1263 | — | en_expansion | 10 | 99,0 | 230 | 10 / 100,0 % | — | 0,83 (global) |
| *Koenigia polystachya* | UE | Reg. 2022/1203 | — | ausente | 0 | — | 181 | 10 / 100,0 % | — | 0,83 (global) |
| *Lagarosiphon major* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 227 | 10 / 90,0 % | elodea_densa 1 | 0,8300 (n=10) |
| *Lespedeza cuneata* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Ludwigia grandiflora* | UE | Reg. 2016/1141 | sí [género] | ocasional_o_puntual | 1 | — | 474 | 10 / 70,0 % | ludwigia_peploides 2, arundo_donax 1 | 0,8300 (n=10) |
| *Ludwigia peploides* | UE | Reg. 2016/1141 | sí [género] | en_expansion | 58 | 99,0 | 211 | 10 / 80,0 % | ludwigia_grandiflora 1, alternanthera_philoxeroides 1 | 0,8300 (n=10) |
| *Lygodium japonicum* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Lysichiton americanus* | UE | Reg. 2016/1141 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Microstegium vimineum* | UE | Reg. 2017/1263 | — | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Myriophyllum aquaticum* | UE+ej | Reg. 2016/1141 | sí | ocasional_o_puntual | 1 | 99,0 | 230 | 10 / 90,0 % | ceratophyllum_demersum 1 | 0,8300 (n=10) |
| *Myriophyllum heterophyllum* | UE | Reg. 2017/1263 | — | ausente | 0 | — | 181 | 10 / 80,0 % | elodea_densa 1, gambusia_holbrooki 1 | 0,8300 (n=10) |
| *Nanozostera japonica* | UE | Reg. 2025/1422 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,8240 (género) |
| *Parthenium hysterophorus* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 209 | 10 / 100,0 % | — | 0,83 (global) |
| *Pennisetum setaceum* | UE | Reg. 2017/1263 | sí | en_expansion | 109 | 1,585 | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Persicaria perfoliata* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Pistia stratiotes* | UE | Reg. 2022/1203 | sí | ausente | 0 | — | 229 | 10 / 100,0 % | — | 0,83 (global) |
| *Prosopis juliflora* | UE | Reg. 2019/1262 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Pueraria montana var. lobata* | UE | Reg. 2016/1141 | — | ausente | 0 | — | 180 | 10 / 80,0 % | vitis_vinifera 1, ipomoea_indica 1 | 0,83 (global) |
| *Reynoutria japonica* | UE | Reg. 2025/1422 | sí | establecida | 13 | 0,253 | 492 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Reynoutria sachalinensis* | UE | Reg. 2025/1422 | — | ocasional_o_puntual | 1 | 0,0 | 192 | 10 / 80,0 % | reynoutria_japonica 2 | 0,8300 (n=10) |
| *Reynoutria x bohemica* | UE | Reg. 2025/1422 | — | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,8235 (género) |
| *Rugulopteryx okamurae* | UE+ej | Reg. 2022/1203 | — | en_expansion | 101 | 99,0 | 972 | 20 / 80,0 % | valonia_utricularis 1, dictyota_dichotoma 1 | 0,8300 (n=20) |
| *Salvinia molesta* | UE | Reg. 2019/1262 | sí [género] | ausente | 0 | — | 190 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Triadica sebifera* | UE | Reg. 2019/1262 | — | ausente | 0 | — | 185 | 10 / 90,0 % | melia_azedarach 1 | 0,8300 (n=10) |
| *Zebrasoma flavescens* | W | — | — | fuera lista UE | — | — | 179 | 10 / 90,0 % | acanthurus_dussumieri 1 | 0,83 (global) |
| *Siganus luridus* | W+ej | — | — | fuera lista UE | — | — | 229 | 10 / 80,0 % | schedophilus_ovalis 1, arca_noae 1 | 0,8300 (n=10) |
| *Lagocephalus sceleratus* | W+ej | — | — | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Halophila stipulacea* | W | — | — | fuera lista UE | — | — | 229 | 10 / 80,0 % | cymodocea_nodosa 1, caulerpa_prolifera 1 | 0,8300 (n=10) |
| *Rapana venosa* | W | — | — | fuera lista UE | — | — | 230 | 10 / 70,0 % | stramonita_haemastoma 2, aporrhais_pespelecani 1 | 0,8300 (n=10) |
| *Lophocladia trichoclados* | W | — | — | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Bursatella leachii* | W | — | — | fuera lista UE | — | — | 825 | 30 / 96,7 % | stylocheilus_polyomma 1 | 0,8300 (n=30) |
| *Balistoides conspicillum* | W | — | — | fuera lista UE | — | — | 839 | 30 / 90,0 % | tubastraea_micranthus 1, phyllidiella_pustulosa 1 | 0,8300 (n=30) |
| *Oculina patagonica* | W | — | — | fuera lista UE | — | — | 988 | 30 / 83,3 % | coris_melanura 1, perforatus_perforatus 1 | 0,8300 (n=30) |
| *Caulerpa taxifolia* | W | — | sí | fuera lista UE | — | — | 361 | 30 / 90,0 % | caulerpa_cylindracea 1, ulva_compressa 1 | 0,8300 (n=30) |
| *Caulerpa cylindracea* | W+ej | — | — | fuera lista UE | — | — | 866 | 30 / 56,7 % | caulerpa_racemosa 10, coris_julis 1 | 0,8300 (n=30) |
| *Asparagopsis armata* | W | — | sí | fuera lista UE | — | — | 923 | 30 / 86,7 % | stephanolepis_hispida 1, hildenbrandia_rubra 1 | 0,8300 (n=30) |
| *Asparagopsis taxiformis* | W | — | sí (excepto Canarias) | fuera lista UE | — | — | 972 | 30 / 66,7 % | asparagopsis_armata 3, mullus_surmuletus 1 | 0,8300 (n=30) |
| *Fistularia commersonii* | W+ej | — | — | fuera lista UE | — | — | 421 | 30 / 93,3 % | tylosurus_acus 1, echeneis_naucrates 1 | 0,8300 (n=30) |
| *Siganus rivulatus* | W+ej | — | — | fuera lista UE | — | — | 590 | 30 / 83,3 % | sarpa_salpa 2, coryphaena_hippurus 1 | 0,8300 (n=30) |
| *Pterois miles* | W+ej | — | — | fuera lista UE | — | — | 579 | 30 / 63,3 % | pterois_volitans 10, crambe_crambe 1 | 0,8431 (género) |
| *Percnon gibbesi* | W | — | sí (excepto Canarias) | fuera lista UE | — | — | 992 | 30 / 96,7 % | planes_minutus 1 | 0,8300 (n=30) |
| *Callinectes sapidus* | W+ej | — | — | fuera lista UE | — | — | 983 | 20 / 90,0 % | caretta_caretta 1, mugil_cephalus 1 | 0,8300 (n=20) |
| *Aplysia dactylomela* | W | — | — | fuera lista UE | — | — | 816 | 30 / 93,3 % | aplysia_oculifera 2 | 0,8300 (n=30) |
| *Botrylloides violaceus* | W | — | — | fuera lista UE | — | — | 914 | 30 / 73,3 % | crambe_crambe 1, clathrina_rubra 1 | 0,8300 (n=30) |
| *Magallana gigas* | W | — | — | fuera lista UE | — | — | 999 | 30 / 86,7 % | cardita_calyculata 1, microcosmus_sabatieri 1 | 0,8300 (n=30) |
| *Mnemiopsis leidyi* | W | — | sí | fuera lista UE | — | — | 889 | 30 / 80,0 % | leucothea_multicornis 2, bolinopsis_infundibulum 2 | 0,8300 (n=30) |
| *Paraleucilla magna* | W | — | — | fuera lista UE | — | — | 818 | 28 / 78,6 % | exaiptasia_diaphana 1, sycon_raphanus 1 | 0,8300 (n=28) |
| *Codium fragile* | W | — | sí | fuera lista UE | — | — | 394 | 30 / 96,7 % | placida_dendritica 1 | 0,8300 (n=30) |
| *Sargassum muticum* | W | — | sí | fuera lista UE | — | — | 907 | 30 / 83,3 % | caulerpa_cylindracea 1, elodea_densa 1 | 0,8300 (n=30) |
| *Womersleyella setacea* | W | — | sí | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Stypopodium schimperi* | W | — | sí | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,8264 (género) |
| *Acrothamnion preissii* | W | — | sí | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Kapraunia schneideri* | W | — | — | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Cortaderia selloana* | ej | — | sí (excepto Canarias) [género] | fuera lista UE | — | — | 148 | 30 / 40,0 % | cortaderia_jubata 18 | 0,83 (global) |
| *Opuntia ficus-indica* | ej | — | — | fuera lista UE | — | — | 1170 | 30 / 100,0 % | — | 0,8300 (n=30) |
| *Carpobrotus edulis* | ej | — | sí | fuera lista UE | — | — | 1154 | 30 / 96,7 % | papio_ursinus 1 | 0,8300 (n=30) |
| *Carpobrotus acinaciformis* | ej | — | sí (excepto Canarias) | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,8266 (género) |
| *Myiopsitta monachus* | ej | — | sí | fuera lista UE | — | — | 989 | 30 / 100,0 % | — | 0,8300 (n=30) |
| *Lophocladia lallemandii* | ej | — | sí | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |

**Lectura.** `Lista`: UE = lista de la UE según nuestro fichero (no verificado contra el Reglamento 1143/2014 en EUR-Lex); W = lista Weitzmann provisional; ej = ejemplo añadido. «fuera lista UE» (columna Presencia) = sin fila en `presencia_cat_es.csv`, no «no presente».

## Tratamiento especial (alto impacto mediático)

| Especie | Fotos | Vectores | Eval n | Acierto | Confusiones principales |
|---|---:|---:|---:|---:|---|
| *Pterois miles* | 546 | 500 | 40 | 60 % | *P. volitans* 14, *Arbacia lixula* 2 |
| *Callinectes sapidus* | 978 | 978 | 30 | 90 % | *Caretta*, *Mugil*, *Cronius* (1 c/u) |
| *Rugulopteryx okamurae* | 1004 | 1027 | 30 | 86,7 % | *Valonia*, *Posidonia*, *Octopus*, *Dictyota* (1 c/u) |

(Cifras del 13-oct; los vectores de la tabla anterior son del índice p15.) Umbral jerárquico de las tres = 0,8467. Callinectes no es especie de preocupación de la UE; sí está en Weitzmann.

## Estatus legal (verificado 15-oct-2026)

- **UE 1143/2014 (especies de preocupación para la Unión):** columna «UE 1143/2014» = primer Reglamento de Ejecución en cuyo texto aparece el binomio, comprobado en el texto oficial (Publications Office, CELEX): [2016/1141](https://eur-lex.europa.eu/eli/reg_impl/2016/1141/oj), [2017/1263](https://eur-lex.europa.eu/eli/reg_impl/2017/1263/oj), [2019/1262](https://eur-lex.europa.eu/eli/reg_impl/2019/1262/oj), [2022/1203](https://eur-lex.europa.eu/eli/reg_impl/2022/1203/oj), [2025/1422](https://eur-lex.europa.eu/eli/reg_impl/2025/1422/oj). Resultado: de los 114 taxones de nuestro fichero «UE», **los 114 figuran en alguno de los cinco reglamentos**; no he comprobado retiradas posteriores a 2025/1422 ni reglamentos intermedios (coincidencia por nombre, no por lectura jurídica). *Alopochen aegyptiaca* figura como *A. aegyptiacus* y *Lithobates catesbeianus* como *L. (Rana) catesbeianus* (en el BOE aparece como «Lithobates … catesbeianus», comprobado a mano).
- **CEEA (RD 630/2013, texto consolidado BOE-A-2013-8565, actualizado a dic-2025):** columna «CEEA» = el taxón (o su género con «spp.») figura en el Anexo. Fuente: <https://www.boe.es/buscar/act.php?id=BOE-A-2013-8565>. Incluye el ámbito cuando consta («excepto Canarias»). Modificaciones recientes que recoge el texto: Orden TED/452/2025, TED/339/2023, TED/1126/2020. Ojo: el catálogo lista *Caulerpa racemosa* (no *C. cylindracea*) y solo *Pterois volitans* (no *P. miles*); no se interpreta que lo cubra. *Opuntia ficus-indica* no está (sí *O. dillenii*, *O. maxima*, *O. stricta*).

## Origen, vía de entrada y primera cita (con DOI)

Solo se documenta lo ya verificado en el paper (`ORIGEN_Y_VIAS_EXPANSION.md`, `RESULTADOS_v1.md`), con DOI de Crossref:

- ***Pterois miles***: Indo-Pacífico/Mar Rojo; vía lessepsiana (Suez); primera cita mediterránea Israel 1991 ([Golani & Sonin 1992](https://doi.org/10.1007/bf02906001)); expansión ([Bariche et al. 2013](https://doi.org/10.12681/mms.470); [Kletou et al. 2016](https://doi.org/10.1186/s41200-016-0065-y)). Zenetos et al. 2022 la cita como establecida/invasora ([DOI](https://doi.org/10.12681/mms.29106)).
- ***Caulerpa cylindracea***: Indo-Pacífico; transporte marítimo; vías en [Katsanevakis et al. 2013](https://doi.org/10.1016/j.ocecoaman.2013.02.024); primer registro mediterráneo **pendiente de DOI**.
- ***Oculina patagonica***: expansión hacia el norte con estructuras artificiales y calentamiento ([Serrano et al. 2013](https://doi.org/10.1371/journal.pone.0052739)).
- ***Percnon gibbesi*** y ***Aplysia dactylomela***: criptogénicas según [Zenetos et al. 2022](https://doi.org/10.12681/mms.29106) (no invasoras sin matiz).

**Pendiente de verificar (no se rellena sin fuente):** origen, vía, primera cita en Cataluña/España con DOI e impacto ecológico/económico/sanitario para el resto de taxones. El estatus legal ya está arriba. Primeras observaciones ciudadanas en Cataluña: `presencia_cat_es.csv` (campos `primera_obs_catalunya`) y `RESULTADOS_v1.md` §6.

## Carencias: los 29 sin vectores en p15

Comprobado el 15-oct-2026 contra GBIF Backbone (nombre aceptado, sin tocar iNat/Minka) y contra los 2.988 nombres del índice p15. **Ninguno se resuelve por sinonimia**: en ningún caso el nombre aceptado está en la galería, así que los 29 faltan de verdad. Especies parecidas que sí están en galería (*Zostera/Nanozostera noltii*, *Reynoutria japonica*, *Carpobrotus edulis*, *Misgurnus anguillicaudatus*, *Faxonius immunis/rusticus*, *Stypopodium zonale*, *Cenchrus longisetus*) son **otras especies**, no equivalentes.

| Taxón de la lista | Nombre aceptado (GBIF) | Veredicto |
|---|---|---|
| *Herpestes javanicus* | *Herpestes javanicus* | nombre aceptado; **falta de verdad** |
| *Tamias sibiricus* | *Tamias sibiricus* | nombre aceptado; **falta de verdad** |
| *Fundulus heteroclitus* | *Fundulus heteroclitus* | nombre aceptado; **falta de verdad** |
| *Misgurnus bipartitus* | *Misgurnus mohoity* | sinónimo; el nombre aceptado **tampoco** está en galería |
| *Orconectes limosus* | *Faxonius limosus* | sinónimo; el nombre aceptado **tampoco** está en galería |
| *Orconectes virilis* | *Faxonius virilis* | sinónimo; el nombre aceptado **tampoco** está en galería |
| *Lampropeltis getula* | *Lampropeltis getula* | nombre aceptado; **falta de verdad** |
| *Xenopus laevis* | *Xenopus laevis* | nombre aceptado; **falta de verdad** |
| *Solenopsis geminata* | *Solenopsis geminata* | nombre aceptado; **falta de verdad** |
| *Wasmannia auropunctata* | *Wasmannia auropunctata* | nombre aceptado; **falta de verdad** |
| *Cipangopaludina chinensis* | *Cipangopaludina chinensis* | nombre aceptado; **falta de verdad** |
| *Andropogon virginicus* | *Andropogon virginicus* | nombre aceptado; **falta de verdad** |
| *Broussonetia papyrifera* | *Broussonetia papyrifera* | nombre aceptado; **falta de verdad** |
| *Eichhornia crassipes* | *Pontederia crassipes* | sinónimo; el nombre aceptado **tampoco** está en galería |
| *Heracleum mantegazzianum* | *Heracleum mantegazzianum* | nombre aceptado; **falta de verdad** |
| *Heracleum persicum* | *Heracleum persicum* | nombre aceptado; **falta de verdad** |
| *Lysichiton americanus* | *Lysichiton americanus* | nombre aceptado; **falta de verdad** |
| *Nanozostera japonica* | *Zostera japonica* | sinónimo; el nombre aceptado **tampoco** está en galería |
| *Pennisetum setaceum* | *Cenchrus setaceus* | sinónimo; el nombre aceptado **tampoco** está en galería |
| *Prosopis juliflora* | *Prosopis juliflora* | nombre aceptado; **falta de verdad** |
| *Reynoutria x bohemica* | *Reynoutria bohemica* | nombre aceptado; **falta de verdad** |
| *Lagocephalus sceleratus* | *Lagocephalus sceleratus* | nombre aceptado; **falta de verdad** |
| *Lophocladia trichoclados* | *Lophocladia trichoclados* | nombre aceptado; **falta de verdad** |
| *Womersleyella setacea* | *Womersleyella setacea* | nombre aceptado; **falta de verdad** |
| *Stypopodium schimperi* | *Stypopodium schimperi* | nombre aceptado; **falta de verdad** |
| *Acrothamnion preissii* | *Acrothamnion preissii* | nombre aceptado; **falta de verdad** |
| *Kapraunia schneideri* | — | no verificado (GBIF solo a rango superior) |
| *Carpobrotus acinaciformis* | *Carpobrotus acinaciformis* | nombre aceptado; **falta de verdad** |
| *Lophocladia lallemandii* | *Lophocladia lallemandii* | nombre aceptado; **falta de verdad** |

Notas: *Lophocladia lallemandii* (CEEA, Orden de inclusión) y *L. trichoclados* (Weitzmann) aparecen como dos nombres aceptados distintos en GBIF; conviene decidir con Minka cuál usar antes de cosechar (manda Minka). Con eval n = 10, un 100 % no es fiable. Confusión destacada: *Cortaderia selloana* (40 %, 18 de sus fallos son *C. jubata*) y *Pterois miles* frente a *P. volitans* (10 de 30).

## Acciones propuestas

1. Cosecha local (sin iNat) para los 29 que faltan de verdad, empezando por los marinos de Weitzmann (*Lagocephalus sceleratus*, *Lophocladia*, *Womersleyella*) y los terrestres con presencia en Cataluña.
2. Ampliar eval a n ≥ 30 en los de categoría «establecida» o «en_expansion».
3. Confusores para el índice p17: pares *Pterois miles*/*P. volitans* y géneros con pares críticos.
4. Completar origen/vía/primera cita/impacto con DOI y el cruce CEEA.
