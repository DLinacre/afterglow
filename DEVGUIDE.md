# AFTERGLOW — Developer Guide

A single-file HTML5 game. Open `afterglow.html` in any modern browser (desktop or
mobile). No build step, no server, no dependencies. Everything lives in that one file.

The whole thing is written so you can add content by dropping **one entry into a
registry**. You almost never need to touch the game loop.

---

## 1. File layout (search for these banner comments in the `<script>`)

| Banner | What's there |
|---|---|
| `CONFIG` | All tunable numbers (guide size, scoring generosity, coach behaviour). |
| `GLOW SPRITE ATLAS` | Pre-rendered radial glow sprites + `stamp()` / `stampLine()`. |
| `PALETTES` | Colour schemes. |
| `BRUSHES` | Brush behaviours (how many "pens" and where they go). |
| `SHAPES` | Curated onboarding shapes. |
| `PROC_GENERATORS` | Endless procedural shape makers. |
| `TIPS` | Rotating "learn to draw" advice lines. |
| `STATE` | The single `S` object holding all runtime state. |
| `MAIN LOOP` | `frame()` — draws bg, guide, coach, and paints. |
| `SCORING` | `developPhoto()` + `showResult()` (coverage vs. spill). |

---

## 2. Add a new PALETTE (colours)

```js
{ name: 'Sunset', lo: 350, hi: 40, speed: 1.0, cycle: false },
```
- `lo`/`hi` = hue range in degrees. `cycle:true` sweeps the whole range continuously
  (good for rainbow); `cycle:false` oscillates gently between `lo` and `hi`.
- `speed` = how fast the colour moves. Done — it appears in the ⚙ panel automatically.

## 3. Add a new BRUSH

A brush is `{ id, name, cn, connected, make }`.
- `connected: true` → consecutive pen positions are joined with a smooth line
  (draws continuous trails). `false` → each frame stamps loose dabs (e.g. Sparkler).
- `make()` returns an object with `reset()` and `pens(s)`.
- `pens(s)` returns an array of `{x, y, size, hueDeg}` — one per "pen".
  **Keep the array length stable while the finger is down** if `connected`, so lines
  join correctly.
- `s` gives you: `{x, y, dt, t, phase, pal, dim, spd}` where `x,y` = finger position,
  `dim = min(width,height)`, `pal` = current palette (use `paletteHue(pal, phase)`),
  and `spd` = smoothed pointer speed 0..1 (use it for pressure-style width, see the
  **Ink** brush). Final pen sizes are automatically multiplied by the user's brush-size
  setting, so just return your natural sizes.

Example — a two-pen "wings" brush:
```js
{ id:'wings', name:'Wings', cn:'twin trails', connected:true, make:()=>({
    reset(){},
    pens(s){ const o=s.dim*0.04; const h=paletteHue(s.pal,s.phase);
      return [ {x:s.x-o,y:s.y,size:s.dim*0.06,hueDeg:h},
               {x:s.x+o,y:s.y,size:s.dim*0.06,hueDeg:h} ]; }
  })},
```

## 4. Add a SHAPE

**Curated** (shown early, in order) — push to `SHAPES`:
```js
{ name:'Wave', gen: () => { const p=[]; for(let x=-0.4;x<=0.4;x+=0.02)
    p.push({x, y:Math.sin(x*10)*0.2}); return [p]; } },
```
A `gen()` returns **strokes**: an array of point-arrays, each point `{x,y}` in
normalized units (roughly -0.5..0.5, centred on 0). Multiple strokes = pen lifts.

**Procedural** (endless) — push a function to `PROC_GENERATORS`:
```js
(rng) => { const p=[]; for(let t=0;t<=6.34;t+=0.02)
    p.push({x:Math.cos(t)*0.4, y:Math.sin(t*3)*0.4});
    return norm({ name:'Curl', strokes:[p] }); },
```
- Use the passed seeded `rng()` (0..1) and `rint(rng,a,b)` for repeatable randomness.
- Wrap odd-sized shapes in `norm(obj)` to auto-fit them to the frame.
- Photos ≤ `CONFIG.curatedLevels` use `SHAPES`; after that a generator is picked
  deterministically per photo number (photo #N is always identical).

## 5. Add a TIP (teaching)

Just add a string to `TIPS`. Tips appear in post-photo feedback and can be surfaced
anywhere via `pickTip()`.

---

## 6. Tuning gameplay (`CONFIG`)

| Key | Effect |
|---|---|
| `guideBoxFrac` | Size of the guide relative to the screen. |
| `scoreCell` | Scoring grid resolution (smaller = stricter/slower). |
| `curatedLevels` | How many hand-picked photos before procedural takes over. |
| `litThreshold` | Brightness needed to count a pixel as "painted". |
| `coverageGain` | Higher = more forgiving on filling the shape. |
| `spillPenalty` | Higher = punishes painting outside the shape more. |
| `autoCoachLevels` | Photos where the trace coach auto-plays. |

---

## 7. Modes

- **Guided** (`Start Shooting`): endless photos, guide + coach + scoring.
- **Free Studio**: no target — pure art. Pick brush/palette in ⚙, `Save Photo`
  exports a PNG (background + light trails, watermarked).

## 7b. Undo / Redo

The paint layer uses **snapshot-based history**. Before each stroke begins (`down()`),
`pushHistory()` saves the current paint canvas as `ImageData`. `undo()` / `redo()` swap
between the `S.history` and `S.future` stacks (max 20 steps). Any action that changes the
canvas non-interactively (Clear) also calls `pushHistory()` first. History is reset on
level load and on resize (snapshots would be the wrong pixel size otherwise). If you add
a new canvas-mutating action, call `pushHistory()` before it and you're done.

## 8. Ideas parked for later
- Ambient audio that responds to painting speed.
- "No-go" dark zones for a tension mode (mask already exists — just penalise entry live).
- Daily challenge (seed the RNG with the date → everyone gets the same photo).
- Share sheet on mobile via the Web Share API using the saved PNG blob.
- Unlockable brushes/palettes tied to star totals.

Each of these slots into an existing registry or the `frame()` loop without a rewrite.
