# Verso LMG — Mapa de estructura

> Referencia rápida para no tener que releer `index.html` (2750 líneas) ni
> `jsx/host.jsx` (2486 líneas) completos en cada sesión. Actualizar esta tabla
> cuando cambien secciones grandes o se resuelvan bugs de plataforma nuevos.
>
> Panel CEP de After Effects para timestamping de letras/SRT, 11 canales.
> Corre dentro de AE (CSInterface) y como web standalone (`IN_AE` = false).

## Archivos

| Archivo | Líneas | Rol |
|---|---|---|
| `index.html` | ~2750 | UI + toda la lógica del panel (CSS + JS inline, un solo `<script>`) |
| `jsx/host.jsx` | ~2486 | ExtendScript (ES3) que corre dentro de AE: manipula comps/capas |
| `CSInterface.js` | ~103 | Bridge CEP recortado a mano (NO es el oficial de Adobe — ver más abajo) |
| `CSXS/manifest.xml` | — | Manifest de la extensión (nombre "Verso LMG") |

Instalado en Mac/Win en `…/Adobe/CEP/extensions/Verso LMG/` — es una **copia
separada** de esta carpeta maestra. Cualquier cambio aquí hay que copiarlo a la
carpeta instalada para que AE lo vea.

---

## `index.html` — CSS (líneas 1–478)

- `1–7` — head, CSP (`connect-src` permite `lrclib.net` para autosync)
- `10–32` `:root` — design tokens: paleta gris (`--accent:#8a8a8a`, antes
  indigo), fondo `--bg:#0E0E0E`, contenedores `--container:#171717`
- `35–` `.app` shell, `.songs` (lista de canales, ancho fijo 330px, oculta en
  el layout actual — ver nota abajo), `.rsz-h`/`.rsz-v` (resizers)
- `92–111` `.lyrics`, `.lyr-stage`, `.lyr-flow` (texto con transform
  translateY, transición `.55s` = deriva karaoke), `.lyr-fade` top/bot
  (degradados con paradas suavizadas — banding corregido, ver Diseño)
- `113–188` **TIMELINE ZONE**: `.tl-track`, `.tl-marker` (marcadores grises),
  `.tl-playhead`, `.t-center-timer` (timer + `.t-pbar` barra de progreso)
- `362–476` **"VERSO 2 REDESIGN"**: rediseño de pills flotantes + Details
  panel; `#songsPanel,#rszH{display:none!important}` — la lista de canales
  clásica está oculta, todo vive en el Details panel flotante (`.details-panel`)
  que se abre sobre la letra

## `index.html` — HTML `<body>` (líneas 478–577)

```
.app
  .workspace
    #songsPanel (oculto, ver arriba)
    #rszH
    #rightCol
      #lyricsPanel
        #detailsPanel      ← floating card: 11 canales + barra global
        #lyrContent
          .lyr-title (#lyrSong, #lyrArtist)
          #lyrStage (.lyr-fade top, #lyrFlow, .lyr-fade bot)
          .lyr-controls    ← timer/status, lo arma buildActionBar() en JS
      #rszV
      #timelineZone
        .tl-row (#tlLeft, .tl-mid > #tlWrap > #tlTrack+#tlMarkers+#tlPlayhead, #tlRight)
        #tlMini (oculto, minimap removido)
#eov (modal export SRT, solo web)
#asov (modal Smart Sync / autosync LRCLIB+Genius)
#rateMenu, #importMenu, #toast
#fin, #finMulti (inputs file ocultos)
```

## `index.html` — JavaScript (líneas 583–2750)

Secciones marcadas con `// ─── NOMBRE ───` en el código — buscar con Grep
`// ─── ` para saltar directo:

