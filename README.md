# Argos — Zabbix Homelab

A self-hosted Zabbix monitoring stack for homelab / infrastructure supervision purposes, running via Docker Compose (MySQL backend, Zabbix server, Zabbix web frontend on nginx, and Zabbix agent2 for host/container monitoring). Named after Argus Panoptes, the all-seeing giant of Greek mythology — fitting for a monitoring tool.

> ⚠️ **This is a homelab / lab setup, not a production-hardened deployment.** No TLS termination, single-host MySQL, and no external backups are configured by default. See [Security notes](#security-notes) before exposing this beyond your local network.

## Prerequisites

- A Linux host (Ubuntu/Debian tested) with Docker and Docker Compose
- Git

## Installation

### 1. Install Docker and dependencies

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common git docker.io docker-compose -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER && newgrp docker
```

### 2. Clone the repository

```bash
git clone https://github.com/broxcat/argos.git
cd argos
```

### 3. Configure your environment

Copy the sample file and fill in your own values:

```bash
cp .env.sample .env
nano .env
```

### 4. Generate strong secrets

Set `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD` to strong, unique values. You can generate them with:

```bash
openssl rand -base64 24
```

> 🔒 `.env` is git-ignored and must **never** be committed. See [Security notes](#security-notes).

### 5. Start the stack

```bash
docker compose up -d
```

Follow the logs until the server reports it's ready and the web frontend can reach the database:

```bash
docker compose logs -f zabbix-server
```

### 6. Access the platform

Browse to `http://<your-host>:8080` and log in with the default Zabbix credentials (`Admin` / `zabbix`), then change the admin password immediately.

## Services included

| Service | Image | Role |
|---|---|---|
| zabbix-mysql | `mysql:8.0` | Database backend for Zabbix |
| zabbix-server | `zabbix/zabbix-server-mysql:ubuntu-7.0-latest` | Core Zabbix server (polling, triggers, alerting) |
| zabbix-web | `zabbix/zabbix-web-nginx-mysql:ubuntu-7.0-latest` | Web frontend (nginx + PHP), served directly on its own port |
| zabbix-agent | `zabbix/zabbix-agent2:ubuntu-7.0-latest` | Local agent monitoring the Docker host (containers, processes, filesystem) |

## Security notes

- **`zabbix-agent` host-level mounts** (`/var/run/docker.sock`, `/proc`, `/sys`, and `/:/hostfs:ro`) give the agent deep visibility into the Docker host — container control surface via `docker.sock`, and read access to host processes and filesystem. This container should never be exposed to untrusted networks, and the `zabbix-net` network should stay internal to the host.
- **DB credentials** (`MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`) live only in `.env`, which is git-ignored and must never be committed. If you ever commit it by mistake, rotate every credential inside immediately.
- **Ports**: only `zabbix-web` (`8080`) and `zabbix-server`'s trapper port (`10051`, needed for active agent checks) are published. The MySQL port is not exposed externally by default — it's only reachable from other containers on `zabbix-net`.

## Maintenance

- Pin image versions in `docker-compose.yml` rather than tracking `latest`/`ubuntu-7.0-latest` indefinitely, to avoid unplanned upgrades.
- Back up the `zabbix-mysql-data` volume regularly — it holds all historical monitoring data and configuration.

## Credits / inspiration

This setup follows the official Zabbix container deployment guide:
- [Zabbix Documentation — Installation with containers](https://www.zabbix.com/documentation/current/en/manual/installation/containers)
