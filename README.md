# 🏡 Homelab Sanctuary

> **A living, animated map of your homelab.**
>
> Turn infrastructure health into a cozy little world: hosts become rooms, services become characters, and Prometheus becomes the heartbeat that keeps the sanctuary alive.

**Homelab Sanctuary** is a self-contained browser dashboard that visualizes the health of a homelab as an interactive 2D canvas scene. Instead of a conventional grid of status tiles, the application turns infrastructure into an animated environment where every monitored service has a visible presence.

It is deliberately lightweight: the application is essentially a single `index.html`, with no build system, bundler, framework, database, or application server required. A small Nginx configuration serves the page and proxies Prometheus' API through the same origin.

---

## ✨ What makes it different?

Traditional monitoring dashboards are excellent for numbers, graphs, and alerts. Homelab Sanctuary is designed for something slightly different:

**It makes infrastructure fun to look at.**

A server is no longer just a label. It becomes a department. A service is no longer just `probe_success = 1`; it becomes a worker sitting at a desk, an animal in a stable, a plant in a field, a student in a classroom, or a character in a themed world.

The visual state follows the real monitoring data:

- 🟢 **UP** — the service is healthy and active.
- 🔴 **DOWN** — the character reacts to the failure and the surrounding room reflects the problem.
- 🌦️ **Health-driven weather** — the more services fail in a room, the worse its weather becomes.
- 🌍 **Overall weather** — the header summarizes the health of the sanctuary.
- 🪪 **Interactive identity cards** — inspect a service to see its generated name, host, application, and status.

The result is a monitoring dashboard that feels more like exploring a tiny world than reading an operations console.

---

## 🎨 Scenes

The dashboard contains multiple visual worlds that all use the same underlying infrastructure data.

| Scene | Concept |
|---|---|
| 🌿 **Ghibli Inspired** | Cozy hand-painted office with departments, desks, windows, and workers. |
| 🌃 **Neon Tokyo** | Cyberpunk-style characters, neon lighting, and glitch effects. |
| 🚀 **Space Station** | Services become astronauts, aliens, robots, and a space dog. |
| 🕹️ **Retro 16-bit** | Pixel-art inspired infrastructure world. |
| 🐄 **Barnhouse** | Departments become stables and services become animals. |
| 🌱 **Farm** | Departments become plots and services become crops. |
| 🧙 **Fantasy Guild** | Services become wizards, knights, elves, dwarves, and alchemists. |
| 🌊 **Underwater Lab** | Services become underwater researchers and creatures. |
| 🏫 **High School** | Services become students while a teacher reacts to the health of the class. |

Scenes are configuration-driven. The monitoring model does not change when the visual theme changes.

---

## 🧭 Architecture

Homelab Sanctuary intentionally keeps the runtime architecture small:

```text
                         ┌─────────────────────┐
                         │       Browser       │
                         │                     │
                         │  Canvas UI          │
                         │  Scene rendering    │
                         │  Interaction       │
                         └──────────┬──────────┘
                                    │
                             same-origin /api/v1
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │        Nginx        │
                         │                     │
                         │  Static files       │
                         │  /api/v1 proxy      │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │     Prometheus      │
                         │                     │
                         │   probe_success     │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Monitoring targets  │
                         │ / blackbox exporter │
                         └─────────────────────┘
```

The browser does **not** connect directly to every monitored service. It asks Prometheus for the current `probe_success` time series. This keeps the dashboard independent of the individual service implementations.

### Why the Nginx proxy exists

The browser requests:

```text
/api/v1/query?query=probe_success
```

from the same origin that serves the dashboard.

Nginx forwards that path to:

```text
http://prometheus:9090/api/v1/
```

This avoids requiring CORS configuration in Prometheus and lets the dashboard communicate with Prometheus through a simple same-origin API.

The repository's `nginx.conf` listens on port `80` inside the container. If Docker publishes it as `86:80`, the dashboard is available on port **8088** from the LAN.

