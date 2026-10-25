# CLAUDE.md

Guidance for AI coding agents (and humans) working on Homelab Sanctuary.

## What this is

A one-file canvas app (`index.html`) that draws a virtual office. Hosts are rooms and services are workers. It polls Prometheus `probe_success` and shows up/down state. There is no build, bundler or test runner.

## Data contract (do not break)

- Query: `/api/v1/query?query=probe_success` (same origin).
- Each result needs labels `host` and `app` and a value of `1` (up) or anything else (down).
- State lives in `liveServiceState["host:app"]` (boolean), seeded from `staticHostSchema`.
- A failed fetch must not crash the page; it falls back to the current state.

## Code map (all in the single `<script>`)

1. `staticHostSchema`: hosts and services (the source of truth for rooms and workers).
2. Art kit: `hash`, `rng`, `rr` (rounded rect helper), `mk` (builds repeating textures), `T` (Ghibli textures), `THEME` and `JP` (per-host floor and Japanese title).
3. Scene system: `TX` (extra textures), `SC` (scene configs), `S` (active scene), `roomLvl`/`wl` (health to weather), `skyFx`, `fx`, `setScene`.
4. `drawRoom`, `torso`, `head`, `portrait`, `drawWorker`, `fit`: the drawing code.
5. Camera, `calculateLayout` (rooms and worker positions), the hover popup, `animate`, then Prometheus sync and the header UI.

## Conventions and gotchas

- Everything is drawn on one canvas in world coordinates around (0,0). The camera is applied in `animate`.
- Appearance must be deterministic: derive it from `hash(host + app)`. Never use `Math.random()` in layout or drawing.
- Use `>>>` (unsigned shift) on hash values. Signed `>>` produced negative indexes and "undefined" names before.
- Colors that differ per scene must come from `S`. Do not hard-code scene colors in `drawRoom` or `drawWorker`.
- Keep the worker anatomy consistent: the worker faces the viewer, the monitors face the worker (the viewer sees their backs), and the keyboard sits on the worker's side.
- Do not use `localStorage` for state that matters. It is only used to remember the chosen scene.

## Adding a scene

Add an entry to `SC`:

| Field | Purpose |
|---|---|
| `n` | Label in the selector |
| `tex` | Key in `TX` for the floor (omit to use the per-host Ghibli floors) |
| `wall`, `wain` | Wall and lower wall colors (omit for Ghibli textures) |
| `amb` | Warm light color (rgba) |
| `sky` | 3 gradient colors for the window |
| `plq` | Nameplate fill, border, title color, subtitle color |
| `desk` | Desk top light, top dark, front/legs, edge highlight |
| `chair`, `mon` | Chair and monitor casing colors |
| `suit` | Optional array of 5 outfit colors |
| `fx` | Particles: `rain`, `stars`, `bub`, `fire`, `fly` |
| `city`, `planet`, `fish`, `nc` | Window extras (`nc` = no clouds) |
| `nf` | `1` hides the office floor furniture |
| `dec(ctx, x, y, t, i)` | Optional wall decoration drawn at the right of the wall |
| `rain` | Color of rain or meteor streaks in the window |
| `wn`, `wi` | 4 weather names (clear to storm) and optional 4 header icons |
| `kind`, `map`, `list` | `kind: 'animal'` or `'crop'` replaces the office worker with a creature or plant (`animal`, `crop`, `kindSprite`). `map` is host to species, `list` the fallback by name hash |
| `kind: 'cast'`, `chars`, `dn`, `stn` | Themed characters: `chars` lists keys of the `CH` registry (each `CH.x(ctx, P)` draws one character; `P = {up, blink, t, ph}`), `dn` picks the down effect (`glitch`, `float`, `curse`, `net`), `stn(ctx, up, t)` draws the props on the desk (`cast`) |
| `nw` | `1` replaces the window with an open sky band across the wall (`skyBand`) |
| `fn`, `ln` | Optional name lists used for the hover card |
| `school` | `1` swaps monitors for open books, centres the grid, adds extra bottom space and a teacher at bottom centre, drawn from behind and facing the class (`drawTeacher`) whose mood follows `roomLvl(host)` |
| `jp` | Optional map of nameplate subtitles per host (the High School scene uses class names) |
| `px` | `1` renders at half resolution for a pixel-art look |

To add a floor texture, add `[width, height, (ctx, rng) => {...}]` to `TX`.

## Dedication modal

`#ded-btn` (header) opens `#ded-modal`. Its content is filled from the `DEDICATION` constant with `textContent`, so it is safe for plain text. Escape, the backdrop and Close dismiss it.

## Testing

There is no automated test setup. A quick smoke test runs the script in Node with a stub 2D context that accepts any method call, calls `setScene(k)` for every key in `SC`, runs `animate()` a few times, then repeats with some `liveServiceState` entries set to `false`. Check that nothing throws, then look at the page in a browser, since stubs cannot show visual problems.

## Style

Plain JavaScript, no dependencies beyond the Tailwind CDN. Keep the file a single self-contained `index.html`.
