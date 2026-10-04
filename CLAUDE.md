# Kolam Studio

A voice-controlled web app that generates and animates traditional South Indian **sikku kolam**: continuous line patterns drawn around a grid of dots (pulli). The user speaks a command such as "draw a 5 by 5 kolam" or "make it red", and the pattern draws itself stroke by stroke, as if traced by hand in rice flour.

## Architecture

- **One file:** the whole app lives in `index.html`, with markup, a `<style>` block and a `<script>` block.
- **No framework, no build step:** plain HTML, CSS and vanilla JavaScript (ES2020+). Don't add npm, bundlers, TypeScript or libraries.
- **Rendering:** HTML `<canvas>` with the 2D context. Scale for `devicePixelRatio` so lines stay sharp, and animate with `requestAnimationFrame`.
- **Voice:** Web Speech API (`SpeechRecognition` / `webkitSpeechRecognition`). Map recognised phrases to commands with simple keyword matching.
  - Always provide on-screen controls as well, because Firefox and some other browsers don't support speech recognition.
  - Show a clear listening state and repeat the recognised phrase back on screen.
- **External resources:** Google Fonts only.
- **Running it:** open `index.html` in Chrome or Edge. Microphone access needs `localhost` or HTTPS, so for voice use `npx serve` or `python -m http.server`.

### Code conventions

- Organise the script into clear sections: state, grid/dot geometry, kolam path generation, canvas rendering, animation, voice commands, UI.
- Keep a single `state` object as the source of truth. UI and voice both change state, and the renderer reads from it.
- Build kolam paths as data (arrays of points and curves) before drawing them, so they can be animated, replayed and exported.
- Respect `prefers-reduced-motion`: when it's set, draw the kolam immediately and skip decorative motion.

### Kolam generation (Gerdes mirror curves)

- Doubled coordinates: dots at (odd, odd), mirror slots between neighbouring dots at (x + y odd). The line steps diagonally slot to slot; each slot is a crossing, an H mirror (flips dy) or a V mirror (flips dx). The frame always reflects.
- Mirror states are assigned per symmetry orbit (90° turns swap H↔V), then the loops are counted. The generator retries and then hill-climbs until there is exactly one loop. Seeds make results reproducible.
- **Symmetry limits (found by testing, keep them):** any diagonal reflection, and therefore 8-fold symmetry, never gives a single loop. On even grids, any 180° symmetry always gives an even number of loops, so even grids fall back to a left-right mirror.
- Rendering: cubic Béziers between segment midpoints. Pieces are straight at crossings and quarter circles (handle 0.5523) at mirrors, so the line wraps each dot in a true circle.
- The line is filled as variable-width ribbons, one sub-path per piece, all wound counterclockwise so the nonzero fill keeps crossings solid. The thickness waves are periodic over the loop so the closing seam is invisible. Keep the round caps counterclockwise too, or they punch holes.

### Rangoli mode

- `generateRangoli(seed, folds)` returns pure data: a centre flower plus 5–7 rings (petals, teardrops, dots, diyas, scallops), with unit radii and palette *slots*. Colours come from `THEMES[theme].palette`, so a theme change recolours the same design (`recolourRangoli`).
- Each ring renders to its own cropped canvas (`buildRangoliLayers`) with roughened edges, a same-colour blur and powder grain. On screen, rings are revealed by a clockwise wedge clip, from the centre outward. Layers are rebuilt about 180 ms after a resize settles.
- `state.mode` is `'kolam' | 'rangoli'`. `setMode` remembers the last design of each mode and restores it on switching back.

### Designs, gallery and export

- **No repeats:** a design's identity is its fingerprint: a hash of the kolam's mirror layout, or the rangoli's folds and seed. New designs retry seeds until the fingerprint is neither in `seenDesigns` (this session) nor in the gallery. Explicit seeds (gallery, weave or symmetry tweaks) skip the check.
- **Gallery:** `localStorage` key `kolam-studio-gallery` holds recipes (mode, seed, settings, theme, line colour) plus a JPEG thumbnail. Opening an item regenerates it exactly. On a first visit it is seeded with `CLASSICS`.
- **Stills:** `renderStill(px, design)` swaps the global `ctx`/`dpr` to render a finished design off-screen. Every size is in CSS pixels times `dpr`, so stills match the screen. Export is 3000 px with a watermark.

### First-visit intro

- An inline `<head>` script adds `html.intro-on` on a first visit (no `kolam-studio-intro-seen` in `localStorage`) or when the URL has `?intro`, so the app never flashes first. While it's set, `.enter` entrances are paused and the header's children are hidden.
- The glide measures each intro word with its transform cleared, then translates and scales it from its top-left corner onto its header twin. Font, line height and letter spacing must stay identical between `.intro-title`/`.brand` and `.intro-tamil`/`.brand-tamil`, or the landing jumps.
- Headless Chrome screenshots don't show CSS transitions faithfully. Verify motion by reading final boxes with transitions disabled, or in a real browser.

### Voice

