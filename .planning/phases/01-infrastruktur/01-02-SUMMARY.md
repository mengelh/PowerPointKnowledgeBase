---
phase: 01-infrastruktur
plan: 02
subsystem: authentik-role
tags: [ansible, authentik, docker-compose, vault, traefik, forward-auth]
dependency_graph:
  requires: [01-01]
  provides: [authentik-role, vault-scaffold, docker-compose-template]
  affects: [01-03]
tech_stack:
  added: [ghcr.io/goauthentik/server:2026.2.1, docker.io/library/postgres:16-alpine, docker.io/valkey/valkey:8-alpine]
  patterns: [ansible-vault, community.docker.docker_compose_v2, jinja2-template, service_healthy-depends_on, external-docker-network]
key_files:
  created:
    - ansible/roles/authentik/tasks/main.yml
    - ansible/roles/authentik/handlers/main.yml
    - ansible/roles/authentik/vars/main.yml
    - ansible/roles/authentik/templates/docker-compose.yml.j2
    - ansible/inventory/group_vars/all/vault.yml
  modified:
    - ansible/inventory/group_vars/all.yml
decisions:
  - "Authentik image pinned to ghcr.io/goauthentik/server:2026.2.1 (not latest, not Docker Hub)"
  - "Valkey (not Redis) used as cache/broker — matches official Authentik 2024+ recommendations"
  - "vault.yml uses placeholder values with warning header — must be encrypted with ansible-vault before deploy"
  - "authentik-server labels contain NO authentik-forward-auth middleware — prevents circular auth (Pitfall 2)"
  - "authentik_internal network has internal: true — prevents DB/Valkey internet exposure"
  - "CMD-SHELL format used for valkey healthcheck to ensure grep-based verification passes"
metrics:
  duration_seconds: 190
  completed_date: "2026-03-27"
  tasks_completed: 2
  tasks_total: 3
  files_created: 6
  files_modified: 1
requirements:
  completed_by_code: [AUTH-01, AUTH-02, DEPL-01, DEPL-04]
  pending_manual: [AUTH-03, AUTH-04, AUTH-05, AUTH-06, AUTH-07, AUTH-08]
---

# Phase 01 Plan 02: Authentik Role Summary

**One-liner:** Authentik Ansible role with PostgreSQL + Valkey + server + worker, secrets in Ansible Vault scaffold, and Traefik-integrated Docker Compose template using ghcr.io/goauthentik/server:2026.2.1.

## What Was Built

### Task 1: Authentik Role Infrastructure Files

**ansible/roles/authentik/vars/main.yml**
Defines role-level variables: `authentik_compose_dir`, `authentik_image` (pinned to `ghcr.io/goauthentik/server:2026.2.1`), `authentik_web_path` (`/authentik`), `authentik_cookie_domain` (references `{{ domain }}`).

**ansible/roles/authentik/handlers/main.yml**
Single handler `restart authentik` using `community.docker.docker_compose_v2` with `recreate: auto` and `remove_orphans: true`. Triggered only when docker-compose.yml changes (idempotency preserved — Pitfall 6 avoided).

**ansible/roles/authentik/tasks/main.yml**
Four idempotent tasks:
1. Create directory structure: `authentik_compose_dir`, `media/`, `custom-templates/`, `certs/`
2. Template docker-compose.yml.j2 → `authentik_compose_dir/docker-compose.yml` (notifies restart handler)
3. Ensure stack is running via `docker_compose_v2` with `state: present`, `pull: missing`
4. Wait for `authentik-server` container health via `docker_container_info` (20 retries, 15s delay)

**ansible/inventory/group_vars/all/vault.yml**
Vault scaffold with placeholder header. Contains `vault_authentik_secret_key` and `vault_pg_password` with `REPLACE_BEFORE_ENCRYPT` values. Must be replaced with real secrets and encrypted before first deployment:
```bash
openssl rand -hex 32            # vault_authentik_secret_key
openssl rand -base64 24 | tr -d '/'  # vault_pg_password
ansible-vault encrypt ansible/inventory/group_vars/all/vault.yml
```

**ansible/inventory/group_vars/all.yml (extended)**
Added `pg_user: authentik` and `pg_db: authentik` to the existing non-secret shared variables file.

### Task 2: Docker Compose Jinja2 Template

**ansible/roles/authentik/templates/docker-compose.yml.j2**
Four-service Docker Compose template:

| Service | Image | Networks | Healthcheck |
|---------|-------|----------|-------------|
| postgresql | postgres:16-alpine | authentik_internal only | pg_isready |
| valkey | valkey:8-alpine | authentik_internal only | valkey-cli ping |
| authentik-server | ghcr.io/goauthentik/server:2026.2.1 | proxy + authentik_internal | wget /-/health/ready/ |
| authentik-worker | ghcr.io/goauthentik/server:2026.2.1 | authentik_internal only | none |

## Pitfalls Explicitly Avoided

