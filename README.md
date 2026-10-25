# Homelab Sanctuary

A single-file, canvas-based status board for your homelab. Hosts become **office departments**, services become **workers at dual-monitor desks**, and live health comes from **Prometheus**. A healthy service has a calm worker and a green light. A down service has a slumped worker, flashing red LEDs and a pulsing red room border.

No build step, no framework: just `index.html` (Tailwind is loaded from a CDN for the header and panels).

## Features

- Live status from Prometheus `probe_success`, refreshed every 10 seconds
- 9 scenes, switchable from the header: Ghibli Inspired, Neon Tokyo, Space Station, Retro 16-bit, Barnhouse, Farm, Fantasy Guild, Underwater Lab, High School (students are the services, and the teacher at the front gets angrier as more of them go down)
- Health-driven weather: each room's window shows clear, cloudy, rainy or stormy skies depending on how many of its services are down, and the header shows the overall weather
- Hover a worker for an ID card (name, host, service, status). Drag to pan and scroll to zoom with a mouse; on touch screens drag with one finger to pan, pinch to zoom, and tap a worker for the ID card. The view auto-fits until you move it.
- Your scene choice is remembered in `localStorage`

## Quick start

1. Add probes to Prometheus (see below).
2. Serve `index.html` from the **same origin** as Prometheus' HTTP API, so that `GET /api/v1/query` works. A reverse proxy is the easiest way:

```nginx
server {
  listen 86;
  root /var/www/homelab-sanctuary;      # contains index.html
  location / { try_files $uri /index.html; }
  location /api/v1/ { proxy_pass http://prometheus:9090/api/v1/; }
}
```

3. Open `http://your-host:86`. Without Prometheus the page still loads and shows every service as up.

## Prometheus data format

The page runs one query:

```
GET /api/v1/query?query=probe_success
```

and reads each series in `data.result`:

| Part | Meaning |
|---|---|
| `metric.host` | Department (room). Must match a key in `staticHostSchema`. |
| `metric.app` | Service (worker). Must match an entry in that host's list. |
| `value[1]` | `"1"` = up, anything else = down. |

Series without both labels, or not listed in `staticHostSchema`, are ignored. Services that Prometheus has no series for stay "up".

Example with [blackbox_exporter](https://github.com/prometheus/blackbox_exporter), adding the `host` and `app` labels as static labels:

```yaml
scrape_configs:
  - job_name: homelab-probes
    metrics_path: /probe
    params: { module: [http_2xx] }
    static_configs:
      - targets: ['http://192.168.0.22:3000']
        labels: { host: util-pve, app: grafana }
      - targets: ['http://192.168.0.21:8080']
        labels: { host: dev-pve, app: jenkins }
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

Any exporter works as long as the metric is named `probe_success` (or you change the query) and carries `host` and `app` labels.

## Configure your own hosts and services

Edit `staticHostSchema` near the top of the script:

```js
const staticHostSchema = {
  "dev-pve": ["jenkins", "redis", "postgres"],
  "util-pve": ["grafana", "pi-hole"]
};
```

Each key becomes a room, each string a worker. Names, looks and ID numbers are derived from a hash of the host and service name, so they stay the same between reloads. To give a room a Japanese title on its nameplate, add it to the `JP` map. To choose a floor in the Ghibli scene, edit `THEME`.

## Barnhouse and Farm

In **Barnhouse** each department is a stable and each service is an animal in its stall. In **Farm** each department is a plot and each service is a plant. A healthy animal chews and blinks and a healthy plant sways. A down animal droops with an ice pack and thermometer, and a down plant wilts. Choose which animal or crop each host gets in the `SP` and `CR` maps near the top of the script (horse, cow, sheep, pig, duck, chicken, goat, rabbit; tomato, watermelon, pumpkin, corn, rice, wheat, carrot, sunflower). Hosts you don't list get one by name hash.

## Characters in the themed scenes

- **Neon Tokyo:** hacker with a visor, android with an LED face, cat-eared netrunner, ramen-delivery drone. A down service glitches and flickers with an "ERR" bubble.
- **Space Station:** astronaut, alien, maintenance robot, space dog. A down service floats off asleep.
- **Fantasy Guild:** wizard, knight, elf archer, dwarf smith, alchemist. A down service is cursed asleep and turns gray.
- **Underwater Lab:** diver, octopus, crab, jellyfish lamp-keeper, seahorse intern. A down service is tangled in a net.

Which character a service gets comes from its name hash, so it is stable between reloads.

## Dedication (optional)

The heart icon in the header opens a small modal. Put your own text in the `DEDICATION` constant near the end of the script (`title`, `text` and an optional `from` line). Line breaks in `text` are kept.

## Scenes and weather

Scenes are small config objects in the `SC` map, so adding one is mostly choosing colors, a floor texture and a decoration. See `CLAUDE.md` for the field list.

Weather level is computed from the share of down services: 0 down is level 0 (clear), under 25% is level 1 (clouds), under 50% is level 2 (rain), 50% or more is level 3 (storm with lightning). Each scene renames the levels, for example "Solar flare" in the space station.

## Limitations

- Prometheus must be reachable at `/api/v1/query` on the same origin (or add CORS headers and change the URL in `fetchPrometheusMetrics`).
- Tailwind comes from `cdn.tailwindcss.com`. For an offline homelab, vendor it or replace it with compiled CSS.
- No authentication. Do not expose it publicly with an unauthenticated Prometheus behind it.

## License

Add your license of choice here (MIT is common).
