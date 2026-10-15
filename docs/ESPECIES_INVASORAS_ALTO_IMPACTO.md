# Especies invasoras de alto impacto y seguimiento prioritario

*Actualizado 15-oct-2026. Sustituye a «Especies invasoras mediáticas / conflictivas» (13-oct). El flag de código `media_conflictive` se mantiene por compatibilidad y es **alias de `alto_impacto`**; los ficheros `especies_mediaticas_conflictivas_20261013.json` pasan a `especies_invasoras_alto_impacto.json`.*

**Alcance.** Ya no es una lista de tres ejemplos. *Pterois miles*, *Callinectes sapidus* y *Rugulopteryx okamurae* eran solo casos de partida (siguen con umbral 0,8467 y alertas curadas, ver «Tratamiento especial»). Esta ficha cubre **150 taxones**: los 114 de la lista UE de nuestro fichero (`target_species.json`), las 29 marinas de la lista provisional de Weitzmann y los ejemplos terrestres, de agua dulce y marinos indicados por Gustavo que no estaban en ninguna de ellas. Sin promote FAISS; AutoID global intacto (p ≥ 0,83, 1 worker).

## Método

- **Galería:** vectores por especie en el índice p15 en producción (`species_ids.npy`, 1.289.637 vectores).
- **Eval, acierto y confusores:** `dataset/calib_raw_t05.jsonl` (decisiones del panel de campo, top-1 de especie); «confusores» = especie más frecuente del top-1 cuando falla. Con n pequeño (10–30) los porcentajes son indicativos.
- **Umbral:** `calibration_hierarchical.json` (por especie; si no hay, por género; si no, 0,83 global).
- **Presencia, observaciones y ratio:** `artifacts/invasoras/presencia_cat_es.csv` (solo para la lista UE; iNaturalist+BioQuest ya descargados, sin llamadas nuevas). «ratio esf. 5 a» = media 2020–24 / 2015–19 de obs por esfuerzo BioQuest, **sin** normalizar por panel de controles: no leer como tendencia ecológica (§Método del paper).
- Nada de esto llama a iNat ni a Minka.

## Tabla resumen (150 taxones)