---

## 📁 Repository structure

The repository is intentionally tiny:

```text
homelab-sanctuary/
├── index.html       # Entire dashboard: UI, renderer, scenes and monitoring logic
├── nginx.conf       # Static-file server + Prometheus reverse proxy
├── CLAUDE.md        # Developer/AI-agent architecture and modification guide
├── README.md        # Project documentation
└── LICENSE          # GNU GPL v3
```

There is no `package.json`, build directory, backend application, database, or JavaScript dependency tree to maintain.

---

## 🚀 Quick start

### Option 1 — Serve the HTML directly

Because the dashboard is a single HTML file, it can be served by almost any static web server.

However, the browser still needs access to Prometheus through `/api/v1/query`. The recommended approach is therefore to use the included Nginx configuration.

### Option 2 — Docker + Nginx

A typical deployment looks like this:

```yaml
services:
  homelab-sanctuary:
    image: nginx:alpine
    container_name: homelab-sanctuary
    restart: unless-stopped
    ports:
      - "8088:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - utility_default

networks:
  utility_default:
    external: true
```

The important part is that the Nginx container and Prometheus must share a Docker network so that this works:

```nginx
proxy_pass http://prometheus:9090/api/v1/;
```

Then open:

```text
http://<homelab-host>:8088
```

For example:

```text
http://192.0.2.10:8088
```

Until the first successful Prometheus query returns labeled series, the dashboard shows a waiting state rather than inventing hosts or apps. Once metrics arrive, rooms and workers are created automatically.

---

## 📡 Monitoring data contract

The dashboard uses a single Prometheus instant query:

```http
GET /api/v1/query?query=probe_success
```

Each returned series must contain these labels:

| Field | Purpose |
|---|---|
| `metric.host` | Identifies the room/department. |
| `metric.app` | Identifies the service/worker inside that room. |
| `value[1]` | `"1"` means UP; any other value means DOWN. |

The dashboard discovers hosts and apps dynamically from the `host` and `app` labels returned by Prometheus. No host inventory needs to be hard-coded in `index.html`.

### Example Prometheus result

Conceptually, the application expects data resembling:

```json
{
  "metric": {
    "host": "example-host",
    "app": "example-service"
  },
  "value": [
    1760000000,
    "1"
  ]
}
```

A failed request does not crash the application. The dashboard keeps the existing state until a later polling attempt succeeds.

### Important behavior

Every valid `probe_success` result with non-empty `host` and `app` labels adds that host/app pair to the dashboard automatically. A new host creates a room; a new app creates a worker in that room. If a target has no matching Prometheus series, the dashboard cannot discover it. If a previously discovered series disappears, its last known state remains until the page is reloaded or a matching series returns.

---

## 🔭 Prometheus + Blackbox Exporter

> **Example values only:** `192.0.2.0/24` is reserved for documentation. The target IPs and ports below are placeholders; replace them with your own probe targets and ports before using this scrape configuration.

Homelab Sanctuary works particularly well with Prometheus' [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter), because the dashboard only needs a simple UP/DOWN result.

A typical probe configuration looks like this:

```yaml
scrape_configs:
  - job_name: homelab-probes
    metrics_path: /probe
    params:
      module: [http_2xx]

    static_configs:
      - targets: ['http://192.0.2.10:18080']
        labels:
          host: example-host
          app: example-service-one

      - targets: ['http://192.0.2.11:18081']
        labels:
          host: another-example-host
          app: example-service-two

    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target

      - source_labels: [__param_target]
        target_label: instance

      - target_label: __address__
        replacement: blackbox-exporter:9115
```

The exporter does the actual health check. Prometheus stores the resulting `probe_success` metric. Homelab Sanctuary simply visualizes it.

This means the dashboard can also work with another exporter or monitoring system as long as it produces the expected metric and labels.

---

## 🏢 Dynamic infrastructure model