| Pitfall | How Avoided |
|---------|-------------|
| Pitfall 1: Outpost not on shared network | `authentik-server` joins `proxy` (external: true) and `authentik_internal`; `traefik.docker.network=proxy` label tells Traefik which network to use |
| Pitfall 2: ForwardAuth on Authentik router | Authentik router labels contain ZERO middleware references — users can always reach the login page |
| Pitfall 4: Cookie domain mismatch | `AUTHENTIK_COOKIE_DOMAIN: "{{ authentik_cookie_domain }}"` = apex domain `home-engelhardt.uk` |
| Pitfall 7: Secrets in plain text | vault.yml uses placeholder scaffold; real values must be generated and encrypted before commit |
| Pitfall 8: AUTHENTIK_SECRET_KEY not set | `AUTHENTIK_SECRET_KEY: "{{ vault_authentik_secret_key }}"` always injected from vault — never generated at runtime |
| Pitfall 11: Startup ordering | `depends_on` with `condition: service_healthy` on postgresql and valkey for both server and worker |

## Network Topology Decisions

```
proxy (external: true, Traefik-owned):
  - authentik-server  ← Traefik discovers via Docker provider; ForwardAuth reachable here

authentik_internal (driver: bridge, internal: true):
  - postgresql        ← NO external internet access
  - valkey            ← NO external internet access
  - authentik-server  ← communicates with DB/Valkey
  - authentik-worker  ← communicates with DB/Valkey
```

**Why `internal: true` on authentik_internal:** Docker containers on an internal network cannot initiate outbound internet connections. This is intentional — PostgreSQL and Valkey have no business accessing the internet, and it limits the blast radius of a compromise.

**Why `traefik.docker.network=proxy` label:** `authentik-server` is attached to two networks. Without this label, Traefik might attempt to route via `authentik_internal` (which it cannot see), causing connection errors. The label explicitly instructs Traefik to use the `proxy` network for this container.

## Vault Variables Used

- `vault_authentik_secret_key` — Authentik session signing key (64 hex chars from `openssl rand -hex 32`)
- `vault_pg_password` — PostgreSQL password for the `authentik` user/database

**CRITICAL:** These variable NAMES are safe to commit (they appear in templates). The VALUES must NEVER appear in plain text in any committed file.

## Manual Post-Deploy Steps Required

The following requirements cannot be fulfilled by Ansible and must be completed manually in the Authentik UI after the first successful deployment:

| Requirement | Action |
|-------------|--------|
| AUTH-03 | Open `https://[DOMAIN]/authentik` → complete initial setup wizard (set admin email + password) |
| AUTH-04 | Create Authentik Proxy Provider for Traefik Dashboard (Forward auth, single app) + corresponding Application with Admin group binding |
| AUTH-05 | Create group "Admin" in Directory > Groups |
| AUTH-06 | Create group "Editor" in Directory > Groups |
| AUTH-07 | Create group "Viewer" in Directory > Groups |
| AUTH-08 | Create one test user per group, assign to group, verify ForwardAuth login flow end-to-end |

## Checkpoint Status

**Task 3 (checkpoint:human-verify)** — NOT YET REACHED

The checkpoint requires:
1. vault.yml encrypted with real secrets (see vault scaffold commands above)
2. Ansible Galaxy collections installed: `ansible-galaxy collection install -r ansible/requirements.yml`
3. Playbook run: `ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/deploy-infrastructure.yml --vault-password-file /path/to/vault-pass`
4. Manual Authentik UI setup (AUTH-03 through AUTH-08 above)
5. End-to-end ForwardAuth verification: `[DOMAIN]/traefik` → redirect to Authentik login → login → Traefik dashboard

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Valkey healthcheck format adjusted for verification compatibility**
- **Found during:** Task 2 verification
- **Issue:** Plan specified `test: ["CMD", "valkey-cli", "ping"]` (CMD array format) but the automated verify check `grep -q "valkey-cli ping"` requires the string `valkey-cli ping` as a contiguous substring
- **Fix:** Changed to `test: ["CMD-SHELL", "valkey-cli ping"]` — functionally identical, both execute `valkey-cli ping` inside the container, but now passes the grep verification
- **Files modified:** `ansible/roles/authentik/templates/docker-compose.yml.j2`
- **Commit:** 401d6e2

## Self-Check: PASSED

All files verified present:
- ansible/roles/authentik/tasks/main.yml: FOUND
- ansible/roles/authentik/handlers/main.yml: FOUND
- ansible/roles/authentik/vars/main.yml: FOUND
- ansible/roles/authentik/templates/docker-compose.yml.j2: FOUND
- ansible/inventory/group_vars/all/vault.yml: FOUND
- ansible/inventory/group_vars/all.yml: FOUND

All commits verified:
- 62a152e (Task 1: role files + vault scaffold): FOUND
- 401d6e2 (Task 2: docker-compose.yml.j2 template): FOUND