| Especie | Lista | Presencia CAT | Obs CAT | ratio esf. 5 a | Vectores p15 | Eval n / acierto | Confusores (fallos top-1) | Umbral |
|---|---|---|---:|---|---:|---|---|---|
| *Acridotheres cristatellus* | UE | ausente | 0 | — | 467 | 10 / 80,0 % | sturnus_vulgaris 1, columba_livia 1 | 0,8223 (género) |
| *Acridotheres tristis* | UE | ausente | 0 | — | 434 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Alopochen aegyptiaca* | UE | establecida | 167 | 0,349 | 911 | 30 / 90,0 % | anas_platyrhynchos 2, fulica_atra 1 | 0,8300 (n=30) |
| *Corvus splendens* | UE | ausente | 0 | — | 200 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Oxyura jamaicensis* | UE | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Pycnonotus cafer* | UE | ausente | 0 | — | 180 | 10 / 90,0 % | galerida_cristata 1 | 0,83 (global) |
| *Pycnonotus jocosus* | UE | ausente | 0 | — | 230 | 10 / 100,0 % | — | 0,83 (global) |
| *Threskiornis aethiopicus* | UE | establecida | 16 | 0,135 | 229 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Axis axis* | UE | ausente | 0 | — | 182 | 10 / 80,0 % | capra_hircus 1, tragelaphus_strepsiceros 1 | 0,8300 (n=10) |
| *Callosciurus erythraeus* | UE | ausente | 0 | — | 230 | 10 / 70,0 % | felis_catus 1, sciurus_niger 1 | 0,8235 (género) |
| *Callosciurus finlaysonii* | UE | ausente | 0 | — | 227 | 10 / 90,0 % | callosciurus_erythraeus 1 | 0,8300 (n=10) |
| *Castor canadensis* | UE | ausente | 0 | — | 180 | 10 / 90,0 % | quercus_robur 1 | 0,8300 (n=10) |
| *Cervus nippon* | UE | ausente | 0 | — | 183 | 10 / 70,0 % | cervus_elaphus 3 | 0,8317 (género) |
| *Herpestes javanicus* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Muntiacus reevesi* | UE | ausente | 0 | — | 180 | 10 / 60,0 % | tragelaphus_scriptus 2, sus_scrofa 1 | 0,83 (global) |
| *Myocastor coypus* | UE | establecida | 498 | 0,902 | 214 | 30 / 96,7 % | bos_taurus 1 | 0,8300 (n=30) |
| *Nasua nasua* | UE | ausente | 0 | — | 180 | 10 / 90,0 % | helogale_parvula 1 | 0,8300 (n=10) |
| *Neogale vison* | UE | establecida | 109 | 0,282 | 180 | 10 / 70,0 % | sus_scrofa 1, felis_catus 1 | 0,8300 (n=10) |
| *Nyctereutes procyonoides* | UE | ausente | 0 | — | 200 | 10 / 80,0 % | crocuta_crocuta 1, procyon_lotor 1 | 0,8300 (n=10) |
| *Ondatra zibethicus* | UE | ausente | 0 | — | 228 | 10 / 90,0 % | cyprinus_carpio 1 | 0,8300 (n=10) |
| *Procyon lotor* | UE | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Sciurus carolinensis* | UE | ausente | 0 | — | 229 | 10 / 100,0 % | — | 0,83 (global) |
| *Sciurus niger* | UE | ausente | 0 | — | 180 | 10 / 90,0 % | sciurus_carolinensis 1 | 0,83 (global) |
| *Tamias sibiricus* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Ameiurus melas* | UE | ocasional_o_puntual | 7 | 0,506 | 227 | 10 / 70,0 % | cyprinus_carpio 2, micropterus_salmoides 1 | 0,83 (global) |
| *Channa argus* | UE | ausente | 0 | — | 180 | 10 / 90,0 % | zosterisessor_ophiocephalus 1 | 0,8300 (n=10) |
| *Fundulus heteroclitus* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Gambusia affinis* | UE | ocasional_o_puntual | 1 | — | 450 | 10 / 50,0 % | gambusia_holbrooki 4, crangon_crangon 1 | 0,8300 (n=10) |
| *Gambusia holbrooki* | UE | en_expansion | 128 | 1,619 | 973 | 30 / 83,3 % | gambusia_affinis 2, aphia_minuta 1 | 0,8300 (n=30) |
| *Lepomis gibbosus* | UE | establecida | 36 | 0,731 | 419 | 30 / 96,7 % | gambusia_holbrooki 1 | 0,8300 (n=30) |
| *Misgurnus anguillicaudatus* | UE | ocasional_o_puntual | 2 | 0,084 | 230 | 10 / 80,0 % | dasybranchus_gajolae 1, nerophis_ophidion 1 | 0,8300 (n=10) |
| *Misgurnus bipartitus* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,8173 (género) |
| *Morone americana* | UE | ausente | 0 | — | 180 | 10 / 80,0 % | pomadasys_incisus 2 | 0,8300 (n=10) |
| *Perccottus glenii* | UE | ausente | 0 | — | 186 | 10 / 80,0 % | zosterisessor_ophiocephalus 1, pomatoschistus_minutus 1 | 0,8300 (n=10) |
| *Plotosus lineatus* | UE | ausente | 0 | — | 258 | 30 / 100,0 % | — | 0,8300 (n=30) |
| *Pseudorasbora parva* | UE | ocasional_o_puntual | 8 | 0,126 | 229 | 10 / 60,0 % | gambusia_holbrooki 2, cyprinus_carpio 1 | 0,83 (global) |
| *Cherax destructor* | UE | ausente | 0 | — | 180 | 10 / 60,0 % | procambarus_clarkii 3, homarus_gammarus 1 | 0,83 (global) |
| *Eriocheir sinensis* | UE | ausente | 0 | — | 210 | 10 / 70,0 % | necora_puber 2, carcinus_aestuarii 1 | 0,83 (global) |
| *Faxonius immunis* | UE | ausente | 0 | — | 180 | 10 / 70,0 % | procambarus_clarkii 3 | 0,8300 (n=10) |
| *Faxonius rusticus* | UE | ausente | 0 | — | 180 | 10 / 50,0 % | pacifastacus_leniusculus 2, upogebia_pusilla 1 | 0,8300 (n=10) |
| *Orconectes limosus* | UE | ocasional_o_puntual | 1 | 99,0 | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Orconectes virilis* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Pacifastacus leniusculus* | UE | establecida | 35 | 0,618 | 227 | 10 / 70,0 % | procambarus_clarkii 3 | 0,8300 (n=10) |
| *Procambarus clarkii* | UE+ej | establecida | 646 | 0,453 | 483 | 30 / 93,3 % | channa_argus 1, procambarus_virginalis 1 | 0,8300 (n=30) |
| *Procambarus virginalis* | UE | ausente | 0 | — | 150 | 10 / 50,0 % | procambarus_clarkii 5 | 0,8317 (género) |
| *Lampropeltis getula* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Trachemys scripta elegans* | UE | establecida | 263 | 0,234 | 269 | 30 / 86,7 % | mauremys_leprosa 1, felis_catus 1 | 0,8300 (n=30) |
| *Lithobates catesbeianus* | UE | ausente | 0 | — | 230 | 10 / 90,0 % | pelophylax_perezi 1 | 0,8300 (n=10) |
| *Xenopus laevis* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Brachyponera chinensis* | UE | ausente | 0 | — | 186 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Solenopsis geminata* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,8235 (género) |
| *Solenopsis invicta* | UE | ausente | 0 | — | 193 | 10 / 60,0 % | linepithema_humile 2, messor_barbarus 1 | 0,8300 (n=10) |
| *Solenopsis richteri* | UE | ausente | 0 | — | 105 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Vespa mandarinia* | UE | ausente | 0 | — | 180 | 10 / 80,0 % | vespa_velutina 2 | 0,8317 (género) |
| *Vespa velutina nigrithorax* | UE | en_expansion | 156 | 1,626 | 200 | 30 / 93,3 % | polistes_gallicus 1, malus_domestica 1 | 0,8300 (n=30) |
| *Wasmannia auropunctata* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Cipangopaludina chinensis* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Limnoperna fortunei* | UE | ausente | 0 | — | 180 | 10 / 80,0 % | mytilus_edulis 1, lithophaga_lithophaga 1 | 0,8300 (n=10) |
| *Marisa cornuarietis* | UE | ausente | 0 | — | 180 | 10 / 90,0 % | planorbella_duryi 1 | 0,8300 (n=10) |
| *Mulinia lateralis* | UE | ausente | 0 | — | 384 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Arthurdendyus triangulatus* | UE | ausente | 0 | — | 348 | 10 / 90,0 % | lumbricus_terrestris 1 | 0,8300 (n=10) |
| *Bipalium kewense* | UE | ocasional_o_puntual | 3 | 0,042 | 218 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Obama nungara* | UE | establecida | 20 | 0,211 | 227 | 10 / 90,0 % | milax_gagates 1 | 0,8300 (n=10) |
| *Platydemus manokwari* | UE | ausente | 0 | — | 180 | 10 / 70,0 % | obama_nungara 2, milax_gagates 1 | 0,8300 (n=10) |
| *Asterias amurensis* | UE | ausente | 0 | — | 180 | 10 / 70,0 % | marthasterias_glacialis 2, xantho_hydrophilus 1 | 0,8300 (n=10) |
| *Acacia mearnsii* | UE | ausente | 0 | — | 460 | 10 / 100,0 % | — | 0,8235 (género) |
| *Acacia saligna* | UE | en_expansion | 65 | 1,897 | 451 | 10 / 80,0 % | eucalyptus_camaldulensis 1, prunus_amygdalus 1 | 0,8300 (n=10) |
| *Ailanthus altissima* | UE+ej | en_expansion | 1540 | 1,63 | 945 | 30 / 100,0 % | — | 0,8300 (n=30) |
| *Alternanthera philoxeroides* | UE | ocasional_o_puntual | 2 | 99,0 | 314 | 10 / 90,0 % | spoladea_recurvalis 1 | 0,83 (global) |
| *Andropogon virginicus* | UE | ocasional_o_puntual | 1 | 99,0 | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Asclepias syriaca* | UE | ausente | 0 | — | 230 | 10 / 100,0 % | — | 0,83 (global) |
| *Baccharis halimifolia* | UE | ausente | 0 | — | 214 | 10 / 100,0 % | — | 0,83 (global) |
| *Broussonetia papyrifera* | UE | establecida | 175 | 1,065 | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Cabomba caroliniana* | UE+ej | ausente | 0 | — | 451 | 10 / 90,0 % | elodea_densa 1 | 0,8300 (n=10) |
| *Cardiospermum grandiflorum* | UE | ocasional_o_puntual | 3 | 99,0 | 211 | 10 / 80,0 % | araujia_sericifera 1, cucurbita_pepo 1 | 0,83 (global) |
| *Celastrus orbiculatus* | UE | ausente | 0 | — | 180 | 10 / 90,0 % | prunus_domestica 1 | 0,8300 (n=10) |
| *Cortaderia jubata* | UE | ocasional_o_puntual | 2 | — | 453 | 10 / 90,0 % | cortaderia_selloana 1 | 0,8300 (n=10) |
| *Crassula helmsii* | UE | ausente | 0 | — | 184 | 10 / 90,0 % | petrosedum_rupestre 1 | 0,83 (global) |
| *Delairea odorata* | UE | en_expansion | 107 | 1,551 | 229 | 10 / 100,0 % | — | 0,83 (global) |
| *Ehrharta calycina* | UE | ausente | 0 | — | 185 | 10 / 80,0 % | oenothera_lindheimeri 1, melica_ciliata 1 | 0,83 (global) |
| *Eichhornia crassipes* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Elodea nuttallii* | UE | ocasional_o_puntual | 2 | 99,0 | 218 | 10 / 70,0 % | elodea_densa 3 | 0,8300 (n=10) |
| *Gymnocoronis spilanthoides* | UE | ausente | 0 | — | 183 | 10 / 90,0 % | bidens_aurea 1 | 0,83 (global) |
| *Gunnera tinctoria* | UE | ausente | 0 | — | 188 | 10 / 100,0 % | — | 0,83 (global) |
| *Hakea sericea* | UE | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Heracleum mantegazzianum* | UE | ocasional_o_puntual | 2 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Heracleum persicum* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Heracleum sosnowskyi* | UE | ausente | 0 | — | 384 | 10 / 100,0 % | — | 0,83 (global) |
| *Humulus scandens* | UE | ausente | 0 | — | 229 | 10 / 100,0 % | — | 0,8317 (género) |
| *Hydrocotyle ranunculoides* | UE | ausente | 0 | — | 224 | 10 / 90,0 % | lemna_minor 1 | 0,8300 (n=10) |
| *Impatiens glandulifera* | UE | en_expansion | 10 | 99,0 | 230 | 10 / 100,0 % | — | 0,83 (global) |
| *Koenigia polystachya* | UE | ausente | 0 | — | 181 | 10 / 100,0 % | — | 0,83 (global) |
| *Lagarosiphon major* | UE | ausente | 0 | — | 227 | 10 / 90,0 % | elodea_densa 1 | 0,8300 (n=10) |
| *Lespedeza cuneata* | UE | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Ludwigia grandiflora* | UE | ocasional_o_puntual | 1 | — | 474 | 10 / 70,0 % | ludwigia_peploides 2, arundo_donax 1 | 0,8300 (n=10) |
| *Ludwigia peploides* | UE | en_expansion | 58 | 99,0 | 211 | 10 / 80,0 % | ludwigia_grandiflora 1, alternanthera_philoxeroides 1 | 0,8300 (n=10) |
| *Lygodium japonicum* | UE | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Lysichiton americanus* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Microstegium vimineum* | UE | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Myriophyllum aquaticum* | UE+ej | ocasional_o_puntual | 1 | 99,0 | 230 | 10 / 90,0 % | ceratophyllum_demersum 1 | 0,8300 (n=10) |
| *Myriophyllum heterophyllum* | UE | ausente | 0 | — | 181 | 10 / 80,0 % | elodea_densa 1, gambusia_holbrooki 1 | 0,8300 (n=10) |
| *Nanozostera japonica* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,8240 (género) |
| *Parthenium hysterophorus* | UE | ausente | 0 | — | 209 | 10 / 100,0 % | — | 0,83 (global) |
| *Pennisetum setaceum* | UE | en_expansion | 109 | 1,585 | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Persicaria perfoliata* | UE | ausente | 0 | — | 180 | 10 / 100,0 % | — | 0,83 (global) |
| *Pistia stratiotes* | UE | ausente | 0 | — | 229 | 10 / 100,0 % | — | 0,83 (global) |
| *Prosopis juliflora* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Pueraria montana var. lobata* | UE | ausente | 0 | — | 180 | 10 / 80,0 % | vitis_vinifera 1, ipomoea_indica 1 | 0,83 (global) |
| *Reynoutria japonica* | UE | establecida | 13 | 0,253 | 492 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Reynoutria sachalinensis* | UE | ocasional_o_puntual | 1 | 0,0 | 192 | 10 / 80,0 % | reynoutria_japonica 2 | 0,8300 (n=10) |
| *Reynoutria x bohemica* | UE | ausente | 0 | — | **sin galería en p15** | 0 / — | — | 0,8235 (género) |
| *Rugulopteryx okamurae* | UE+ej | en_expansion | 101 | 99,0 | 972 | 20 / 80,0 % | valonia_utricularis 1, dictyota_dichotoma 1 | 0,8300 (n=20) |
| *Salvinia molesta* | UE | ausente | 0 | — | 190 | 10 / 100,0 % | — | 0,8300 (n=10) |
| *Triadica sebifera* | UE | ausente | 0 | — | 185 | 10 / 90,0 % | melia_azedarach 1 | 0,8300 (n=10) |
| *Zebrasoma flavescens* | W | fuera lista UE | — | — | 179 | 10 / 90,0 % | acanthurus_dussumieri 1 | 0,83 (global) |
| *Siganus luridus* | W+ej | fuera lista UE | — | — | 229 | 10 / 80,0 % | schedophilus_ovalis 1, arca_noae 1 | 0,8300 (n=10) |
| *Lagocephalus sceleratus* | W+ej | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Halophila stipulacea* | W | fuera lista UE | — | — | 229 | 10 / 80,0 % | cymodocea_nodosa 1, caulerpa_prolifera 1 | 0,8300 (n=10) |
| *Rapana venosa* | W | fuera lista UE | — | — | 230 | 10 / 70,0 % | stramonita_haemastoma 2, aporrhais_pespelecani 1 | 0,8300 (n=10) |
| *Lophocladia trichoclados* | W | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Bursatella leachii* | W | fuera lista UE | — | — | 825 | 30 / 96,7 % | stylocheilus_polyomma 1 | 0,8300 (n=30) |
| *Balistoides conspicillum* | W | fuera lista UE | — | — | 839 | 30 / 90,0 % | tubastraea_micranthus 1, phyllidiella_pustulosa 1 | 0,8300 (n=30) |
| *Oculina patagonica* | W | fuera lista UE | — | — | 988 | 30 / 83,3 % | coris_melanura 1, perforatus_perforatus 1 | 0,8300 (n=30) |
| *Caulerpa taxifolia* | W | fuera lista UE | — | — | 361 | 30 / 90,0 % | caulerpa_cylindracea 1, ulva_compressa 1 | 0,8300 (n=30) |
| *Caulerpa cylindracea* | W+ej | fuera lista UE | — | — | 866 | 30 / 56,7 % | caulerpa_racemosa 10, coris_julis 1 | 0,8300 (n=30) |
| *Asparagopsis armata* | W | fuera lista UE | — | — | 923 | 30 / 86,7 % | stephanolepis_hispida 1, hildenbrandia_rubra 1 | 0,8300 (n=30) |
| *Asparagopsis taxiformis* | W | fuera lista UE | — | — | 972 | 30 / 66,7 % | asparagopsis_armata 3, mullus_surmuletus 1 | 0,8300 (n=30) |
| *Fistularia commersonii* | W+ej | fuera lista UE | — | — | 421 | 30 / 93,3 % | tylosurus_acus 1, echeneis_naucrates 1 | 0,8300 (n=30) |
| *Siganus rivulatus* | W+ej | fuera lista UE | — | — | 590 | 30 / 83,3 % | sarpa_salpa 2, coryphaena_hippurus 1 | 0,8300 (n=30) |
| *Pterois miles* | W+ej | fuera lista UE | — | — | 579 | 30 / 63,3 % | pterois_volitans 10, crambe_crambe 1 | 0,8431 (género) |
| *Percnon gibbesi* | W | fuera lista UE | — | — | 992 | 30 / 96,7 % | planes_minutus 1 | 0,8300 (n=30) |
| *Callinectes sapidus* | W+ej | fuera lista UE | — | — | 983 | 20 / 90,0 % | caretta_caretta 1, mugil_cephalus 1 | 0,8300 (n=20) |
| *Aplysia dactylomela* | W | fuera lista UE | — | — | 816 | 30 / 93,3 % | aplysia_oculifera 2 | 0,8300 (n=30) |
| *Botrylloides violaceus* | W | fuera lista UE | — | — | 914 | 30 / 73,3 % | crambe_crambe 1, clathrina_rubra 1 | 0,8300 (n=30) |
| *Magallana gigas* | W | fuera lista UE | — | — | 999 | 30 / 86,7 % | cardita_calyculata 1, microcosmus_sabatieri 1 | 0,8300 (n=30) |
| *Mnemiopsis leidyi* | W | fuera lista UE | — | — | 889 | 30 / 80,0 % | leucothea_multicornis 2, bolinopsis_infundibulum 2 | 0,8300 (n=30) |
| *Paraleucilla magna* | W | fuera lista UE | — | — | 818 | 28 / 78,6 % | exaiptasia_diaphana 1, sycon_raphanus 1 | 0,8300 (n=28) |
| *Codium fragile* | W | fuera lista UE | — | — | 394 | 30 / 96,7 % | placida_dendritica 1 | 0,8300 (n=30) |
| *Sargassum muticum* | W | fuera lista UE | — | — | 907 | 30 / 83,3 % | caulerpa_cylindracea 1, elodea_densa 1 | 0,8300 (n=30) |
| *Womersleyella setacea* | W | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Stypopodium schimperi* | W | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,8264 (género) |
| *Acrothamnion preissii* | W | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Kapraunia schneideri* | W | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |
| *Cortaderia selloana* | ej | fuera lista UE | — | — | 148 | 30 / 40,0 % | cortaderia_jubata 18 | 0,83 (global) |
| *Opuntia ficus-indica* | ej | fuera lista UE | — | — | 1170 | 30 / 100,0 % | — | 0,8300 (n=30) |
| *Carpobrotus edulis* | ej | fuera lista UE | — | — | 1154 | 30 / 96,7 % | papio_ursinus 1 | 0,8300 (n=30) |
| *Carpobrotus acinaciformis* | ej | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,8266 (género) |
| *Myiopsitta monachus* | ej | fuera lista UE | — | — | 989 | 30 / 100,0 % | — | 0,8300 (n=30) |
| *Lophocladia lallemandii* | ej | fuera lista UE | — | — | **sin galería en p15** | 0 / — | — | 0,83 (global) |

