# CLAUDE.md

Project-specific instructions for Claude Code and other AI coding agents working on **Homelab Sanctuary**. Read this file before editing the app. Treat the existing implementation as the source of truth; when this guide and the code disagree, inspect the code and update this guide only when the architecture has intentionally changed.

## Project at a glance

Homelab Sanctuary is a small, self-contained canvas dashboard that turns homelab monitoring into a themed virtual world. Hosts are represented as rooms or departments; monitored applications are represented as workers, animals, plants, or themed characters depending on the selected scene. Prometheus provides live availability.

- Main application: `index.html`.
- Rendering: native Canvas 2D API and plain JavaScript.
- Styling: existing CSS and the Tailwind CDN already referenced by the page.
- Monitoring: same-origin Prometheus HTTP API, normally exposed through an Nginx reverse proxy.
- Build tooling: no bundler, package manager, framework, or formal test runner is currently required.
- Deployment: static files served by a web server/container; Prometheus remains a separate service.

Keep the app lightweight and easy to host. Do not introduce a framework, build system, dependency, or extra service for a small change unless the user explicitly requests it or there is a clear, explained need.

## Working rules

1. **Inspect before editing.** Read the relevant sections of `index.html` and trace callers, state, and rendering effects before changing code. The application is intentionally concentrated in one file.
2. **Make the smallest coherent change.** Prefer focused edits over broad rewrites, reorganizing the file, renaming many functions, or changing unrelated behavior.
3. **Preserve existing features.** Keep all scenes, host/service data, scene selection, responsive fitting, hover details, animations, weather effects, and the dedication modal working unless the task explicitly changes one of them.
4. **Do not invent project structure.** Verify paths, functions, scene names, configuration keys, and selectors in the repository before referring to or editing them.
5. **Do not silently change the monitoring contract.** If a change requires a new metric, label, endpoint, proxy route, or deployment setting, explain the dependency and update the relevant documentation.
6. **Avoid new dependencies.** Reuse existing browser APIs and helpers. Do not add a build step or dependency just for convenience.
7. **Keep code readable.** Use descriptive names, small helpers where useful, and comments for non-obvious rendering math or compatibility constraints. Avoid unnecessary abstractions.
8. **Never claim a test passed unless you ran it.** If execution is unavailable, say what was inspected and what still needs browser verification.
9. **Do not overwrite user work.** Check the current Git diff/status when working in a local checkout. In GitHub edits, fetch the latest file and use its current SHA before updating it.
10. **Report the outcome clearly.** Summarize changed behavior, files touched, tests actually run, and any remaining manual checks.

## Architecture and data flow

The intended monitoring path is:

```text
Monitored service/host
        ↓
Blackbox Exporter probe
        ↓
Prometheus scrape and time series
        ↓
Nginx same-origin reverse proxy
        ↓
Browser query: /api/v1/query?query=probe_success
        ↓
Parse labels host + app and sample value
        ↓
liveServiceState["host:app"]
        ↓
Worker/creature/plant appearance and room health/weather
```

The browser must remain usable if Prometheus is temporarily unreachable. Before the first successful poll, the registry is empty and the header should show a waiting state. After discovery, the canvas should continue rendering from the last known state; a failed poll must not terminate animation or break scene selection.

### Prometheus data contract — preserve this

- Query the instant-query endpoint at `/api/v1/query?query=probe_success` using the same origin.
- Each relevant result is expected to have `metric.host` and `metric.app` labels.
- The sample value is a string in `value[1]`; `"1"` means UP and a non-`"1"` value means DOWN.
- Runtime state is stored under `liveServiceState["host:app"]` as a boolean.
- `hostAppSchema` starts empty and is populated dynamically from valid Prometheus results. Each distinct `host` creates a room; each distinct `app` under that host creates a worker.
- The data parser must handle missing, malformed, or unexpected results defensively. Do not let a failed request or malformed response throw out of the polling loop.
- Keep the UI's fallback behavior intact when a request fails. Do not mark every service down merely because Prometheus is temporarily unavailable unless the product behavior is deliberately redesigned.
- Do not assume that a service being down will automatically create a time series. Check Prometheus targets, scrape configuration, labels, query results, and proxy routing before changing drawing code.