Hosts and applications are not bundled into the frontend. On each Prometheus poll, the dashboard reads every `probe_success` result carrying non-empty `host` and `app` labels:

- A previously unseen `host` creates a room/department.
- A previously unseen `app` for that host creates a worker/character.
- The service state is updated from the metric value (`1` = UP; otherwise DOWN).
- Layout is recalculated when new hosts or apps are discovered.
- Scene visuals are derived from the discovered names, so arbitrary host labels do not require scene-specific edits.

To add infrastructure, configure Prometheus to expose `probe_success` series with the expected labels. You do not need to edit `index.html` to register a host or app. The labels are identifiers, so keep them stable across scrapes.

---

## 🌦️ Health-driven weather

Health is also used as a visual storytelling mechanism.

For each room, the application calculates the proportion of services that are down:

| Down services | Weather level | Typical meaning |
|---:|---|---|
| `0%` | Level 0 | Clear / healthy |
| `< 25%` | Level 1 | Clouds / early warning |
| `< 50%` | Level 2 | Rain / degraded |
| `≥ 50%` | Level 3 | Storm / major outage |

Scenes can rename or reinterpret these states. For example, a space-themed scene can present the same health levels as increasingly severe space phenomena instead of ordinary weather.

This allows the monitoring signal to influence the environment without changing the underlying monitoring model.

---

## 🐄 Barnhouse & 🌱 Farm

Two scenes replace the traditional office workers with living things.

### Barnhouse

Departments become animal stalls and services become animals.

Healthy animals can animate normally, while a failed service is represented by a visibly unhealthy animal, including effects such as an ice pack or thermometer.

Supported animal types include:

```text
horse · cow · sheep · pig · duck · chicken · goat · rabbit
```

### Farm

Departments become agricultural plots and services become crops.

Healthy crops sway gently. Failed services wilt.

Supported crop types include:

```text
tomato · watermelon · pumpkin · corn
rice · wheat · carrot · sunflower
```

Assignments can be explicitly mapped per host, with deterministic name-based fallbacks for services that are not explicitly assigned.

---

## 🎭 Themed characters

Several scenes contain their own character systems.

### Neon Tokyo

Services can become:

- Hacker with visor
- Android with LED face
- Cat-eared netrunner
- Ramen-delivery drone

A failed service receives a glitch-oriented failure effect.

### Space Station

Services can become:

- Astronaut
- Alien
- Maintenance robot
- Space dog

A failed service can appear to drift or sleep away from its workstation.

### Fantasy Guild

Services can become:

- Wizard
- Knight
- Elf archer
- Dwarf smith
- Alchemist

A failed service is represented as cursed or asleep and can lose its normal color treatment.

### Underwater Lab

Services can become:

- Diver
- Octopus
- Crab
- Jellyfish lamp-keeper
- Seahorse intern

A failed service can become tangled in a net.

Character assignment is deterministic, so a service does not randomly change identity between reloads.

---

## 🏫 High School scene

The High School scene uses a different metaphor while retaining the same monitoring model.

- Each service becomes a student.
- Monitors are replaced with open books.
- The classroom layout is centered around the class.
- A teacher is drawn at the front.
- The teacher's mood follows the room health.
- As more services fail, the classroom becomes increasingly unhappy.

This demonstrates one of the project's main design goals: **monitoring data can drive completely different visual representations without changing the data contract.**

---

## 🖱️ Interaction

The canvas is interactive on both desktop and touch devices.

### Desktop

- **Drag** — pan around the world.
- **Mouse wheel** — zoom.
- **Hover a worker** — show the service identity card.
- **Click controls** — change scenes or open the dedication.

### Touch

- **One-finger drag** — pan.
- **Pinch** — zoom.
- **Tap a worker** — open its identity card.

The scene initially auto-fits the complete layout. Once the user manually moves the camera, the application stops forcing the view back to the automatic fit.

---

## 🪪 Worker identity cards

Selecting a worker exposes a small information card containing:

