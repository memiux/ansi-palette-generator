# ANSI Palette Generator — Claude Code Project

## Overview

A self-contained, single-file HTML tool for designing 16-color ANSI terminal palettes in pastel style. 
The primary use case is generating color values for a **Zed editor theme**, but the output is generic 
enough for any terminal emulator or editor theme format.

**File:** `ansi-palette-generator.html` — zero dependencies, no build step.

---

## Development

Open `ansi-palette-generator.html` directly in a browser — no server, no install, no build. Edit the file, save, refresh.

---

## What the Tool Does

- Displays 24 swatches: 3 rows (Dim / Normal / Bright) × 8 colors
- Colors are generated algorithmically from OKLCH parameters, not hardcoded
- Sliders for per-group L and C, per-color hue angles (including Black and White), Black lightness factor (Black L), White lightness factor (White L), neutral saturation (Neutral S), and a global temperature offset
- Dark/Light mode toggle (each mode has independent L/C params)
- Preset selector with built-in presets (Default, Warm, Cool, Nord-ish, Muted, Vivid, Kizuna AI) and user-saved custom presets persisted in localStorage
- Export current state as a named JSON file; import JSON to restore a saved state
- WCAG contrast badge on each swatch (ratio + AAA/AA/A/fail dot)
- `<pre>` palette showing all hex values, grouped and sorted in rainbow order
- `title` attribute on each swatch shows the resolved hex value

---

## Architecture

### Color Model — OKLCH

HSL was deliberately abandoned in favor of **OKLCH** because HSL is perceptually
non-uniform: blue at `hsl(240, 50%, 75%)` looks much darker than yellow at the
same values. OKLCH has a uniform lightness axis so equal `L` values look equally
bright across all hues.

```
oklch(L  C  H)
  L = perceptual lightness, 0–1
  C = chroma (colorfulness), 0–0.4 practical range
  H = hue angle, 0–360°
```

**Canonical hue angles used:**

| Color   | H (°) |
|---------|-------|
| Red     | 29    |
| Yellow  | 109   |
| Green   | 142   |
| Cyan    | 194   |
| Blue    | 264   |
| Magenta | 328   |

These differ significantly from HSL hue angles. Do not confuse them.

**Default L/C per group (dark mode):**

| Group  | L    | C    |
|--------|------|------|
| Dim    | 0.55 | 0.07 |
| Normal | 0.72 | 0.10 |
| Bright | 0.82 | 0.13 |

**Default L/C per group (light mode):**

| Group  | L    | C    |
|--------|------|------|
| Dim    | 0.42 | 0.07 |
| Normal | 0.52 | 0.10 |
| Bright | 0.60 | 0.13 |

Light mode L values are ~0.13–0.22 lower so colors stay legible against `#f5f5f5`. C values are identical across modes.

### Neutrals (Black / White)

Black and White have independent hue sliders (full range 0–360°, defaults Black=60°,
White=260°). Their lightness is derived from the group's L value via per-neutral factors:

```js
neutralFactor = { Black: <Black L slider, default 0.42>, White: <White L slider, default 1.05> }
neutralL = clamp(groupL * factor, 0.05, 0.98)
```

`neutralFactor.Black` is user-adjustable via the **Black L** slider (range 0.10–0.80, default 0.42). `neutralFactor.White` is user-adjustable via the **White L** slider (range 0.50–1.50, default 1.05).

Neutral chroma is driven by three inputs combined:

```js
neutralC = (c * 0.3 + Math.abs(temperature) * 0.003) * neutralSat
```

- `c * 0.3` — scales with the group's C slider (neutrals at ~30% of chromatic saturation)
- `Math.abs(temperature) * 0.003` — small additive boost from temperature magnitude
- `neutralSat` — global multiplier (0–1) that gates both contributions; at 0 neutrals are fully achromatic

Temperature also offsets the neutral hue angle: `hues[name] + temperature`, same as chromatic colors.

**Default hue angles for neutrals:**

| Color | H (°) |
|-------|-------|
| Black | 60    |
| White | 260   |

### Temperature

A global hue offset applied to all colors (chromatic and neutral): `hues[color] + temperature`.
Range: -20 to +20 degrees. Negative = warm (shift toward red/orange),
positive = cool (shift toward blue/cyan). Also adds a small chroma boost to neutrals
via `Math.abs(temperature) * 0.003`.

---

## State Model

All mutable state lives in these JS objects + two scalars:

```js
const hues       = { Red, Yellow, Green, Cyan, Blue, Magenta, Black, White }  // current hue angles
const params     = { Dim: {l, c}, Normal: {l, c}, Bright: {l, c} }            // live L/C for current mode
const paramsStore = {
  dark:  { Dim: {l, c}, Normal: {l, c}, Bright: {l, c} },  // persisted L/C for dark mode
  light: { Dim: {l, c}, Normal: {l, c}, Bright: {l, c} },  // persisted L/C for light mode
}
let temperature = 0
let neutralSat  = 1          // 0–1 multiplier on neutral chroma (Neutral S slider)
const neutralFactor = { Black: 0.42, White: 1.05 }  // both mutated by Black L / White L sliders

const DEFAULTS = { ... }  // frozen reference for "changed" detection
const PRESETS  = { ... }  // named built-in configurations
```

`params` is the working copy for the current mode — all sliders read/write it directly. `paramsStore` tracks both modes independently so export and mode-switching are always faithful. The invariant: whenever `params` is mutated (slider input, preset apply, mode toggle), `paramsStore[currentMode()]` is kept in sync.

`applySwatches()` is the single render function — all controls call it after
mutating state. No reactive framework, no virtual DOM.

`buildZedPalette()` formats the current palette as Zed theme keys (`terminal.ansi.*`), distinct from the plain hex `<pre>` output produced by `renderPalette()`.

