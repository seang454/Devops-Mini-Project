# Infrastructure Ansible Playbooks — GCP Edition

## Overview

A fully automated Ansible project that provisions a 3-machine CI/CD infrastructure on **Google Cloud Platform**, configures DNS via Cloudflare, installs all services, and retrieves credentials — all from a single playbook run.

| Machine | Role | Services |
|---------|------|----------|
| master01 | CI/CD | Jenkins BlueOcean + DinD, glab, Ansible, Nginx, Certbot, Portainer |
| master02 | Code Quality | SonarQube + PostgreSQL, Nginx, Certbot, Portainer |
| master03 | Artifact Registry | Nexus Repository (Docker + Helm), Nginx, Certbot, Portainer |

---

## How It Works

The main entry point is `playbooks/main-iac.yaml`. It runs 4 plays in sequence — no manual steps between them.

```
PLAY 1 — localhost
  └─ Setup gcloud env (venv + auth)
  └─ Create GCP service account (if key doesn't exist yet)
  └─ Create 3 GCP VMs (master01, master02, master03)
  └─ Wait 60s for VMs to boot
  └─ Fetch each VM's external IP via gcloud CLI

PLAY 1.5 — remote hosts
  └─ Refresh inventory (picks up new IPs)
  └─ Wait for SSH on port 22 (timeout 300s)
  └─ Ping each host — abort if any unreachable

PLAY 2 — remote hosts
  └─ Gather facts
  └─ Run common role (Docker, Oh My Zsh, Portainer)
  └─ Run jenkins role   → master01
  └─ Run sonarqube role → master02
  └─ Run nexus role     → master03

PLAY 3 — localhost
  └─ Health-check all services (40 retries × 15s delay)
  └─ Retrieve and print initial credentials
```

---

## What You Need to Change

> These are the only things you need to edit before running. Everything else is automated.

---

### 1. `group_vars/all.yml`

#### Your identity & paths

```yaml
agent_user: "seang"           # ← change to your Linux username
venv_path: "/home/seang/venv" # ← must match: /home/<agent_user>/venv
ssh_pubkey_path: "/home/seang/.ssh/id_rsa.pub"         # ← your SSH public key path
credential_folder: "/home/seang/miniproject/firstpro/credentials"  # ← where GCP keys are stored
```

#### GCP project

```yaml
google_project_id: "project-8924f831-......"  # ← your GCP project ID
```