When debugging a worker that remains green after its service is stopped, follow the data path in order:

1. Confirm the target is actually being probed and the probe result is failing.
2. In Prometheus, run `probe_success` and inspect the exact `host` and `app` labels and sample value.
3. Confirm the dashboard's expected host/app pair matches those labels exactly.
4. Confirm the browser can reach `/api/v1/query?query=probe_success` through the same-origin proxy.
5. Confirm the response parser updates `liveServiceState["host:app"]` for that exact pair.
6. Confirm the selected scene uses the health state when drawing the relevant worker.

Do not fix a stale-green display by arbitrarily changing colors or hard-coding an app as down; repair the failing link in the monitoring chain.

## Code map

The core implementation lives in the main `<script>` in `index.html`. Use the current file to verify exact definitions and ordering.

- **`hostAppSchema`** — runtime registry of hosts and apps discovered from Prometheus.
- **Art and deterministic helpers** — `hash`, `rng`, `rr` (rounded rectangles), `mk` (repeating textures), and related drawing utilities.
- **Base art and host themes** — `T`, deterministic host-floor selection, and scene-level visual choices.
- **Scene textures/configuration** — `TX` contains extra floor textures; `SC` contains scene configuration; `S` is the active scene.
- **Health and atmosphere** — `roomLvl`, `wl`, `skyFx`, and `fx` connect room health to weather and scene effects.
- **Scene selection** — `setScene` changes the active visual configuration.
- **Drawing** — `drawRoom`, `torso`, `head`, `portrait`, `drawWorker`, and `fit`, plus the scene-specific drawing helpers.
- **Layout and camera** — `calculateLayout`, camera transforms, pointer/hover logic, and `animate`.
- **Monitoring and UI** — Prometheus synchronization, header controls, scene selector, and dedication modal.

The app is a custom canvas renderer, not a 3D engine. Keep descriptions accurate: the scenes create a stylized illustrated world using 2D drawing.

## Rendering and layout invariants

- Draw in canvas world coordinates; the camera transform is applied in the animation/render loop. Do not mix screen coordinates and world coordinates without converting them correctly.
- Preserve the existing coordinate and scaling conventions. Use the established fit/camera helpers rather than adding competing resize logic.
- Layout and appearance must be deterministic for a given host/app and scene. Derive stable variation from `hash(host + app)` and existing seeded RNG helpers.
- **Never use `Math.random()` for persistent worker identity, room layout, or drawing choices.** Randomized visuals that change every frame cause flicker and break identity consistency.
- Use unsigned bit shifts (`>>>`) when converting hash values to indexes. Signed `>>` has previously caused negative indexes and undefined names.
- Keep scene-specific colors and behavior in `S` / `SC`, rather than hard-coding a scene check or palette deep inside generic drawing functions.
- Keep drawing helpers safe to call repeatedly during animation. Avoid mutating shared configuration objects or accumulating state on every frame.
- Be mindful of canvas performance: reuse existing helpers, avoid expensive per-frame work where possible, and do not add unbounded arrays or timers.
- Maintain readable silhouettes and labels at different viewport sizes. Verify both fullscreen and narrow/side-by-side use when changing sizing, text, or layout.
- Preserve the established worker anatomy: workers face the viewer; monitors face the worker so the viewer sees the backs; keyboards sit on the worker's side. Do not reverse the desk orientation accidentally.
- Keep pointer hit-testing aligned with the camera transform and the positions actually drawn.
- Do not use `localStorage` for application state that matters. It is currently used to remember the selected scene; keep persisted state minimal and non-sensitive.

## Scene system

Scenes are configured in `SC`. Prefer adding configuration over branching throughout the renderer. Inspect existing scene entries and shared helpers before adding another scene.

Common scene configuration fields include:

| Field | Meaning |
|---|---|
| `n` | Human-readable scene label |
| `tex` | Key in `TX` for the floor texture; omit to use the existing per-host Ghibli floors |
| `wall`, `wain` | Wall and lower-wall colors |
| `amb` | Ambient/warm light color |
| `sky` | Three colors for the window gradient |
| `plq` | Nameplate fill, border, title, and subtitle colors |
| `desk` | Desk top light/dark, front/legs, and edge highlight |
| `chair`, `mon` | Chair and monitor casing colors |
| `suit` | Optional array of five outfit colors |
| `fx` | Scene particles such as `rain`, `stars`, `bub`, `fire`, or `fly` |
| `city`, `planet`, `fish`, `nc` | Window details; `nc` disables clouds |
| `nf` | Set to `1` to hide office floor furniture |
| `dec(ctx, x, y, t, i)` | Optional wall decoration |
| `rain` | Color used for rain or meteor streaks |
| `wn`, `wi` | Four weather names and optional four header icons |
| `kind`, `map`, `list` | Select animal/crop-style workers and map hosts to species; fallback list selection is hash-based |
| `kind: 'cast'`, `chars`, `dn`, `stn` | Themed character registry keys, down-state effect, and desk props |
| `nw` | Set to `1` to use an open sky band across the wall |
| `fn`, `ln` | Optional first/last-name lists for hover cards |
| `school` | Enables the school layout with books, centered grid, extra bottom space, and teacher |
| `jp` | Optional per-host nameplate subtitle overrides, used for class names in the High School scene |
| `px` | Set to `1` for half-resolution pixel-art rendering |

The `hostAppSchema` registry is populated at runtime from Prometheus; this table documents scene options, not a static inventory. Check how each field is consumed in code before modifying it. When adding a field, wire it through the renderer and provide a sensible default for scenes that omit it.

### Add a scene

1. Inspect similar scene entries and the drawing helpers they use.
2. Add a uniquely named configuration to `SC`, including the required palette fields.
3. Add a `TX` floor texture only if the scene needs one. Texture entries follow the existing `[width, height, (ctx, rng) => {...}]` convention.
4. Reuse existing weather, window, character, and layout helpers where possible.
5. Confirm scene selection, animation, hover details, responsive fitting, and health effects still work.
6. Run the smoke test and visually inspect the new scene in a browser.

### Scene-specific systems

- **Animal/crop scenes:** follow the existing `kind`, `map`, `list`, and `kindSprite` conventions. Keep species/plant selection stable for each host/app.
- **Themed character scenes:** use the existing `CH` registry. Character draw functions receive `P = {up, blink, t, ph}`; preserve the meanings of those fields. Keep down-state effects and desk props compatible with the current scene config.
- **High School:** preserve the book-based desk treatment, centered room grid, extra bottom space, teacher drawn from behind facing the class, and teacher mood driven by `roomLvl(host)`. Use `jp` only according to the current nameplate implementation.
- **Pixel-art scene:** preserve the existing half-resolution rendering path; do not approximate it by applying a CSS filter to every scene.
- **Health/weather:** preserve the current mapping from room health to weather level and its scene-specific labels/icons. If changing thresholds or aggregation, inspect the current implementation and document the behavior.

## Host and service data

When adding or renaming a host or app:

1. Configure Prometheus to emit `probe_success` with non-empty `host` and `app` labels for the target.
2. Keep each label stable and unique enough to identify the intended room and worker; capitalization and punctuation are significant.
3. Do not add the host or app to a hard-coded frontend list. Runtime discovery should register it automatically.
4. Confirm the new room/worker appears after a successful poll and layout recalculation.
5. Verify UP and DOWN states and the hover details.

Do not put credentials, API tokens, private endpoints, or secrets in `index.html`, README screenshots, or committed configuration. Monitoring labels should be descriptive, not secret.

## Interaction, accessibility, and modal behavior

- Preserve the existing scene selector, hover popup, camera/pointer interaction, and header controls.
- Ensure changes to mouse/touch input remain compatible with the canvas scaling and camera.
- Keep controls usable on smaller screens and avoid obscuring important status information.
- The header's `#ded-btn` opens `#ded-modal`. The modal content comes from the `DEDICATION` constant and is inserted with `textContent`; keep plain text insertion safe. Escape, the backdrop, and the Close control dismiss it.
- Do not replace safe `textContent` insertion with `innerHTML` for user-controlled or externally sourced text.
- When adding interactive controls, provide clear labels and preserve keyboard behavior where feasible.

## Nginx and deployment assumptions