| Línea | Sección | Notas |
|---|---|---|
| 583 | CEP BRIDGE | `csInterface`, `IN_AE`, `IS_MAC`, `IS_TOUCH` — single source of truth |
| 601 | AUDIO / TIMELINE STATE | `dur`, `zoom`, `scrollPct`, `stamps[]` |
| 606 | ACTIVE WORKING SET | espeja el canal activo |
| 611 | CHANNELS | array de 11 canales fijos |
| 629 | INTERACTION STATE | drags, `tlDirty`, etc. |
| **643** | **BOOT** | listeners de `AEL` (audio element), **foco/teclado Mac vs Win** (ver sección dedicada abajo), `_registerKeys()` |
| 706 | TOUCH | pan/pinch-zoom timeline (iPad) |
| 794 | CHANNELS fijos | mapeo canal → nombres de comp |
| 812 | control-bar icons | SVGs inline de los botones |
| 930 | DETAILS PANEL open/close | |
| 972 | Barra de progreso | `startProgress/setProgress/stopProgress/_pbarHide` — con red de seguridad anti-atasco (ver Bugs resueltos) |
| 1161 | HOST LOADER | `ensureHost`, `probeHost`, carga `host.jsx` vía XHR si falta |
| 1206 | SEED CHANNELS | |
| 1221 | VIDEO / SHORT MODE | `SHORT_MODE` toggle, afecta casi todas las acciones |
| 1272 | RENDER CHANNELS | pinta la lista de 11 canales |
| **1620** | **TIMELINE** | `renderMarkers`, `renderTimeline`, `positionPlayhead` — marcadores solo se re-renderizan si `tlDirty` |
| **1709** | **RAF / FRAME** | `schedRAF`/`frameUpdate` — loop de reproducción, llama `renderTimeline`+`updateLyricHighlight` cada frame |
| 1528 | LYRICS PANEL | `renderLyricsFlow`, `updateLyricHighlight` (centrado karaoke) |
| 2018 | STAMP | `doStamp()` — coloca marcador en línea actual |
| 2042 | UNDO / REDO | `pushUndo/undoAction/redoAction`, stack de 80 |
| **2047** | **KEYBOARD** | `onKey(e)`, **`_registerKeys()`** — CRÍTICO, ver sección dedicada |
| 2120 | GLOBAL MOUSE | drag de marcadores, resize de paneles |
| 2164 | EXPORT / IMPORT | export SRT web, import SRT en AE |
| 2315 | MASTER / RENDER | `masterRun`, `doRenderOrQueue` (llaman a host.jsx) |
| 2370 | THUMBNAIL | actualiza comps "… TH" |
| 2403 | IMPORT MENU | |
| 2459 | AUTO-SYNC | LRCLIB (fallback, preserva secciones de Genius) |
| 2519 | GENIUS BULK | |
| 2727 | UTILS | `fmtClock`, `esc`, etc. |

---

## `jsx/host.jsx` — mapa de funciones (ExtendScript ES3)

Sin `const/let/arrow/template literals` — ES3 puro. Funciones clave por tema:

**Diálogos/CEP bridge** (líneas 12–33, 2227–2280):
`_isWin`, `_dlgFilter`, `_AUDIO_FILTER`/`_SRT_FILTER` (strings literales — NO
recursivas, ver Bugs resueltos), `_notify`/`_writeResult`/`_writeRaw`/
`_clearResult`/`_readResultText` (triple redundancia para el bug de
evalScript-pierde-return-con-dialogo-modal)

**Import SRT** (19–301, 2065–2203):
`saveSRTFile`, `_parseSRT`, `importSRTToComp`, `importViaStyleController`,
`importLyricsToComp`, `importLyricsToMany`, `bulkImportOne`,
`importSRTBatch(isShort)` — matchea por número líder de archivo (`_leadNum`)

**Capas de estilo/keyframes** (485–1991):
`applyStyleToSRTLayers`, `_buildStyleFrame`, `_buildLyricLayers`,
`_remapWholeLayerKeys`, `_repositionAllKeyframes`, `_retimeLayerTextAnims` —
el motor de reposicionar keyframes al reimportar SRT

**Clasificación de comps** (947–1092):
`_isShortName` (regex `\bShort\b`), `isTHComp`, `getGeneralComps(isShort)`,
`_allLyricsComps`, `_allTHComps`, `getSongItems`, `_songInComp`