- Uses `SpeechRecognition` with `lang = 'en-IN'`, continuous mode, interim results and 4 alternatives. Chrome ends a session after silence, so `onend` restarts it while `state.listening` is true.
- `parseCommand` is deliberately forgiving: number words, "9x9" / "nine by nine", mishearings of "kolam" (column, colon…) and "grid" (grade, greed…). One phrase can carry several actions. Every alternative is tried before showing a hint.
- Voice and the panel share `applyGridSize`, `selectColor`, `drawKolam` and `clearKolam`, so the UI always stays in sync.
- Space toggles the mic everywhere except text fields.
- Tamil: the header toggle sets `state.lang` (`en-IN` / `ta-IN`). `translateTamil` maps Tamil stems, suffixes included, to the English words the parser knows, and `TRANSLITERATED` catches Tamil heard by the English recogniser ("column board" = kolam podu, "R" = azhi). Add new Tamil words there, longest stems first.

### Performance and layout notes

- `render()` caches the whole scene (mirrors, line or rangoli, dots) in `sceneCache` once nothing in it moves. Effects (Deepam lamps, Diwali sparkles, dust) draw on top every frame. Anything new that animates in the scene must be added to `sceneIsStill`.
- Dots are stamped from `dotSprite` canvases, not built from gradients each frame.
- Grid tracks holding the canvas must be `minmax(0, 1fr)`. An `auto` track sizes itself to the canvas's pixel width, which loops with the layout.
- Phones (≤ 900 px): the panel is a bottom sheet (`setPanelOpen`, pointer drag on the handle and title), and the orb hides while it's open.
- Headless Chrome on Windows won't go below about 500 px wide. Use 500 px for phone screenshots, and `--force-prefers-reduced-motion` for complete, unanimated captures.

## Design system: "Courtyard at Dusk"

The look should be premium, cinematic and culturally rooted, like a freshly washed granite courtyard at twilight with a rice-flour kolam glowing on it. Every interaction should feel calm, intentional and high-end.

### Colour tokens

Define these once on `:root` and always use the variables, never raw hex values elsewhere.

```css
:root {
  /* Theme tokens (registered with @property so themes crossfade) */
  --granite:     #0e0b0a; /* background */
  --rice-flour:  #f5efe3; /* text and pulli dots */
  --accent:      #c9a24c; /* UI accent: borders, thumbs, focus rings, dividers */
  --flame:       #c9a24c; /* voice orb, lamp tip, dust, shimmer */
  --theme-line:  #f5efe3; /* the theme's kolam line */

  /* Fixed swatch colours */
  --kolam-line:  #eae833;
  --kumkum-red:  #ba322a;
  --brass-gold:  #c9a24c;

  /* Derived: always color-mix() from theme tokens, never fixed rgb() */
  --rice-flour-muted: color-mix(in srgb, var(--rice-flour) 62%, transparent);
  --surface:          color-mix(in srgb, var(--rice-flour) 4%, transparent);
  --hairline:         color-mix(in srgb, var(--accent) 22%, transparent);
}
```

- The background covers most of the screen. Use the accent and flame colours sparingly.
- Keep text contrast at WCAG AA or better against `--granite`.
- Any new colour in CSS must derive from a theme token, or it won't follow theme changes.

### Festival themes

- `:root[data-theme="pongal" | "diwali" | "deepam" | "margazhi"]` overrides the five theme tokens. Dusk is the default.
- The tokens transition over 800 ms on `:root`. The canvas reads them every frame through `css()` and `colorToRgb()`, which accepts the `rgb()` that registered properties compute to. Keep `render()` busy during the crossfade.
- Canvas-only differences live in `THEMES` in the script: line glow, Diwali sparkles and Deepam corner lamps.
- The chosen theme is saved in `localStorage` and restored in `<head>` before first paint.
- `render()` is called both from rAF and directly, so `dt` must be clamped to 0 or more. A negative `dt` once made the effects appear in the wrong themes.

### Typography

Load all three fonts from Google Fonts.

| Role | Font | Notes |
|---|---|---|
| Display and headings | **Cormorant Garamond** (500/600, italic for accents) | Large sizes and generous tracking; this is the cinematic voice |
| UI text | **Inter** (400/500) | Buttons, labels, status, body; 14 to 16px |
| Tamil text | **Noto Serif Tamil** (400/600) | Tamil names and captions, such as கோலம்; set `lang="ta"` |

```css
--font-display: "Cormorant Garamond", Georgia, serif;
--font-ui:      "Inter", system-ui, sans-serif;
--font-tamil:   "Noto Serif Tamil", serif;
```

### Space, shape and depth

- **White space:** be generous and use an 8px spacing scale (8, 16, 24, 32, 48, 64, 96). Let the kolam canvas be the hero, with few UI elements around it.
- **Corners:** `--radius: 16px` on cards, panels, buttons and dialogs. Use pill shapes only for small chips.
- **Shadows:** soft and low-contrast, for example `0 12px 40px rgb(0 0 0 / 0.45)`. A faint warm glow (`0 0 24px rgb(201 162 76 / 0.15)`) can mark active elements.

### Motion

```css
--ease-out:  cubic-bezier(0.22, 1, 0.36, 1);
--dur-fast:  300ms;
--dur-base:  450ms;
--dur-slow:  600ms;
```

- UI transitions use ease-out curves between 300 and 600ms: fades, small lifts (4 to 8px) and gentle scale changes (0.98 to 1).
- Avoid bounces, springs or anything abrupt.
- Kolam drawing animates along the path at a steady, hand-traced pace, with soft fade-ins for the dots before the lines begin.
- The listening state is a slow, breathing pulse in `--kumkum-red`, never a fast blink.
