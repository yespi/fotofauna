# FotoFauna — Análisis personal Yespi

> **Fecha:** 2026-08-02 · **PRO:** `dca7fa21` · **Autor:** IA (sesión extensiva en HanSolo)
> Análisis tras ~8h de trabajo intensivo sobre el código, los bugs, y la experiencia de usuario.

---

## 🟢 Lo que funciona muy bien

- **Identificación con YOLOFauna + iNat CV**: pipeline robusto, fallback cuando YF no acierta, GPS mejora resultados.
- **Worker paralelo**: `MAX_CONCURRENT=2` con backoff diferenciado (429/red/timeout). Bien diseñado.
- **Autosave/sesión**: doble escritura localStorage + IndexedDB, guardado en `pagehide`, detección de sesión corrupta.
- **Thumbnails**: cold-load sin errores, blob temporal antes del canvas, regeneración solo cuando falta.
- **Auth**: cookies HttpOnly, refresh silencioso, CSP razonable.
- **API rate limiting**: anónimos rate-limited, usuarios exentos.
- **SEO**: 403 fichas, sitemap, robots.txt, contenido crawlable, PRE con noindex.

---

## 🟡 Mejoras prioritarias (impacto alto, esfuerzo bajo-medio)

### 1. Onboarding para nuevos usuarios
**Problema:** El onboarding actual son 3 pasos de texto. Sin fotos de ejemplo cargadas, la UI es un grid vacío con texto "Arrastra o selecciona tus fotos". No hay una demo, un vídeo, ni un botón de "probar con fotos de ejemplo".
**Solución:** Añadir un botón "Probar con fotos de ejemplo" que cargue 5-10 fotos de fauna marina (del dataset YF) con GPS predefinido. El usuario ve la experiencia completa sin tener que buscar sus propias fotos.
**Esfuerzo:** M (medio día)

### 2. Feedback visual durante identificación
**Problema:** Durante la identificación, las fotos pasan por estados `idle → processing → detected`. El indicador visual es un badge pequeño con "⌛ Procesando". No hay barra de progreso global ni estimación de tiempo restante.
**Solución:** Mostrar un contador "X/Y fotos identificadas" en la barra superior, con una animación sutil. Ya existe `idProgressMsgs` para el panel individual, pero falta el global.
**Esfuerzo:** S (1-2h)

### 3. Vista previa de especie en el grid
**Problema:** Cuando una foto se identifica, el nombre de la especie aparece en el badge inferior de la celda. Pero la foto en sí no cambia. El usuario tiene que hacer clic para ver la especie en el panel ID.
**Solución:** Añadir un overlay sutil en la celda (esquina superior derecha) con el icono de la especie y el nombre común. O un borde de color (verde = alta confianza, amarillo = media, gris = sin identificar).
**Esfuerzo:** M

### 4. Deshacer identificación automática
**Problema:** Si la IA identifica mal una especie, el usuario puede rechazarla (✗), pero la foto queda como "sin identificar". No hay un botón "deshacer" que restaure el estado anterior.
**Solución:** Guardar el historial de identificaciones por foto (máx 3) y permitir navegar entre ellas con ← → en el panel ID.
**Esfuerzo:** M

### 5. Atajos de teclado documentados
**Problema:** Hay atajos de teclado (Shift+click, Ctrl+click, flechas, Escape...) pero no están documentados en la UI. El usuario los descubre por accidente.
**Solución:** Añadir un botón "?" o "Atajos" en la barra inferior que muestre un modal con los atajos disponibles.
**Esfuerzo:** S

### 6. Filtros guardados
**Problema:** Los filtros del grid (estado, especie, fecha) se aplican pero no se recuerdan entre sesiones. El usuario tiene que re-aplicarlos cada vez.
**Solución:** Guardar el estado de los filtros en `localStorage` junto con la sesión.
**Esfuerzo:** S

---

## 🔴 Problemas de UX que duelen

### 7. El grid sin fotos es intimidante
**Problema:** Un usuario nuevo ve: un grid vacío, un panel izquierdo con "Ubicación", y un mensaje de "Arrastra tus fotos". No hay nada que invite a interactuar. La página de inicio (escritorio sin sesión) tiene los 3 pasos de onboarding, pero una vez dentro, el grid vacío es frío.
**Solución:** Mientras el grid está vacío, mostrar una galería de ejemplos (especies comunes) con un CTA "Carga tus fotos para identificar fauna". Similar al estado "mystery" del panel ID pero a escala grid.
**Esfuerzo:** M

### 8. Demasiada información en el panel ID
**Problema:** El panel de identificación muestra: foto recortada, especie sugerida, buscador, resultados de búsqueda, mapa de ubicación, notas, grupo de fotos... Todo en un espacio vertical de ~600px. Es abrumador.
**Solución:** Colapsar secciones por defecto. El buscador y los resultados solo se muestran al hacer clic en "Buscar especie". El mapa solo si hay GPS. Simplificar la jerarquía visual.
**Esfuerzo:** M

### 9. El panel de ubicación es complejo
**Problema:** El panel de ubicación tiene: buscador de direcciones, mapa Leaflet, puntos guardados, botones de zoom, y los marcadores de las fotos. Para asignar ubicación a una foto, el flujo es: buscar dirección → seleccionar resultado → clic en "Aplicar ubicación". No es obvio.
**Solución:** Simplificar a: (1) hacer clic en el mapa para poner un pin, (2) botón "Usar esta ubicación". El buscador de direcciones es secundario. Los puntos guardados en un desplegable.
**Esfuerzo:** L

