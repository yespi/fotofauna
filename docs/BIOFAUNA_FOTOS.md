# BioFauna Fotos — bulk download (admins)

FotoFauna (https://fotofauna.yespi.es) includes an admin-only panel **BioFauna
Fotos** (rail under Auto-ID, `#bf-fotos`).

Admins browse the live BioFauna gallery as a taxonomic tree, preview images
(lightbox + loupe), estimate size, then pick **taxonomic folders** or a
**flat species-folder layout** and download **independent ZIP files** (≤ 2 GB
each — not split-volume archives). Each ZIP contains:

- photos + CSV in each species folder
- `README.txt` (origin, licences, FotoFauna / BioFauna / BioQuest)
- `00_LEEME.txt`
- `DOBLE_CLIC_Unir_CSVs.bat` — **run this** (double-click)
- `NO_ABRIR_motor_unir_CSVs.ps1` — engine used by the `.bat`; do not open it

Extract every ZIP into the **same photos folder**, then run the `.bat` there.
It walks all subfolders and writes `FF_merged_species.csv` at that root.
Do not run scripts from inside the ZIP viewer.

A species stays in one ZIP unless that species alone exceeds ~2 GB.
Photos remain copyright of the original observers (typically Creative Commons
on Minka / iNaturalist). See
[terceros y licencias](https://fotofauna.yespi.es/legal/terceros-y-licencias.html).

Code lives in the private ops repo. Public AI engine:
[yespi/biofauna](https://github.com/yespi/biofauna).
