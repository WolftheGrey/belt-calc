# Belt Drive Planner

Interactive 2D belt-drive calculator and visual planner — for designing layouts
of motorized pan/tilt heads and similar small geared belt drives.

A single HTML file. No build, no backend, no installation — just open
`index.html` in a browser.

**🌐 Live: https://wolfthegrey.github.io/belt-calc/**

## What it does

A browser tool where you "sketch" a belt layout (1- or 2-stage), drag shafts
and the tensioner with the mouse, and immediately see:

- belt length and how close it is to the nearest standard
- wrap angle on the smaller pulley and number of teeth in mesh
- ratio of each stage and total reduction
- output angular resolution accounting for motor microstepping
- assembly bounding box

The style is engineering blueprint: dark-blue background, monospace font,
millimeter grid.

## Running

```bash
# just open in a browser
open index.html
```

Or via HTTP:

```bash
python3 -m http.server 8000
# then http://localhost:8000
```

React and htm are loaded as ES modules from [esm.sh](https://esm.sh) on first
load.

## Features

### Topology and profiles

- **1 or 2 reduction stages**
- Belt profiles: **GT2-2M, GT3-2M, HTD-3M, HTD-5M** — with full built-in
  tables of standard closed-loop belt lengths
- Pitch diameter is `D = T × pitch / π`; the belt is rendered on the pitch
  circle (visually just inside the thin pulley outline)

### Scene controls

| Gesture | Action |
|---|---|
| `Drag` element | move (Shift = single axis) |
| `Click` | select — Blender-style XY gizmo appears |
| `Drag` red/green arrow | move strictly along that axis |
| `←↑→↓` | nudge 1 mm (Shift ×10) |
| `Double-click` | X/Y modal + snap-to-standard |
| `Wheel` | smooth zoom around cursor |
| `+` / `−` / `0` (or Cmd/Ctrl variants) | zoom buttons |
| `Drag` RMB / empty canvas / `Space`+drag | pan |

### Belt length

- **Badge near each belt** — standard length in large text, actual length +
  delta below
- **Double-click the badge** → modal for direct L editing; driven shaft
  repositions numerically
- **Double-click the C value** on the dimension line → modal for editing
  the center distance
- **🔒 Lock** in the length modal: fixes belt length. The driven shaft
  auto-adjusts whenever teeth or tensioner change so L stays equal to
  `lockL`. When a locked L can't be achieved — input and badge highlighted
  red.

### Tensioner

- Toggled with a checkbox, spawned **inside** the belt loop (for INT) or
  **outside** (for EXT)
- **INT (auto)** — convex hull: the belt wraps the tensioner with its
  toothed side as a 3rd pulley
- **EXT (inverted)** — internal-tangent geometry: tensioner outside the
  loop, the belt's smooth back presses it inward
- Switching mode in the dropdown — the tensioner jumps across the belt
  tangent, preserving side
- **EXT tensioner detaches** automatically when its circle is fully clear
  of the natural 2-pulley belt boundary (no contact → nothing to press
  against). **INT tensioner** remains part of the convex hull at any
  non-collinear position.
- **"Reset tension"** button moves the tensioner back to a neutral
  position where the belt becomes the natural 2-pulley loop.

### Export and persistence

- **Copy URL** — compact base64 serialization of the full state in the URL
  (for sharing configurations)
- **Copy JSON** — the same as JSON
- **Export SVG** — vector schematic for documentation/CAD
- **Export PNG** — raster schematic + a side table with element
  coordinates, pulley diameters, belt lengths, ratio, and angular
  resolution. The viewport auto-fits the content with 25 mm padding while
  preserving canvas aspect ratio.
- **Auto-save** — every state change is mirrored to the URL and
  localStorage. Opening a saved URL restores the configuration; a clean
  URL — the last local session.

## Architecture

A single `index.html`, ~3300 lines. Inside `<script type="module">`:

| Layer | Contents |
|---|---|
| **Constants** | belt profiles, default state, scene size, ZOOM_STEP, SNAP_STEP |
| **Geometry helpers** | `pitchDiameter`, `openBeltLength`, `wrapAngleSmall`, `nearestStandard`, `dist` |
| **Tangent calculations** | `externalTangentAngles` / `internalTangentAngles` (with clamping for numerical stability) |
| **Belt path builders** | `buildBelt` — general convex hull for N circles; `buildBeltInverted` — EXT tensioner via internal tangents |
| **PNG export** | `exportPng` + `drawExportTable` |
| **Tensioner positioning** | `tensionerPosForSide` — spawn and mode-switch jump |
| **Lock enforcer** | `applyLockForStage` + `enforceLocks` — bisection over C to maintain a fixed belt length |
| **State** | URL ⇄ localStorage, `mergeDefaults` |
| **React components** | via [htm](https://github.com/developit/htm) (template-literal JSX-like, no Babel) |

### React tree

```
App
├── Toolbar (Reset / Copy URL / Copy JSON / Export SVG/PNG / Snap)
└── Layout
    ├── ParameterPanel       — left column (topology, stages, motor)
    ├── Scene                — central SVG canvas
    │   ├── Rulers
    │   ├── BeltLayer        — belt straights and arcs
    │   ├── Dimensions       — dashed C-lines with labels
    │   ├── BeltBadgeLayer   — large belt-length badges
    │   ├── PulleyLayer      — Motor / Intermediate / Output shafts
    │   ├── TensionerLayer   — tensioners
    │   ├── ArrowWidget      — XY gizmo for the selected element
    │   └── ZoomCtl          — +/⊡/− buttons
    ├── ResultsPanel         — right column (calculations)
    ├── EditModal            — element coordinates
    ├── DistanceModal        — center distance C
    └── BeltLengthModal      — belt length L + lock
```

### Key invariants

- **Pulleys at outer diameter, belt at pitch diameter** — the belt renders
  as a distinct curve just inside the pulley outline (physically correct,
  since teeth engage at pitch, and it doesn't merge visually with the
  pulley ring).
- **SVG Y-down throughout**. `atan2/cos/sin` follow the same convention;
  "math CCW" = "visual CW"; the SVG arc sweep flag is set per arc
  direction.
- **Belt builders never return null** for an otherwise-renderable input —
  there's a degenerate fallback (line through circle centers) so transient
  bad configurations don't make the whole belt vanish.
- **Numerical stability**: arguments to `Math.acos` are clamped to
  `[-1, 1]`; `pickDir` uses a cross-product tiebreaker for the symmetric
  2-circle case.

## Stack

- **React 18** + **htm 3** via [esm.sh](https://esm.sh) — no build step
- SVG for all graphics, Canvas only for PNG export
- `localStorage` + `URLSearchParams` for persistence

## Browser support

Targets modern browsers with ES module support (released around 2017+):

- Chrome / Edge 79+
- Firefox 60+
- Safari 13+ (iOS Safari 13+)
- Android Chrome (current)

On narrow viewports (< 880 px) a banner appears pointing to Chrome's
"Desktop site" mode, which gives a workable ~980 px viewport. Tablets in
landscape work natively.

## Context

Built for designing a pan/tilt camera head, so the feature set targets
small stepper drives and fine-pitch toothed belts (GT2 / GT3 / HTD-3M /
HTD-5M). For V-belts, round belts or chains — different formulas would
be needed.

## License

MIT — see [LICENSE](LICENSE).

## Credits

- Concept & direction: Stanislav Volkov
- Implementation: Claude Code · Anthropic