`captureCurrentState()` snapshots all live state into a preset-shaped object — used by export and "Save Preset".

---

## Canvas Hex Conversion

**Critical pattern.** `getComputedStyle(el).backgroundColor` returns
`color(srgb 0.003... ...)` for OKLCH colors in modern browsers, not `rgb(...)`.
Parsing this string with regex is broken.

The correct approach is to resolve the color through a 1×1 canvas:

```js
const canvas = document.createElement('canvas');
canvas.width = canvas.height = 1;
const ctx = canvas.getContext('2d');

function toRGB(cssColor) {
  ctx.clearRect(0, 0, 1, 1);
  ctx.fillStyle = cssColor;   // accepts any valid CSS color
  ctx.fillRect(0, 0, 1, 1);
  const d = ctx.getImageData(0, 0, 1, 1).data;
  return [d[0], d[1], d[2]]; // always clamped 8-bit sRGB
}
```

This is used for: hex display, `title` attribute, WCAG contrast computation.

---

## WCAG Contrast

Standard relative luminance formula applied to the resolved sRGB values:

```js
function linearize(c) {
  c /= 255;
  return c <= 0.04045 ? c / 12.92 : ((c + 0.055) / 1.055) ** 2.4;
}
function luminance(r, g, b) {
  return 0.2126 * linearize(r) + 0.7152 * linearize(g) + 0.0722 * linearize(b);
}
function contrastRatio(lum1, lum2) {
  return (Math.max(lum1, lum2) + 0.05) / (Math.min(lum1, lum2) + 0.05);
}
```

Levels: AAA ≥ 7.0 (green dot), AA ≥ 4.5 (yellow), A ≥ 3.0 (orange), fail (red).

---

## Slider "Changed" Highlighting

Output elements next to each slider are highlighted when their value differs
from `DEFAULTS`. The key invariant: **all slider outputs must be `<output>`
elements**, not custom tags like `<o>`.

- CSS targets `.slider-row output.changed`
- `.value` property on `<output>` reflects to displayed text content
- `markOutput(outputEl, currentStr, defaultStr)` toggles the `.changed` class

**Known footgun:** If `<o>` is accidentally used instead of `<output>`,
`output.value = x` silently sets a JS property without updating the DOM,
and the CSS rule never matches. This bug has occurred multiple times.
Always verify with: `grep -c '<output' ansi-palette-generator.html` — expected count is 18.

---

## Preset System

Each preset is a full state snapshot:

```js
{
  hues:         { Black, Red, Yellow, Green, Cyan, Blue, Magenta, White },
  temp:         0,
  neutralSat:   1,
  neutralBlack: 0.42,
  neutralWhite: 1.05,
  params: {
    dark:  { Dim: {l, c}, Normal: {l, c}, Bright: {l, c} },
    light: { Dim: {l, c}, Normal: {l, c}, Bright: {l, c} },
  },
}
```

`applyPreset(preset)` reads `currentMode()`, applies `preset.params[mode]` to
the live `params` state, populates **both** `paramsStore.dark` and `paramsStore.light`
from the preset, calls `syncSlidersToState(preset)` to update all slider
positions and output values/classes, then calls `applySwatches()`.

The mode toggle re-applies the active preset's L/C params for the new mode (if
a preset is selected). If no preset is active, the toggle flushes `params` →
`paramsStore[prev]` and loads `paramsStore[next]` → `params`, so dark and light
modes each remember their own independently tuned L/C values.

Manual slider interaction clears the preset select (`value = ''`).

**Custom preset persistence** uses localStorage key `"custom-presets"` — a flat
JSON object keyed by preset name. Custom preset option values are prefixed with
`"custom:"` (e.g. `"custom:My Preset"`) to distinguish them from built-in keys in
the `<select>`. `getActivePreset()` handles both namespaces.

**Neutral hue angles are preset-specific** — each preset defines Black/White hues
to match its vibe (warm amber for Warm, slate-blue for Cool, etc.) rather than
using the DEFAULTS values.

---

## Rainbow Sort Order

Swatches are ordered: **Black, Red, Yellow, Green, Cyan, Blue, Magenta, White**.

This order is used in both the DOM (swatch grid) and the `<pre>` palette output.
The `RAINBOW` array drives palette rendering:

```js
const RAINBOW = ['Black', 'Red', 'Yellow', 'Green', 'Cyan', 'Blue', 'Magenta', 'White'];
```

---

## Dark / Light Mode

Each preset stores separate `dark` and `light` L/C params. Toggling mode
re-applies the correct set from the active preset (or from `paramsStore` when
no preset is active). Light mode uses lower L values (~0.42–0.60) so colors
read well against `#f5f5f5`.

---

## Hue Slider Ranges

Per-color hue sliders are constrained to keep each color recognizable.
Black and White use the full range since any tint direction is valid for a neutral.

| Color   | Min | Max | Rationale                    |
|---------|-----|-----|------------------------------|
| Black   | 0   | 360 | Full range — neutral tint    |
| Red     | 10  | 60  | Red to orange                |
| Yellow  | 75  | 130 | Orange-yellow to lime        |
| Green   | 120 | 175 | Lime to teal-green           |
| Cyan    | 175 | 220 | Green-cyan to sky            |
| Blue    | 220 | 295 | Sky to blue-violet           |
| Magenta | 295 | 360 | Blue-violet to red-magenta   |
| White   | 0   | 360 | Full range — neutral tint    |

---

## Browser Compatibility

OKLCH in CSS requires: Chrome 111+, Firefox 113+, Safari 15.4+.
`ctx.fillStyle` accepting OKLCH requires the same versions (canvas resolves
via the same CSS color parsing pipeline).