**Range / Master** (1180–1390):
`adjustRangeComp` (extiende Song a duración completa antes de fijar work
area), `_shortMainTwin`, `_shortTwinName`/`_videoTwinName`/`_shortTwinByName`
(mapeo video↔short por nombre), `masterRun(isShort)`

**Render** (1399–1484):
`doRenderOrQueue(sendToAME, isShort)` — filtra comps por **label de color**:
`label 11` (naranja) = main video, `label 2` (amarillo) = main short, TH vía
`isTHComp` excluyendo Short TH. **NO por número/nomenclatura** — ver Bugs
resueltos, esa fue la causa de que comps como "01 Distrito" o "AS Scenes"
entraran al render por error.

**Canales por número** (1492–1637):
`_leadNum`, `_lyricsCompByNum`, `_thCompByNum`, `_songByNum`,
`getProjectChannels` — todo el mapeo canal-número vive aquí

**Thumbnails** (2392–2486):
`updateThumbnailComp`, `updateThumbnailBoth` (video+short), mode-aware
fallback para no confundir TH de video con Short TH

---

## Mac vs Windows — foco y teclado (LA PARTE DELICADA)

Esta fue la parte más iterada de todo el panel. **Está funcionando — no tocar
sin razón fuerte.** Resumen de la arquitectura final:

### El bug real (causa raíz)
`CSInterface.js` de este panel es una versión **recortada a mano** (no la
oficial de Adobe) que originalmente **no implementaba `registerKeyEventsInterest`**.
Se agregó en líneas ~96–103 de `CSInterface.js`:
```js
CSInterface.prototype.registerKeyEventsInterest = function (keyEventsInterest) {
  if (!window.__adobe_cep__ || !window.__adobe_cep__.registerKeyEventsInterest) return false;
  return window.__adobe_cep__.registerKeyEventsInterest(keyEventsInterest);
};
```
Sin esto, cualquier llamada a `csInterface.registerKeyEventsInterest(...)` era
un no-op silencioso — por eso Space reproducía la composición de AE en vez del
panel, sin importar qué keyCode se usara.

### Por qué Windows nunca lo necesitó
En Windows basta con `preventDefault()`/`stopPropagation()` en el `keydown` a
nivel DOM (capture phase, `window.addEventListener('keydown', onKey, true)`)
— CEF en Windows controla el enrutado de teclado por completo una vez
enfocado. En Mac, AE intercepta Space/Cmd+Z a nivel nativo (Cocoa) **antes**
de que el DOM pueda actuar — de ahí la necesidad real de
`registerKeyEventsInterest`.

### Foco (index.html, boot ~línea 650–670)
- **Windows**: `reclaimFocus()` llama `window.focus()` + `blur()` de botones,
  en cada `mousedown`/`pointerdown`/`touchstart` (captura). Funciona perfecto,
  **no tocar**.
- **Mac**: `reclaimFocus()` es **no-op total** (`if(!IN_AE || IS_MAC) return;`).
  `window.focus()` en Mac rompía el play y sacaba del panel — se intentó y se
  revirtió. En Mac el clic nativo ya enfoca el WebView; no hace falta más.

### Registro de teclas (`_registerKeys()`, index.html ~línea 2065)
- **Windows**: formato `K(code, ctrl, meta, shift)` con las 4 banderas
  siempre explícitas. keyCode JS estándar (Z=90, Space=32, Y=89...).
- **Mac**: mismo keyCode JS (32 para Space — el bug antiguo usaba el virtual
  de macOS 49, INCORRECTO). Para combos **sin modificador** (Space, Enter,
  Delete) usa formato mínimo (`m(code)`, solo `keyCode`). Para combos **con
  modificador** (Cmd+Z, Ctrl+Z, Cmd+Y) usa el formato explícito de 4 banderas
  (`K2`, igual que Windows) porque el formato mínimo con modificador nunca se
  confirmó — se reforzó tras confirmar que Space funcionaba pero antes de
  probar undo/redo. Además incluye respaldos con virtual keycodes de macOS
  (Z=6, Y=16, Space=49, etc.) por si esta build de CEP los matchea así.