- Generated worker name
- Generated worker ID
- Host / department
- Application / service
- Current UP/DOWN state

The dashboard therefore provides a human-friendly way to inspect the same service identity that exists in Prometheus.

---

## ❤️ Dedication

The header contains a heart button that opens a dedication modal.

The text is configured in the `DEDICATION` constant inside `index.html` and supports:

- `title`
- `text`
- optional `from`

The modal uses `textContent` for user-provided text rather than injecting HTML, keeping the dedication content safe as plain text.

---

## 🎨 Adding a scene

Scenes are configuration objects in the `SC` registry rather than separate pages or applications.

A scene can define:

- Display name
- Floor texture
- Walls and lighting
- Window sky colors
- Weather names/icons
- Desk and chair colors
- Monitor appearance
- Character systems
- Animals or crops
- Particles and environmental effects
- Decorations
- Open-sky layouts
- Pixel-art rendering
- Specialized layouts such as the classroom

The complete scene field reference is maintained in [`CLAUDE.md`](CLAUDE.md).

The renderer is designed so that scene-specific colors and behavior come from the active scene configuration instead of being scattered through the drawing functions.

---

## 🧩 Implementation details

The application is intentionally written as plain browser JavaScript.

The major areas inside `index.html` are:

1. **Dynamic host/app registry** — populated from Prometheus `host` and `app` labels at runtime.
2. **Art kit** — deterministic hashing, random generation, textures, floors, colors, and reusable drawing helpers.
3. **Scene system** — textures, scene configurations, active scene, health/weather mapping, and environmental effects.
4. **Rendering system** — rooms, workers, characters, furniture, windows, and decorations.
5. **Camera and layout** — room placement, worker placement, zoom, panning, and auto-fit behavior.
6. **Interaction layer** — hover/touch detection and identity cards.
7. **Prometheus synchronization** — periodic health polling and state updates.
8. **Header UI** — scene selection, weather summary, infrastructure summary, and dedication modal.

The entire application intentionally remains in one HTML file so it can be copied, served, backed up, and modified without a toolchain.

---

## 🧪 Testing

There is currently no automated browser test suite, build pipeline, or package manager.

The project's development guidance describes a lightweight smoke-test approach:

1. Execute the script with a stub 2D canvas context.
2. Load every configured scene.
3. Run several animation frames.
4. Repeat with selected `liveServiceState` entries set to `false`.
5. Confirm that no JavaScript exception is thrown.
6. Open the actual page in a browser for visual validation.

The browser check remains important because a canvas stub can detect JavaScript errors but cannot validate visual composition, animation, interaction, or responsive behavior.

---

## ⚠️ Important limitations

### Prometheus is expected at the same origin

The browser expects:

```text
/api/v1/query
```

If you do not use the supplied reverse-proxy approach, you will need to configure CORS appropriately and update the Prometheus request logic in `index.html`.

### Tailwind is loaded from a CDN

The HTML currently loads Tailwind from:

```text
https://cdn.tailwindcss.com
```

This is convenient for the small UI overlay but means the dashboard is not completely self-contained when deployed in an offline environment.

For a fully disconnected homelab, vendor Tailwind or replace the small set of utility classes with local CSS.

### No authentication layer

Homelab Sanctuary itself does not authenticate users or protect Prometheus.

Do **not** expose the dashboard together with an unauthenticated Prometheus API to the public Internet.

Put appropriate network controls, authentication, VPN access, or a protected reverse proxy in front of the monitoring stack when remote access is required.

### Initial service state

The host/app registry starts empty. Rooms and workers appear only after a successful Prometheus query returns `probe_success` series with non-empty `host` and `app` labels. The displayed UP/DOWN state is taken from each returned sample.

A missing series does not automatically mean DOWN. If a previously discovered series disappears, the dashboard keeps its last known state for the current page session. If you need absence of a probe to mean failure, implement that behavior in the monitoring/query layer rather than assuming the visualization can infer it.

---

## 🔧 Troubleshooting