**Lectura.** `Lista`: UE = lista de la UE según nuestro fichero (no verificado contra el Reglamento 1143/2014 en EUR-Lex); W = lista Weitzmann provisional; ej = ejemplo añadido. «fuera lista UE» = sin fila en `presencia_cat_es.csv`, no «no presente».

## Tratamiento especial (alto impacto mediático)

| Especie | Fotos | Vectores | Eval n | Acierto | Confusiones principales |
|---|---:|---:|---:|---:|---|
| *Pterois miles* | 546 | 500 | 40 | 60 % | *P. volitans* 14, *Arbacia lixula* 2 |
| *Callinectes sapidus* | 978 | 978 | 30 | 90 % | *Caretta*, *Mugil*, *Cronius* (1 c/u) |
| *Rugulopteryx okamurae* | 1004 | 1027 | 30 | 86,7 % | *Valonia*, *Posidonia*, *Octopus*, *Dictyota* (1 c/u) |

(Cifras del 13-oct; los vectores de la tabla anterior son del índice p15.) Umbral jerárquico de las tres = 0,8467. Callinectes no es especie de preocupación de la UE; sí está en Weitzmann.

## Origen, vía de entrada y primera cita (con DOI)

Solo se documenta lo ya verificado en el paper (`ORIGEN_Y_VIAS_EXPANSION.md`, `RESULTADOS_v1.md`), con DOI de Crossref:

- ***Pterois miles***: Indo-Pacífico/Mar Rojo; vía lessepsiana (Suez); primera cita mediterránea Israel 1991 ([Golani & Sonin 1992](https://doi.org/10.1007/bf02906001)); expansión ([Bariche et al. 2013](https://doi.org/10.12681/mms.470); [Kletou et al. 2016](https://doi.org/10.1186/s41200-016-0065-y)). Zenetos et al. 2022 la cita como establecida/invasora ([DOI](https://doi.org/10.12681/mms.29106)).
- ***Caulerpa cylindracea***: Indo-Pacífico; transporte marítimo; vías en [Katsanevakis et al. 2013](https://doi.org/10.1016/j.ocecoaman.2013.02.024); primer registro mediterráneo **pendiente de DOI**.
- ***Oculina patagonica***: expansión hacia el norte con estructuras artificiales y calentamiento ([Serrano et al. 2013](https://doi.org/10.1371/journal.pone.0052739)).
- ***Percnon gibbesi*** y ***Aplysia dactylomela***: criptogénicas según [Zenetos et al. 2022](https://doi.org/10.12681/mms.29106) (no invasoras sin matiz).

**Pendiente de verificar (no se rellena sin fuente):** origen, vía, primera cita en Cataluña/España con DOI, impacto ecológico/económico/sanitario y estatus legal para el resto de taxones; en concreto el **catálogo CEEA (RD 630/2013)**, que no está en nuestros ficheros y hay que cruzar contra el BOE. Primeras observaciones ciudadanas en Cataluña: `presencia_cat_es.csv` (campos `primera_obs_catalunya`) y `RESULTADOS_v1.md` §6.

## Carencias

- **29 taxones sin vectores en p15** (no se pueden identificar; el modelo contestaría con otra especie):

- *Herpestes javanicus*: sin vectores en el índice p15
- *Tamias sibiricus*: sin vectores en el índice p15
- *Fundulus heteroclitus*: sin vectores en el índice p15
- *Misgurnus bipartitus*: sin vectores en el índice p15
- *Orconectes limosus*: sin vectores en el índice p15
- *Orconectes virilis*: sin vectores en el índice p15
- *Lampropeltis getula*: sin vectores en el índice p15
- *Xenopus laevis*: sin vectores en el índice p15
- *Solenopsis geminata*: sin vectores en el índice p15
- *Wasmannia auropunctata*: sin vectores en el índice p15
- *Cipangopaludina chinensis*: sin vectores en el índice p15
- *Andropogon virginicus*: sin vectores en el índice p15
- *Broussonetia papyrifera*: sin vectores en el índice p15
- *Eichhornia crassipes*: sin vectores en el índice p15
- *Heracleum mantegazzianum*: sin vectores en el índice p15
- *Heracleum persicum*: sin vectores en el índice p15
- *Lysichiton americanus*: sin vectores en el índice p15
- *Nanozostera japonica*: sin vectores en el índice p15
- *Pennisetum setaceum*: sin vectores en el índice p15
- *Prosopis juliflora*: sin vectores en el índice p15
- *Reynoutria x bohemica*: sin vectores en el índice p15
- *Lagocephalus sceleratus*: sin vectores en el índice p15
- *Lophocladia trichoclados*: sin vectores en el índice p15
- *Womersleyella setacea*: sin vectores en el índice p15
- *Stypopodium schimperi*: sin vectores en el índice p15
- *Acrothamnion preissii*: sin vectores en el índice p15
- *Kapraunia schneideri*: sin vectores en el índice p15
- *Carpobrotus acinaciformis*: sin vectores en el índice p15
- *Lophocladia lallemandii*: sin vectores en el índice p15

  Algunos son sinonimias o variedades que quizá existan bajo otro nombre (revisar caso a caso antes de cosechar).
- **Eval débil:** la mayoría de la lista UE tiene n = 10 filas de eval; un acierto de 100 % con n = 10 no es una medida fiable.
- **Confusión conocida:** *Pterois miles* (60 %) frente a *P. volitans*.

## Acciones propuestas

1. Cosecha local (sin iNat) para los taxones sin galería, empezando por los marinos de Weitzmann (*Lagocephalus sceleratus*, *Lophocladia*, *Womersleyella*) y los terrestres con presencia en Cataluña.
2. Ampliar eval a n ≥ 30 en los de categoría «establecida» o «en_expansion».
3. Confusores para el índice p17: pares *Pterois miles*/*P. volitans* y géneros con pares críticos.
4. Completar origen/vía/primera cita/impacto con DOI y el cruce CEEA.
