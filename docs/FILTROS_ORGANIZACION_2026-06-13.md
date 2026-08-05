# FotoFauna — Organización filtros (13-jun-2026)

**Build:** `2e42ac2a` (PRO) · Backend `vision_image.py`  
**Changelog completo:** [`CHANGELOG.md`](CHANGELOG.md) (entrada 2026-06-13)

---

## Principio

| Regla | Motivo |
|-------|--------|
| **8 filtros globales** + **💨 Antipartículas** local | Global quitado — el inpaint total emborronaba; local es más seguro |
| **✨ Auto** sin Before/After | Filtro más usado; BA ralentiza edición masiva |
| Barra **una sola línea** | `Exp` · `Cor` · `Col` + scroll horizontal |
| Exposición **mutuamente excluyente** | Auto / Subexp. / Sobreexp. / Contraste — uno solo |
| **Sin batch en lote** | Dock inferior eliminado (jun-2026) |

---

## Barra en Recortar (una línea)

```
Exp ✨ 🌙 ☀️ ◑  |  Cor 🔍 🎯 ☁️ 💨  |  Col 🌊 🔴
                              ↑ Antipartículas (herramienta local)
```

---

## Grupos lógicos

| Grupo | Filtros | Rol |
|-------|---------|-----|
| **Exposición** | ✨ Auto, 🌙 Subexp., ☀️ Sobreexp., ◑ Contraste | Luminosidad (1 activo) |
| **Corrección** | 🔍 Nitidez, 🎯 Enfocar, ☁️ Bruma, **💨 Antipartículas** | Detalle, atmósfera, motas locales |
| **Color** | 🌊 Marina, 🔴 Rojos | Balance cromático |

---

## Cuándo usar cada uno

| Filtro | Usar cuando… | Evitar cuando… |
|--------|--------------|----------------|
| **✨ Auto** | Edición rápida; foto «casi bien» | Necesitas control fino |
| **🌙 Subexp.** | Sombras profundas | Foto equilibrada — empieza al **60–70%** |
| **☀️ Sobreexp.** | Luces quemadas | Subexposición — empieza al **60–70%** |
| **◑ Contraste** | Imagen plana | Ya tiene buen contraste |
| **🔍 Nitidez** | Ligera falta de foco / JPEG | Muy borrosa |
| **🎯 Enfocar** | Blur real (movimiento, agua) | No altera brillo global |
| **☁️ Bruma** | Velo azul/verde, agua turbia | No sustituye motas puntuales |
| **💨 Antipartículas** | Polvo flotante / backscatter **en zonas concretas** | No es filtro global — pinta mota a mota |
| **🌊 Marina** | Dominante azul submarino | Foto terrestre |
| **🔴 Rojos** | Dominante roja (algas, flash) | Balance OK |

---

## Intensidad (dial flotante)

Aparece cuando hay filtros activos con slider (intensidad). Debounce ~380 ms.

---

## Antipartículas 💨

1. Pulsa **💨 Antipartículas** en el grupo **Cor**.
2. Ajusta **⌀ pincel** (default 56).
3. **Pinta** sobre las motas (trazo azul).
4. Al soltar → API inpinta **solo las motas** en esa zona (no el fondo).
5. **↩ Deshacer** trazo (Ctrl+Z en modo antipartículas) o undo global ↩.
6. **✓ Listo** o **Esc** para salir.
7. Si activas otro filtro, se cierra antipartículas automáticamente.

---

## Before/After

No disponible para antipartículas ni ✨ Auto.