Find this in [GCP Console → Project Info](https://console.cloud.google.com/).

#### Service account

```yaml
sa_name: "serviceaccount"              # ← name of the GCP service account to create/use
sa_display_name: "My Service Account"  # ← human-readable label (cosmetic only)
```

The generated key will be saved as `{{ credential_folder }}/{{ sa_name }}-key.json`. The playbook skips creation if the file already exists.

#### Machine sizes & zones

```yaml
machine_specs:
  large:
    machine_type: "e2-standard-2"   # ← master01 size
    zone: "asia-northeast3-a"       # ← master01 zone
    disk_size: 20
    disk_type: "pd-standard"
  medium:
    machine_type: "e2-medium"       # ← master02 & master03 size
    zone: "asia-northeast3-b"
    disk_size: 20
    disk_type: "pd-standard"
```

> master02 and master03 are hardcoded to `asia-east2-a` inside `machines_info`. Change their `zone` directly there if needed.

#### Ubuntu version

```yaml
ubuntu_specs:
  v22:
    image: "projects/ubuntu-os-cloud/global/images/family/ubuntu-2204-lts"  # ← currently in use
  v24:
    image: "projects/ubuntu-os-cloud/global/images/family/ubuntu-2404-lts"  # ← switch to 24.04
```

To switch to Ubuntu 24.04, update each entry in `machines_info` from `{{ ubuntu_specs.v22.image }}` to `{{ ubuntu_specs.v24.image }}`.

#### Domain & email

```yaml
base_domain: "seang.shop"                   # ← your domain
certbot_email: "pengseangsim210@gmail.com"  # ← Let's Encrypt registration email
```

#### DNS subdomains

```yaml
dns_mappings:
  - subdomain: "jenkins-test"      # → jenkins-test.seang.shop      → master01
  - subdomain: "sonarqube-test"    # → sonarqube-test.seang.shop    → master02
  - subdomain: "nexus-test"        # → nexus-test.seang.shop        → master03
  - subdomain: "docker-repo-test"  # → docker-repo-test.seang.shop  → master03
```

Only change the `subdomain` values. The `machine_name` mapping is automatic via `machines_info` index — don't change those unless you also reorder `machines_info`.

#### Jenkins container name

```yaml
jenkins_container_name: "myjenkins-blueocean"  # ← used in docker exec commands
```

---

### 2. `group_vars/secrets.yml`

This file holds sensitive values and should **never be committed to git**. Add it to `.gitignore`.

```yaml
# group_vars/secrets.yml
cloudflare_api_token: "your_cloudflare_api_token"
cloudflare_zone_id: "your_cloudflare_zone_id"
sonarqube_db_password: "a_strong_password_here"
```

Encrypt it with Ansible Vault to store it safely:

```bash
ansible-vault encrypt group_vars/secrets.yml
```

---

### 3. `inventory.ini`

Set the correct SSH user and Python interpreter per host. The `ansible_python_interpreter` must be set **here** per-host, not in `group_vars/all.yml` — setting it globally causes Ansible to use the venv Python on localhost during `Gathering Facts`, before the venv exists, which breaks the first run.

```ini
[jenkins_hosts]
master01 ansible_host=<filled-automatically> ansible_user=seang ansible_python_interpreter=/home/seang/venv/bin/python3

[sonarqube_hosts]
master02 ansible_host=<filled-automatically> ansible_user=seang ansible_python_interpreter=/home/seang/venv/bin/python3

[nexus_hosts]
master03 ansible_host=<filled-automatically> ansible_user=seang ansible_python_interpreter=/home/seang/venv/bin/python3
```

Change `seang` to your `agent_user` value in all three entries.

### 4. `roles/jenkins/var` — Jenkins

```yaml
jenkins_image_version: "lts-jdk21"  # ← Jenkins Docker image tag , you change as you wish 
```

Common values:

| Tag | Description |
|-----|-------------|
| `lts-jdk21` | Latest LTS on JDK 21 (current default) |
| `lts-jdk17` | Latest LTS on JDK 17 |
| `2.492-jdk21` | Pinned version on JDK 21 |

Browse all tags at [hub.docker.com/r/jenkins/jenkins/tags](https://hub.docker.com/r/jenkins/jenkins/tags).

---

### 5. `roles/sonarqube` — SonarQube

```yaml
sonarqube_admin_password: "Seang "  # ← Change this before deploying to production, you change as you wish 
```

This replaces the default `admin` / `admin` login on first boot. Pick a strong password — SonarQube enforces a minimum complexity requirement.

---

## Running the Playbooks

### Provision + deploy everything

```bash
ansible-playbook -i inventory.ini playbooks/main-iac.yaml
```

### Destroy everything

```bash
# ⚠️ WARNING: Deletes ALL GCP VMs, containers, volumes, and data
ansible-playbook -i inventory.ini playbooks/destroy.yaml
```

### Deploy to a single machine only

```bash
ansible-playbook -i inventory.ini playbooks/main-iac.yaml --limit master01
```

### Dry run (no changes made)

```bash
ansible-playbook -i inventory.ini playbooks/main-iac.yaml --check
```

---

## After Setup

Services are available at these URLs once the playbook completes. PLAY 3 prints initial credentials automatically.

| Service | URL | Default Credentials |
|---------|-----|---------------------|
| Jenkins | `https://jenkins-test.seang.shop` | Printed by PLAY 3 |
| SonarQube | `https://sonarqube-test.seang.shop` | `admin` / `admin` — change immediately |
| Nexus | `https://nexus-test.seang.shop` | Printed by PLAY 3 |
| Docker Registry | `https://docker-repo-test.seang.shop` | Same as Nexus |
| Portainer | `https://<machine-ip>:9443` | Set on first login |

To manually retrieve credentials at any time:

```bash
# Jenkins
docker exec myjenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword

# Nexus
docker exec nexus cat /nexus-data/admin.password
```

---

## Directory Structure

```
ansible-infra/
├── ansible.cfg
├── inventory.ini                      # ← Set ansible_user and ansible_python_interpreter here
├── requirements.yml
├── group_vars/
│   ├── all.yml                        # ← Main config — most changes go here
│   └── secrets.yml                    # ← Cloudflare tokens + DB passwords (never commit)
├── host_vars/
│   ├── master01.yml
│   ├── master02.yml
│   └── master03.yml
└── playbooks/
    ├── main-iac.yaml                  # ← Entry point — runs all 4 plays in sequence
    ├── destroy.yaml                   # ← Tears down all GCP VMs
    └── tasks/
        ├── setup-env.yaml             # gcloud auth + venv setup
        ├── create-masters-task.yaml   # GCP VM creation
        ├── gap-password.yaml          # Health checks + credential retrieval
        └── destroy-machine-tasks.yaml
```

---

## Quick Reference — Everything to Change

| File | Variable | What to set |
|------|----------|-------------|
| `group_vars/all.yml` | `agent_user` | Your Linux username |
| `group_vars/all.yml` | `google_project_id` | Your GCP project ID |
| `group_vars/all.yml` | `credential_folder` | Path to store GCP keys |
| `group_vars/all.yml` | `ssh_pubkey_path` | Your SSH public key path |
| `group_vars/all.yml` | `sa_name` | GCP service account name |
| `group_vars/all.yml` | `base_domain` | Your domain (e.g. `example.com`) |
| `group_vars/all.yml` | `certbot_email` | Email for Let's Encrypt |
| `group_vars/all.yml` | `dns_mappings[*].subdomain` | Subdomain name for each service |
| `group_vars/all.yml` | `machine_specs` | VM size and zone per tier |
| `group_vars/secrets.yml` | `cloudflare_api_token` | Cloudflare API token |
| `group_vars/secrets.yml` | `cloudflare_zone_id` | Cloudflare zone ID |
| `inventory.ini` | `ansible_user` | SSH user on each VM (repeat of `agent_user`) |
| `host_vars/master01.yml` | `jenkins_image_version` | Jenkins Docker image tag (e.g. `lts-jdk21`) |
| `host_vars/master02.yml` | `sonarqube_admin_password` | SonarQube admin password (replaces default `admin`/`admin`) |