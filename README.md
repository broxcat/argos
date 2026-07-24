# Argos — Zabbix Homelab

A self-hosted Zabbix monitoring stack for homelab / infrastructure supervision purposes, running via Docker Compose (MySQL backend, Zabbix server, Zabbix web frontend, Zabbix agent2 for host/container monitoring) behind an nginx reverse proxy with self-signed TLS. Named after Argus Panoptes, the all-seeing giant of Greek mythology — fitting for a monitoring tool.

> ⚠️ **This is a homelab / lab setup, not a production-hardened deployment.** Self-signed TLS certificates, single-host MySQL, and no external backups are configured by default. See [Security notes](#security-notes) before exposing this beyond your local network.

## Prerequisites

- A Linux host (Ubuntu/Debian tested) with Docker and Docker Compose
- OpenSSL (for generating the self-signed certificate)
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

Set `MYSQL_ROOT_PASSWORD` to a strong, unique value. You can generate it with:

```bash
openssl rand -base64 24
```

> 🔒 `.env` is git-ignored and must **never** be committed. See [Security notes](#security-notes).

### 5. Generate a self-signed TLS certificate (for nginx)

```bash
mkdir -p certs
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout certs/zabbix.key \
  -out certs/zabbix.crt \
  -days 365 \
  -subj "/CN=localhost"
```

### 6. Start the stack

```bash
docker compose up -d
```

Follow the logs until the server reports it's ready and the web frontend can reach the database:

```bash
docker compose logs -f zabbix-server
```

### 7. Access the platform

Browse to `https://<your-host>` and log in with the default Zabbix credentials (`Admin` / `zabbix`), then change the admin password immediately. Your browser will warn about the self-signed certificate — that's expected for a local/homelab setup.

## Services included

| Service | Image | Role |
|---|---|---|
| zabbix-mysql | `mysql:${MYSQL_VERSION}` | Database backend for Zabbix |
| zabbix-server | `zabbix/zabbix-server-mysql:${ZABBIX_VERSION}` | Core Zabbix server (polling, triggers, alerting) |
| zabbix-web | `zabbix/zabbix-web-nginx-mysql:${ZABBIX_VERSION}` | Web frontend (PHP), reachable only from the internal network |
| zabbix-agent | `zabbix/zabbix-agent2:${ZABBIX_VERSION}` | Local agent monitoring the Docker host (containers, processes, filesystem) |
| nginx | `nginx:${NGINX_VERSION}` | Reverse proxy terminating TLS and forwarding to `zabbix-web` |

## Security notes

- **`zabbix-agent` host-level mounts** (`/var/run/docker.sock`, `/proc`, `/sys`, and `/:/hostfs:ro`) give the agent deep visibility into the Docker host — container control surface via `docker.sock`, and read access to host processes and filesystem. This container should never be exposed to untrusted networks, and the `zabbix-net` network should stay internal to the host.
- **DB credentials** (`MYSQL_ROOT_PASSWORD`) live only in `.env`, which is git-ignored and must never be committed. If you ever commit it by mistake, rotate it immediately. Note that `zabbix-server` and `zabbix-web` connect to MySQL as `root` using this password, since there is no separate `zabbix` DB user.
- **Self-signed TLS** is fine for a local homelab. If you expose this instance beyond `localhost`, replace it with a certificate from a real CA (e.g. Let's Encrypt) and put it behind proper network access controls.
- **`.env` and `certs/`** are excluded via `.gitignore` and must never be pushed to this or any fork of this repo.
- **Ports**: only nginx (`80`/`443`) and `zabbix-server`'s trapper port (`10051`, needed for active agent checks) are published. `zabbix-web`'s direct port and the MySQL port are not exposed externally — they're only reachable from other containers on `zabbix-net`.

## Maintenance

- Pin image versions via the `MYSQL_VERSION`, `ZABBIX_VERSION`, and `NGINX_VERSION` variables in `.env` rather than tracking `latest` indefinitely, to avoid unplanned upgrades.
- Back up the `zabbix-mysql-data` volume regularly — it holds all historical monitoring data and configuration.

## Credits / inspiration

This setup follows the official Zabbix container deployment guide:
- [Zabbix Documentation — Installation with containers](https://www.zabbix.com/documentation/current/en/manual/installation/containers)