### 10. No hay modo oscuro
**Problema:** La app usa variables CSS (`--surface`, `--text`, etc.) pero no hay toggle de tema claro/oscuro. Biólogos de campo usan la app de noche y el tema claro deslumbra.
**Solución:** Añadir toggle ☀/🌙 que persista en localStorage. Las variables CSS ya están; solo falta el switch y los colores del tema claro.
**Esfuerzo:** M

---

## 🟢 Mejoras de rendimiento (bajo esfuerzo)

### 11. Lazy loading de imágenes en el grid
**Problema:** El grid carga todas las miniaturas aunque no sean visibles (fuera del viewport). Con 100+ fotos, el navegador decodifica 100 imágenes.
**Solución:** Usar `loading="lazy"` en los `<img>` del grid (ya está) + `IntersectionObserver` para no renderizar celdas fuera del viewport. O usar `content-visibility: auto` en CSS.
**Esfuerzo:** S

### 12. Pre-carga de dependencias pesadas
**Problema:** `ff-vendor-loader.js` carga Leaflet, Cropper y JSZip bajo demanda. Esto evita parsear 280KB de JS al inicio, pero introduce latencia cuando el usuario abre el panel de ubicación/recorte por primera vez.
**Solución:** Pre-cargar los vendors en idle (requestIdleCallback) después de que el grid esté listo, sin bloquear la interacción inicial.
**Esfuerzo:** S

---

## 🟡 Funcionalidades que faltan

### 13. Exportar a CSV/JSON
**Problema:** El botón "Exportar" solo genera un ZIP con las fotos y un manifiesto. No hay forma de exportar los metadatos (especie, confianza, GPS, fecha) como CSV para análisis externo.
**Solución:** Añadir opción "Exportar metadatos" que descargue un CSV con todas las columnas.
**Esfuerzo:** S

### 14. Comparar fotos lado a lado
**Problema:** Cuando hay varias fotos del mismo individuo (grupo), solo se ve una a la vez. Para comparar ángulos o detalles, el usuario tiene que navegar con ← →.
**Solución:** Botón "Comparar" que muestre 2-4 fotos del grupo en una cuadrícula temporal.
**Esfuerzo:** M

### 15. Historial de sesiones
**Problema:** El autosave guarda una sesión, pero solo la última. Si el usuario quiere volver a una sesión anterior, no puede.
**Solución:** Guardar las últimas 5 sesiones en IndexedDB con timestamp. Mostrar lista "Sesiones anteriores" al cargar la app.
**Esfuerzo:** M

---

## 📊 Monolitos (deuda estructural)

### 16. PhotoGrid.js (3558 líneas)
84 funciones en `setup()`, template Vue de ~2250 líneas. Extraer:
- Template del panel ID → componente separado
- Template del panel ubicación → componente separado  
- Lógica de filtros → composable `use-filters.js`
- Lógica de exportación → composable `use-export.js`
**Esfuerzo:** L (varias sesiones)

### 17. use-recortar.js (2964 líneas)
El editor de recorte es un monstruo. Extraer:
- Renderizado del canvas → `crop-canvas.js`
- Controles de ajuste → `crop-controls.js`
- Lógica de filtros → `crop-filters.js`
**Esfuerzo:** L

---

## 🎨 Estética

### 18. La paleta de colores es monocromática
Verde oscuro (#0c140c) para fondos, verde claro para acentos. Funciona pero es... sosa. No transmite "fauna marina" ni "naturaleza". Los badges de especie podrían tener colores por grupo taxonómico (peces = azul, moluscos = naranja, etc.).

### 19. Los iconos son emojis
📍⚠️🔍↻ — funcionan pero no son consistentes entre plataformas. Un set de iconos SVG personalizados (incluso 5-6) mejoraría mucho la percepción de calidad.

### 20. La home de escritorio sin sesión está bien
Los 3 pasos "Sube / IA detecta / Publica" con las capturas son efectivos. El CTA "Arrastra o selecciona tus fotos" es claro. No tocar.

---

## 📋 Resumen por esfuerzo

| Esfuerzo | Tareas |
|----------|--------|
| **S** (1-2h) | #2 Feedback identificación, #5 Atajos teclado, #6 Filtros guardados, #11 Lazy images, #12 Pre-carga vendors, #13 Exportar CSV |
| **M** (medio día) | #1 Onboarding demo, #3 Vista previa grid, #4 Deshacer ID, #7 Grid vacío ejemplos, #8 Panel ID colapsable, #10 Modo oscuro, #14 Comparar fotos, #15 Historial sesiones |
| **L** (varias sesiones) | #9 Panel ubicación, #16 PhotoGrid split, #17 use-recortar split |
| **Estética** | #18 Paleta colores, #19 Iconos SVG, #20 Home (OK) |

**Si solo pudiera hacer 3 cosas, haría:**
1. #1 Onboarding con fotos de ejemplo (retención de nuevos usuarios)
2. #10 Modo oscuro (usan la app de noche)
3. #3 Vista previa de especie en el grid (gratificación inmediata al identificar)

---

## 🔧 Lo que NO tocaría

- El worker de identificación: está bien diseñado y es estable.
- El autosave/sesión: robusto tras los fixes de hoy.
- El pipeline YF→iNat: funciona, los geo priors lo mejorarán.
- La auth: correcta, sin fugas.
- SEO: bien implementado, no reabrir.