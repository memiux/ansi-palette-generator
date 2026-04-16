# ANSI Color Palette Tool — Claude Code Project

## Overview

A self-contained, single-file HTML tool for designing 16-color ANSI terminal
palettes in pastel style. The primary use case is generating color values for
a **Zed editor theme** (`kizuna-ai.json`), but the output is generic enough
for any terminal emulator or editor theme format.

**File:** `ansi-colors.html` — zero dependencies, no build step.

---

## What the Tool Does

- Displays 24 swatches: 3 rows (Dim / Normal / Bright) × 8 colors
- Colors are generated algorithmically from OKLCH parameters, not hardcoded
- Sliders for per-group L and C, per-color hue angles, and a global temperature offset
- Dark/Light mode toggle
- Preset selector (Default, Warm, Cool, Nord-ish, Muted, Vivid)
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

**Default L/C per group:**

| Group  | L    | C    |
|--------|------|------|
| Dim    | 0.55 | 0.07 |
| Normal | 0.72 | 0.10 |
| Bright | 0.82 | 0.13 |

### Neutrals (Black / White)

Black and White have no hue (C=0). Their lightness is derived from the group's
L value via fixed factors:

```js
neutralFactor = { Black: 0.42, White: 1.05 }
neutralL = clamp(groupL * factor, 0.05, 0.98)
```

When temperature ≠ 0, neutrals receive a tiny chroma tint:
- `C = abs(temperature) * 0.003`
- Warm (temp < 0): H = 60° (yellow)
- Cool (temp > 0): H = 260° (blue)

### Temperature

A global hue offset applied to all chromatic colors: `hues[color] + temperature`.
Range: -20 to +20 degrees. Negative = warm (shift toward red/orange),
positive = cool (shift toward blue/cyan).

---

## State Model

All mutable state lives in three JS objects + one scalar:

```js
const hues   = { Red, Yellow, Green, Cyan, Blue, Magenta }  // current hue angles
const params = { Dim: {l, c}, Normal: {l, c}, Bright: {l, c} }
let temperature = 0

const DEFAULTS = { ... }  // frozen reference for "changed" detection
const PRESETS  = { ... }  // named configurations
```

`applySwatches()` is the single render function — all controls call it after
mutating state. No reactive framework, no virtual DOM.

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
Always verify with: `grep -c '<output' ansi-colors.html` — expected count is 13.

---

## Preset System

Each preset is a full state snapshot:

```js
{
  hues:   { Red, Yellow, Green, Cyan, Blue, Magenta },
  temp:   0,
  params: { Dim: {l, c}, Normal: {l, c}, Bright: {l, c} },
}
```

`applyPreset(preset)` mutates state, calls `syncSlidersToState(preset)` to
update all slider positions and output values/classes, then calls `applySwatches()`.

Manual slider interaction clears the preset select (`value = ''`).

---

## Rainbow Sort Order

Swatches are ordered: **Black, Red, Yellow, Green, Cyan, Blue, Magenta, White**.

This order is used in both the DOM (swatch grid) and the `<pre>` palette output.
The `RAINBOW` array drives palette rendering:

```js
const RAINBOW = ['Red', 'Yellow', 'Green', 'Cyan', 'Blue', 'Magenta', 'Black', 'White'];
```

Note: the grid DOM order differs (Black first) from RAINBOW (Red first).
This is intentional — the grid puts neutrals on the edges visually.

---

## Dark / Light Mode

Two separate L/C value sets are recommended for proper contrast in both modes.
Currently the tool uses a single set of slider values regardless of mode.
This is a known limitation — the same L=0.72 looks fine on dark but may
wash out on light backgrounds.

A future improvement would be storing separate L/C pairs per mode in each
preset, and applying the correct set on toggle.

---

## Hue Slider Ranges

Per-color hue sliders are constrained to keep each color recognizable:

| Color   | Min | Max | Rationale                    |
|---------|-----|-----|------------------------------|
| Red     | 10  | 60  | Red to orange                |
| Yellow  | 75  | 130 | Orange-yellow to lime        |
| Green   | 120 | 175 | Lime to teal-green           |
| Cyan    | 175 | 220 | Green-cyan to sky            |
| Blue    | 220 | 295 | Sky to blue-violet           |
| Magenta | 295 | 360 | Blue-violet to red-magenta   |

---

## Browser Compatibility

OKLCH in CSS requires: Chrome 111+, Firefox 113+, Safari 15.4+.
`ctx.fillStyle` accepting OKLCH requires the same versions (canvas resolves
via the same CSS color parsing pipeline).

---

## Planned / Discussed Improvements

These were discussed but not yet implemented:

1. **URL hash state** — encode all slider values into `location.hash` on change.
   Enables bookmarking and sharing a specific palette configuration.

2. **Export button** — copy current palette as:
   - CSS custom properties: `--color-red: #C47A72;`
   - Zed JSON snippet for `kizuna-ai.json`

3. **Per-mode L/C in presets** — separate dark/light values so the same preset
   looks correct in both modes.

4. **Preset save slots** — `localStorage`-backed named slots for user-defined
   configurations (device-local only).

---

## Zed Theme Integration

The end goal is populating `kizuna-ai.json`. Typical Zed theme palette keys:

```json
{
  "players": [{ "cursor": "#FF...", ... }],
  "editor.background": "#...",
  "terminal.ansi.red": "#...",
  "terminal.ansi.bright_red": "#...",
  ...
}
```

Map: Dim row → no direct ANSI equivalent (can be used for UI chrome),
Normal row → `terminal.ansi.*`, Bright row → `terminal.ansi.bright_*`.

---

## Code Style Notes

- Tabs (size 2), no spaces
- No superfluous comments
- Descriptive variable names, no single-letter variables outside loops
- Exception-based error philosophy (no defensive null checks in happy path)
- Keep it a single file — no build tooling, no external dependencies
