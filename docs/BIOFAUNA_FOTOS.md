# BioFauna Fotos — bulk download (admins)

FotoFauna (https://fotofauna.yespi.es) includes an admin-only panel **BioFauna
Fotos** (rail icon under Auto-ID, deep link `#bf-fotos`).

Admins browse the BioFauna photo catalog as a taxonomic tree, preview images
(lightbox + loupe), estimate size, and download **independent ZIP files**
(≤ 2 GB each — not split-volume archives). Each ZIP contains:

- one folder per species (photos + CSV)
- `README.txt` (origin, licences, FotoFauna / BioFauna / BioQuest)
- `00_LEEME.txt` and `Merge_FF_CSV_Files.bat` / `.ps1`

Extract every ZIP into the **same photos folder**, then run the `.bat` there.
It writes `FF_merged_species.csv` in that folder. Do not run the script from
inside the ZIP viewer.

A species folder stays in one ZIP unless that species alone exceeds ~2 GB.
Photos remain copyright of the original observers (typically Creative Commons
on Minka / iNaturalist). See
[terceros y licencias](https://fotofauna.yespi.es/legal/terceros-y-licencias.html).

Code lives in the private ops repo (`ecosistema-fauna`). Public AI engine:
[yespi/biofauna](https://github.com/yespi/biofauna).