- Se re-registra en `boot` con `setTimeout` a 800ms y 2500ms (por si el
  registro inicial ocurre antes de que CEP esté listo en Mac) y en el evento
  `focus` de la ventana.
- Hay un **toast de diagnóstico** (solo Mac, una vez): "Verso: teclas
  registradas (N)" si el registro corrió sin lanzar, o warning si
  `registerKeyEventsInterest` no existe. Útil para debug futuro sin adivinar.

### `onKey(e)` (index.html ~línea 2048)
Lógica de teclado del lado panel (independiente de plataforma): Cmd/Ctrl+Z
→ undo, +Shift → redo, Cmd/Ctrl+Y → redo, Space → `togglePlay()`, Enter →
`doStamp()`, Delete/Backspace → borra marcadores seleccionados. Ignora si el
foco está en INPUT/TEXTAREA/SELECT.

### Si hay que tocar esto en el futuro
1. **NO reintroducir manipulación de foco en Mac** (`window.focus()`,
   `.focus()` a cualquier elemento) — causaba que se saliera del panel.
2. Si un atajo nuevo no funciona en Mac, sospechar primero de
   `registerKeyEventsInterest` (¿está en el array de `_registerKeys`? ¿con
   las banderas completas si lleva modificador?) antes que de `onKey`.
3. El toast de diagnóstico ya está — si algo falla, pedirle al usuario que
   reporte qué toast ve al abrir el panel.

---

## Diseño (colores/gradientes) — notas de decisiones tomadas

- Paleta: gris opaco `#8a8a8a` como acento (reemplazó indigo original).
  Fondo general `#0E0E0E`, contenedores `#171717`.
- Letra activa = blanco `.98` peso 600; siguiente = blanco `.5`; pasada =
  blanco `.5`; futura = blanco `.26`.
- `.lyr-fade` top/bot: gradientes con **paradas suavizadas** (varios stops en
  vez de 2) para evitar banding (bandas Mach) visible en el CEF de AE, que no
  hace dithering de gradientes lineales simples.
- `.t-pbar` (barra de progreso): ancho fijo `72px` + `margin:3px auto 0` para
  centrado exacto bajo el timer, independiente de resoluciones de layout flex
  distintas entre Win/Mac.

---

## Bugs resueltos (para no repetirlos)

1. **`_dlgFilter` recursión infinita** — un script de reemplazo sustituyó
   strings literales dentro de la propia definición del helper. Lección:
   validar por EJECUCIÓN (Node `new Function()`), no solo sintaxis.
2. **SRT solo importaba a Short, no a video** — `_selectedLyricsComps()`
   devolvía comps Short seleccionadas; se corrigió con enumeración explícita
   de Lyrics de video.
3. **Botón Import usaba solo `ch.shortLyr`** — se corrigió a
   `importLyricsToMany([ch.lyr, ch.shortLyr])`.
4. **Short TH no actualizaba** — fallback devolvía el TH de video. Se hizo
   mode-aware.
5. **Render incluía comps ajenas** ("01 Distrito", "AS Scenes") — el filtro
   por número era demasiado amplio. Solución final: filtro por **label de
   color** (11 naranja / 2 amarillo), no por nomenclatura numérica.
6. **Marker adjustment saltaba de línea** — optimización de
   `updateLyricHighlight` pintaba `nextLi` como activa en huecos. Fix:
   `activeLi` solo para la línea realmente activa.
7. **Barra de progreso pegada en Mac** — se agregó auto-ocultado de seguridad
   (`_safety` timeout) por si algún camino no llamaba `stopProgress`.
8. **Play/teclado en Mac** — ver sección dedicada arriba. Causa raíz:
   `registerKeyEventsInterest` faltante en `CSInterface.js`.

---

## Pendiente

- Replicar el conjunto completo de fixes de Verso LMG a **Verso** (single-
  channel) y **Verso CM** (11 canales) — directiva del usuario desde el
  inicio: probar todo en LMG primero, replicar después de confirmar.
  A la fecha de este documento: Mac/teclado, foco, barra de progreso y
  gradiente de letra están **confirmados funcionando en LMG**, aún no
  replicados a Verso/CM.