- The static app is served by a web server/container, commonly Nginx.
- The browser queries Prometheus through a same-origin path beginning `/api/v1/`; Nginx is expected to proxy this path to the Prometheus service (typically `prometheus:9090` on the shared Docker network).
- A typical homelab deployment may publish Nginx port 80 to a LAN port such as `86`, but do not hard-code a specific host IP or port into browser code.
- The browser should not connect directly to an internal Docker hostname. Such hostnames resolve inside Docker networks, not necessarily on a user's device.
- If modifying deployment files, inspect the actual Compose/Nginx configuration and network names first. Do not assume that a Docker network is explicitly named in the Compose file.
- Keep Prometheus and exporter credentials/configuration outside the public static frontend. Do not expose administrative endpoints unnecessarily.

## Testing and verification

There is no formal automated test runner in the current project. Use the strongest practical verification available.

### Smoke test

A lightweight Node-based smoke test can run the inline script with a stub canvas 2D context that accepts drawing method calls. It should:

1. Load/extract the script without changing the committed file.
2. Initialize the required browser globals with minimal stubs.
3. Call `setScene(k)` for each key in `SC`.
4. Run `animate()` for a few frames.
5. Repeat with selected `liveServiceState` entries set to `false`.
6. Check that no exception is thrown.

This catches basic runtime errors, but a stub context cannot detect visual defects, incorrect hit targets, unreadable text, or real browser API differences. Do not report it as a complete visual test.

### Browser smoke test

When a browser is available, verify:

- The page loads without console errors.
- Every scene can be selected and rendered.
- The canvas resizes and remains usable at fullscreen and narrow widths.
- The dedication modal opens and closes using its button, backdrop, Close control, and Escape.
- A known service can transition between UP and DOWN when Prometheus data changes.
- Prometheus fetch failures do not stop animation or freeze the UI.
- Hover/pointer details remain aligned with the correct worker.
- Labels, weather effects, and scene-specific characters remain legible.

If no browser or execution environment is available, explicitly list browser checks as not run.

## Troubleshooting guide

### A service stays green when stopped

Do not start by editing the worker's color. Verify, in order:

- The exporter can reach the target and reports a failed probe.
- Prometheus is scraping the correct exporter and stores `probe_success` as `0`.
- The query returns the expected `host` and `app` labels.
- Nginx forwards `/api/v1/` to the correct Prometheus instance.
- The frontend parser matches the returned labels and updates the correct `host:app` key.
- The renderer uses the live state rather than a stale/default value.

Remember that a failed HTTP fetch and a failed probe are different conditions: the first means the dashboard cannot read monitoring data; the second is a monitoring result that should mark the corresponding service DOWN.

### A scene throws or has undefined labels

Check configuration keys, hash-derived indexes, array bounds, and whether optional fields are handled when omitted. Use unsigned hash shifts and avoid assuming every scene has every optional field.

### A layout or identity changes unexpectedly

Look for `Math.random()`, unstable array ordering, a changed `host:app` key, or layout calculations that depend on frame timing. Use deterministic helpers.

### Scene colors leak between themes

Move scene-specific values into `SC` and read the active configuration through `S`. Check that helpers do not mutate shared scene objects.

## Change checklist

Before considering a change complete:

- [ ] The change is limited to the requested behavior.
- [ ] Existing scenes and host/service data are preserved.
- [ ] The Prometheus contract and failure fallback remain intact, unless intentionally changed.
- [ ] Appearance and layout remain deterministic.
- [ ] Responsive layout and interaction still make sense.
- [ ] No secrets or unnecessary dependencies were introduced.
- [ ] Relevant smoke tests were run, and unrun checks are identified.
- [ ] Documentation was updated if behavior, configuration, or deployment assumptions changed.

## Style and scope

Use plain JavaScript and the existing CSS/HTML structure. Keep `index.html` self-contained unless the user requests a deliberate architecture change. Prefer simple, direct implementations over speculative abstractions. Do not perform opportunistic redesigns while fixing an unrelated bug.

When the user asks for a larger refactor, explain the trade-offs first, preserve behavior, and split the work into verifiable steps. When uncertain about an existing behavior, inspect the code rather than guessing.
