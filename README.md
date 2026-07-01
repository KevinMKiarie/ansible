# Ansible Multi-Environment Infrastructure

A beginner-friendly Ansible project that provisions and manages **development**, **staging**, and **production** environments using roles, inventory groups, and environment-specific variables. Docker containers simulate real servers so the entire setup runs locally with zero cloud cost.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Environments](#environments)
- [Roles](#roles)
- [Playbooks](#playbooks)
- [How It Works](#how-it-works)
- [Adding a New Environment](#adding-a-new-environment)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project demonstrates how to use Ansible to manage multiple environments from a single codebase. Each environment (dev, staging, prod) has its own:

- **Inventory** — defines which servers belong to that environment
- **Group vars** — environment-specific variables (ports, versions, worker counts)
- **Isolated containers** — Docker containers act as servers for local testing

The same playbooks and roles run across all environments. Only the variables change.

---

## Architecture

```
Your Machine (macOS)
│
├── ansible-playbook -i inventories/dev/       → targets dev containers
├── ansible-playbook -i inventories/staging/   → targets staging containers
└── ansible-playbook -i inventories/prod/      → targets prod containers

Docker Containers (simulated servers)
┌─────────────────────────────────────────────────────────────┐
│     DEV                  STAGING              PROD          │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────┐  │
│  │ dev-web-01   │     │staging-web-01│     │prod-web-01  │  │
│  │ SSH: 2201    │     │ SSH: 2211    │     │ SSH: 2221   │  │
│  │ HTTP: 8080   │     │ HTTP: 8081   │     │ HTTP: 8082  │  │
│  └──────────────┘     └──────────────┘     └─────────────┘  │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────┐  │
│  │  dev-db-01   │     │staging-db-01 │     │ prod-db-01  │  │
│  │  SSH: 2202   │     │ SSH: 2212    │     │ SSH: 2223   │  │
│  └──────────────┘     └──────────────┘     └─────────────┘  │
└─────────────────────────────────────────────────────────────┘

Ansible connects via SSH → runs roles → configures each server
```

---

## Project Structure

```
ansible-multi-env/
├── ansible.cfg                        # Ansible configuration
├── docker-compose.yml                 # Defines all 6 containers
├── docker/
│   ├── Dockerfile                     # Debian SSH + Python image
│   └── authorized_keys                # Public key injected into containers
├── ansible_key                        # Private SSH key (gitignored)
├── ansible_key.pub                    # Public SSH key
├── inventories/
│   ├── dev/
│   │   ├── hosts.ini                  # Dev server list
│   │   └── group_vars/
│   │       └── all.yml                # Dev variables
│   ├── staging/
│   │   ├── hosts.ini                  # Staging server list
│   │   └── group_vars/
│   │       └── all.yml                # Staging variables
│   └── prod/
│       ├── hosts.ini                  # Prod server list
│       └── group_vars/
│           └── all.yml                # Prod variables
├── roles/
│   ├── common/
│   │   └── tasks/
│   │       └── main.yml               # Base setup for all servers
│   ├── webserver/
│   │   ├── tasks/
│   │   │   └── main.yml               # Nginx install and config
│   │   ├── handlers/
│   │   │   └── main.yml               # Restart Nginx on config change
│   │   └── templates/
│   │       └── nginx.conf.j2          # Nginx config template
│   └── app/
│       ├── tasks/
│       │   └── main.yml               # App deployment tasks
│       └── templates/
│           └── index.html.j2          # HTML template with env variables
└── playbooks/
    ├── site.yml                       # Master playbook (full setup)
    └── deploy.yml                     # App-only deployment
```

---

## Prerequisites

| Tool | Minimum Version | Install |
|---|---|---|
| Ansible | 2.14+ | `brew install ansible` |
| Docker Desktop | 4.0+ | [docker.com](https://www.docker.com/products/docker-desktop) |
| Docker Compose | v2+ | Included with Docker Desktop |
| Python | 3.8+ | `brew install python` |

Verify everything is installed:

```bash
ansible --version
docker --version
docker compose version
python3 --version
```

---

## Quick Start

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd ansible-multi-env
```

### 2. Generate SSH keys

```bash
ssh-keygen -t ed25519 -f ./ansible_key -N ""
cp ansible_key.pub docker/authorized_keys
```

### 3. Start the containers

```bash
docker compose up -d --build
```

Verify all 6 containers are running:

```bash
docker compose ps
```

### 4. Test Ansible connectivity

```bash
ansible all -i inventories/dev/ -m ping
```

Expected output:
```
dev-web-01 | SUCCESS => { "ping": "pong" }
dev-db-01  | SUCCESS => { "ping": "pong" }
```

### 5. Run the playbook

```bash
# Dev
ansible-playbook playbooks/site.yml -i inventories/dev/

# Staging
ansible-playbook playbooks/site.yml -i inventories/staging/

# Prod
ansible-playbook playbooks/site.yml -i inventories/prod/
```

### 6. Verify each environment

```bash
curl http://localhost:8080   # Dev
curl http://localhost:8081   # Staging
curl http://localhost:8082   # Prod
```

---

## Environments

Each environment is isolated with its own variables defined in `group_vars/all.yml`.

| Variable | Dev | Staging | Prod |
|---|---|---|---|
| `env` | development | staging | production |
| `app_port` | 8080 | 8080 | 80 |
| `nginx_worker_processes` | 1 | 2 | 4 |
| `app_version` | latest | latest | 1.0.0 |
| `db_name` | app_dev | app_staging | app_prod |

Prod intentionally pins `app_version` to a specific release and runs more Nginx workers to reflect a real production configuration.

---

## Roles

### `common`
Runs on every server regardless of its function. Handles:
- APT cache update
- Installing base packages (`curl`, `git`, `htop`, `vim`)
- Setting timezone to `Etc/UTC`
- Creating the `/opt/app` application directory

### `webserver`
Runs on hosts in the `[webservers]` group. Handles:
- Installing Nginx
- Deploying `nginx.conf` from a Jinja2 template (injects `app_port` and `nginx_worker_processes`)
- Ensuring Nginx is enabled and running
- Triggering a restart via handler when config changes

### `app`
Runs on hosts in the `[webservers]` group after `webserver`. Handles:
- Ensuring `/opt/app` exists with correct permissions
- Deploying `index.html` from a Jinja2 template (injects `env`, `app_version`, `app_port`)

---

## Playbooks

### `playbooks/site.yml`
Full provisioning playbook. Runs `common` on all servers, then `webserver` and `app` on web servers only.

```bash
ansible-playbook playbooks/site.yml -i inventories/dev/
```

### `playbooks/deploy.yml`
Lightweight playbook for redeploying the app without re-running the full setup. Useful when you only change application files.

```bash
ansible-playbook playbooks/deploy.yml -i inventories/prod/
```

---

## How It Works

### Inventory groups
Each `hosts.ini` defines two groups: `[webservers]` and `[databases]`. The `site.yml` playbook targets them separately — `common` runs on `all`, while `webserver` and `app` run only on `webservers`.

### Jinja2 templates
Ansible uses Jinja2 to inject variables into config files at deploy time. For example, `nginx.conf.j2` contains:

```nginx
worker_processes {{ nginx_worker_processes }};
...
listen {{ app_port }};
```

When deployed to dev, this becomes `worker_processes 1` and `listen 8080`. In prod it becomes `worker_processes 4` and `listen 80`.

### Idempotency
Every task is idempotent — running the playbook multiple times produces the same result without unintended side effects. Ansible checks the current state before making changes and skips tasks that are already in the desired state.

### Handlers
The `Restart Nginx` handler in the webserver role only fires when the config file actually changes. This avoids unnecessary restarts on every playbook run.

---

## Adding a New Environment

1. Create the inventory directory:
```bash
mkdir -p inventories/qa/group_vars
```

2. Create `inventories/qa/hosts.ini` with your server IPs and ports.

3. Create `inventories/qa/group_vars/all.yml` with environment-specific variables.

4. If using Docker, add the new containers to `docker-compose.yml` with unique ports.

5. Run the playbook:
```bash
ansible-playbook playbooks/site.yml -i inventories/qa/
```

---

## Troubleshooting

### Ansible can't connect to a host
```bash
# Check containers are running
docker compose ps

# Test SSH manually
ssh -i ./ansible_key -p 2201 ubuntu@localhost

# Test Ansible ping
ansible all -i inventories/dev/ -m ping
```

### Docker build fails with network error
Add DNS to Docker Engine settings (`Settings → Docker Engine`):
```json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```
Then rebuild:
```bash
docker compose up -d --build
```

### Nginx serving default page instead of app
The containers are ephemeral. If you recreate them with `docker compose up -d`, re-run the playbook:
```bash
ansible-playbook playbooks/site.yml -i inventories/dev/
```

### Port already in use
Check what's using the port:
```bash
lsof -i :8080
```
Stop the conflicting process or change the port mapping in `docker-compose.yml`.