### A service stays green even though it is down

Check these in order:

1. Confirm Prometheus has a `probe_success` series for the service.
2. Confirm the series contains non-empty `host` and `app` labels. The dashboard discovers these labels dynamically.
4. Confirm `value[1]` changes from `"1"` to another value when the service is unavailable.
5. Open the dashboard's `/api/v1/query?query=probe_success` endpoint through the same origin and inspect the returned JSON.
6. Confirm the dashboard's Nginx container can resolve and reach `prometheus:9090`.
7. Confirm the Nginx container and Prometheus container share the same Docker network.

A useful mental model is:

```text
Service DOWN
     ↓
Blackbox probe fails
     ↓
probe_success = 0
     ↓
Prometheus stores the series
     ↓
Browser queries probe_success
     ↓
host + app identify the worker
     ↓
Worker becomes DOWN
```

If the chain breaks anywhere before the browser receives the correct labeled series, the visualization cannot know that the service is down.

### The page loads but all services remain green

This usually means one of the following:

- Prometheus is not reachable through `/api/v1/query`.
- The query returns no matching series.
- The `host` or `app` label is missing or empty, so the dashboard cannot discover the series.
- Nginx cannot reach the Prometheus container.
- The page is being served from a different origin without the required CORS configuration.

### The page does not load at all

Check:

```text
Browser → Nginx → static index.html
```

before troubleshooting Prometheus.

The HTML page itself does not require Prometheus to render.

---

## 🔐 Security notes

Homelab Sanctuary is a visualization layer, not a security boundary.

Keep the following principles in mind:

- Do not publish Prometheus directly to the Internet.
- Do not expose the dashboard publicly unless the reverse proxy is protected appropriately.
- Avoid putting credentials, API keys, tokens, or private infrastructure secrets into `index.html`.
- Remember that anything embedded in a public JavaScript file is visible to the browser.
- If the dashboard is exposed through a VPN, keep the Prometheus endpoint restricted to the same trusted network where practical.

---

## 📜 License

This project is distributed under the **GNU General Public License v3.0**. See [`LICENSE`](LICENSE) for the complete license text.

---

## 🖼️ Screenshots

The repository includes screenshots showing the different visual scenes and monitoring states.

<img width="1226" height="1257" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/48f44181-5dfb-48f2-8f9a-20f3eb86ec54" />

<img width="1229" height="1262" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/91a9cab7-dca2-42ab-8471-0421122d11ca" />

<img width="1232" height="1264" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/b7ba5f37-34dd-4ce9-9d47-860e31692488" />

<img width="1232" height="1262" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/0101f384-7777-4c98-a99e-520455f0b6bc" />

<img width="1234" height="1267" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/9e83f008-b329-472a-a6e8-e0f5ccabeade" />

<img width="1230" height="1263" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/e9f7e1a5-9a15-4252-9455-196569a8f84b" />

<img width="1231" height="1264" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/7ed29b90-295f-4a1c-a81b-1798cb3d982f" />

<img width="1235" height="1263" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/43acc480-fb2d-4012-a19a-e8ca94bbf380" />

<img width="1234" height="1262" alt="Homelab Sanctuary screenshot" src="https://github.com/user-attachments/assets/fd4bf56b-d5af-4d96-b902-e37945d97ee3" />

---

## 💡 Project philosophy

Homelab Sanctuary is intentionally opinionated about one thing:

> **Infrastructure does not have to look boring.**

A homelab is a place to learn, experiment, break things, fix them, and build systems that are genuinely useful. Monitoring is part of that experience, so the monitoring dashboard can be useful **and** enjoyable.

The project favors:

- Minimal infrastructure
- Plain browser technology
- Deterministic visuals
- A small and understandable data contract
- Configuration over duplicated rendering logic
- Graceful failure when monitoring data is temporarily unavailable
- A visual representation that makes outages immediately noticeable

The result is a tiny monitoring application with a surprisingly large personality.
