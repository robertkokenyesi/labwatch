# labwatch

A small, self-hosted monitoring stack for my homelab, built with Docker Compose. It watches both the server itself and every container running on it.

## What's inside

| Service | Job |
| --- | --- |
| **Prometheus** | Collects and stores metrics, scraping every 15 seconds |
| **Node Exporter** | Reports the host's CPU, memory, disk and network stats |
| **cAdvisor** | Reports per-container CPU, memory and network usage |
| **Grafana** | Dashboards, reading its data from Prometheus |

## How it fits together

Exporters (Node Exporter, cAdvisor) expose numbers. Prometheus visits them and stores what it finds. Grafana asks Prometheus for the data and draws the dashboards.

All services share one existing external Docker network, `rcomputing`, and find each other by container name (for example `node-exporter:9100`). No ports are published on the host. Grafana is reached through a reverse proxy (Nginx Proxy Manager) on the same network at `grafana:3000`.

## Project structure

```
labwatch/
  docker-compose.yml
  .gitignore
  grafana/
    .env.example
    data/
      .gitkeep
  prometheus/
    prometheus.yml
```

The real `grafana/.env` file and the contents of the data folders are created on the server and are never committed. `grafana/data/.gitkeep` only exists so Git keeps the empty folder.

## Requirements

- Docker and Docker Compose
- A Docker network for the stack to join. In my setup it is called `rcomputing`. Use your own: create one with `docker network create <your-network-name>`, or check existing ones with `docker network ls`, then replace `rcomputing` with your network name in all four services and in the `networks` section at the bottom of `docker-compose.yml`.

## Setup

1. Clone the repo on the server:

   ```
   git clone <your-repo-url> labwatch
   cd labwatch
   ```
2. Create the data folders if they are missing (Git does not carry their contents):

   ```
   mkdir -p prometheus/data grafana/data
   ```
3. Set the folder permissions. The containers run as non-root users, so they cannot write to folders owned by your own user. Prometheus runs as user 65534 and Grafana as user 472. Without this step the containers crash on start with permission errors:

   ```
   sudo chown -R 65534:65534 prometheus/data
   sudo chown -R 472:472 grafana/data
   ```
4. Rename the example file to `.env` inside the grafana folder, then set a strong Grafana admin password in it. The Compose file reads `./grafana/.env`:

   ```
   cp grafana/.env.example grafana/.env
   nvim grafana/.env
   ```

   The file contains one line, `GF_SECURITY_ADMIN_PASSWORD=`, followed by your password.
5. Check the Compose file is valid, then start the stack:

   ```
   docker compose config
   docker compose up -d
   ```
6. Check everything is running:

   ```
   docker compose ps
   ```
7. Run `git status`. If any files from the data folders show up as changes, adjust `.gitignore` so they are never committed.

## First-time Grafana setup

1. Log in with user `admin` and the password from your `.env` file.
2. Go to Connections, then Data sources, and add **Prometheus** with the URL `http://prometheus:9090`.
3. Import a dashboard by ID: a Node Exporter dashboard for host stats, and a cAdvisor or Docker dashboard for container stats.
4. In Prometheus, open Status, then Targets, and confirm every target shows as UP.

## Secrets

No passwords live in this repository. `grafana/.env.example` holds a placeholder only; the real `grafana/.env` is created on the server and is gitignored. If a real secret is ever committed by accident, change it immediately, because deleting the file does not remove it from Git history.

## Backups

Everything lives under this one folder: the Compose file, the Prometheus config, and both data folders. Back up the folder to back up the stack. For a clean copy of the Prometheus data, stop the container briefly first (`docker compose stop prometheus`).

## Updating

```
docker compose pull
docker compose up -d
```

Never run `docker compose down -v` unless you intend to wipe volumes.

## Adding more targets

To monitor something else, add an exporter service to `docker-compose.yml` on the `rcomputing` network, then add one more job to `prometheus/prometheus.yml` pointing at its container name and port. Run `docker compose up -d`, then restart Prometheus so it reads the new config.

## Security notes

- No service publishes a port. Only Grafana should be exposed, and only through the reverse proxy.
- cAdvisor runs `privileged: true` (as in its upstream examples) so it can read container stats. This is acceptable only because it is never published and only Prometheus talks to it.

## Roadmap

- Uptime Kuma for up/down alerting on websites
- Optional Alertmanager for disk and memory alerts
- Pre-provisioned Grafana data source and dashboards
- MySQL/MariaDB exporters for database metrics

## Credits

Initial ideas borrowed from [stefanprodan/dockprom](https://github.com/stefanprodan/dockprom), then rewritten from scratch as a leaner stack.
