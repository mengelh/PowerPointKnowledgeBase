---
phase: 01-infrastruktur
plan: 01
subsystem: infra
tags: [ansible, traefik, docker, docker-compose, jinja2, reverse-proxy, tls, prometheus, forwardauth]

# Dependency graph
requires: []
provides:
  - Ansible project skeleton with requirements.yml (community.docker >= 3.12.0)
  - Docker role: installs Docker CE, Compose plugin, python3-docker with idempotent guards
  - Traefik role: proxy network creation, all Jinja2 templates, container deployment via docker_compose_v2
  - Traefik static config: HTTPS entrypoints, HTTP-to-HTTPS redirect, JSON access logs, Prometheus on :8082
  - Traefik dynamic config: self-signed TLS options, authentik-forward-auth middleware, protected dashboard router
affects:
  - 01-02 (Authentik role joins proxy network, uses authentik-forward-auth middleware defined here)
  - 01-03 (Semaphore deployment uses these playbooks and inventory)

# Tech tracking
tech-stack:
  added:
    - traefik:v3.3
    - community.docker >= 3.12.0 (provides docker_compose_v2, docker_network modules)
    - Ansible roles pattern: docker, traefik (basis for authentik in plan 02)
  patterns:
    - File provider for Traefik dynamic config (NOT Docker labels) — enables non-container middleware definitions
    - Proxy network created by Traefik role, referenced as external by all future service stacks
    - community.docker.docker_compose_v2 with pull: missing for pinned-tag deployments
    - Handler-based Traefik restart (notify: restart traefik) — only restarts on config change
    - creates: guards on shell tasks for idempotent Docker repository setup

key-files:
  created:
    - ansible/requirements.yml
    - ansible/inventory/hosts.yml
    - ansible/inventory/group_vars/all.yml
    - ansible/playbooks/deploy-infrastructure.yml
    - ansible/roles/docker/tasks/main.yml
    - ansible/roles/traefik/tasks/main.yml
    - ansible/roles/traefik/handlers/main.yml
    - ansible/roles/traefik/vars/main.yml
    - ansible/roles/traefik/templates/docker-compose.yml.j2
    - ansible/roles/traefik/templates/traefik.yml.j2
    - ansible/roles/traefik/templates/middlewares.yml.j2
    - ansible/roles/traefik/templates/dashboard.yml.j2
    - ansible/roles/traefik/templates/tls.yml.j2
  modified: []

key-decisions:
  - "Self-signed TLS: tls:{} with no certresolver — Traefik auto-generates cert; no ACME/Let's Encrypt needed for private test-VM"
  - "ForwardAuth middleware in file provider (middlewares.yml.j2), NOT Docker labels — allows non-container auth enforcement"
  - "ForwardAuth address uses internal HTTP (http://authentik-server:9000) not HTTPS — avoids redirect loop (Pitfall 2)"
  - "proxy network created by Traefik role before docker_compose_v2 — all future stacks join as external network"
  - "pull: missing strategy — prevents image churn on re-runs; supports pinned tag upgrades"

patterns-established:
  - "Pattern 1: Traefik dynamic config via /config/dynamic directory with file provider watch:true — add new .yml files to add routes/middleware without restart"
  - "Pattern 2: Proxy network ownership — Traefik role owns network creation; all other compose stacks reference it as external"
  - "Pattern 3: Non-secret vars in group_vars/all.yml; secrets go to group_vars/all/vault.yml (encrypted)"
  - "Pattern 4: Handler restart uses recreate:auto — only recreates containers when config hash changes"

requirements-completed: [TRFK-01, TRFK-02, TRFK-03, TRFK-04, TRFK-05, TRFK-06, DEPL-01, DEPL-02]

# Metrics
duration: 2min
completed: 2026-03-27
---

# Phase 01 Plan 01: Ansible project skeleton + Traefik role with self-signed TLS, file-provider ForwardAuth, and Prometheus metrics

**Traefik v3.3 deployed via Ansible community.docker.docker_compose_v2 with self-signed TLS, JSON access logs, Prometheus on :8082, and authentik-forward-auth middleware in dynamic file provider protecting the dashboard**

## Performance

- **Duration:** ~2 min
- **Started:** 2026-03-27T11:08:56Z
- **Completed:** 2026-03-27T11:11:36Z
- **Tasks:** 2
- **Files modified:** 13

## Accomplishments

- Ansible project skeleton with community.docker collection requirement and inventory with KB_VM_HOST env override
- Docker role with idempotent GPG key + repository setup (creates: guards) and Docker CE + Compose plugin + python3-docker installation
- Traefik role with all Jinja2 templates implementing TRFK-01 through TRFK-06: exposedByDefault:false, self-signed TLS, HTTP redirect, JSON logs, Prometheus, protected dashboard

## Task Commits

Each task was committed atomically:

1. **Task 1: Ansible project skeleton — docker role and top-level playbook** - `d45797b` (feat)
2. **Task 2: Traefik role — tasks, handlers, vars, and all Jinja2 templates** - `4864791` (feat)

## Files Created/Modified

- `ansible/requirements.yml` - Declares community.docker >= 3.12.0 collection dependency
- `ansible/inventory/hosts.yml` - Static inventory with KB_VM_HOST env override for Semaphore
- `ansible/inventory/group_vars/all.yml` - Non-secret vars: domain, deploy_base_dir, traefik_image, traefik_log_level
- `ansible/playbooks/deploy-infrastructure.yml` - Top-level playbook: docker -> traefik -> authentik roles
- `ansible/roles/docker/tasks/main.yml` - Docker CE installation with idempotent shell guards and systemd enable
- `ansible/roles/traefik/vars/main.yml` - Derived paths: config_dir, dynamic_dir, compose_dir
- `ansible/roles/traefik/handlers/main.yml` - Handler restart via docker_compose_v2 recreate:auto
- `ansible/roles/traefik/tasks/main.yml` - Dir creation, proxy network, template tasks, compose deployment
- `ansible/roles/traefik/templates/docker-compose.yml.j2` - Traefik service: ports 80/443, docker.sock:ro, healthcheck --ping
- `ansible/roles/traefik/templates/traefik.yml.j2` - Static config: entrypoints, file provider, JSON accessLog, Prometheus
- `ansible/roles/traefik/templates/middlewares.yml.j2` - authentik-forward-auth forwardAuth with 11 authResponseHeaders
- `ansible/roles/traefik/templates/dashboard.yml.j2` - Dashboard router: PathPrefix(/traefik), tls:{}, authentik-forward-auth@file
- `ansible/roles/traefik/templates/tls.yml.j2` - TLS options: minVersion VersionTLS12, modern cipher suites

## Decisions Made

- Self-signed TLS implemented via `tls: {}` with no certresolver — Traefik auto-generates certificate when tls is enabled without a resolver. Chosen for private test-VM (no public internet exposure, ACME not viable).
- ForwardAuth middleware defined in Traefik file provider (dynamic/middlewares.yml), not Docker labels. This allows the middleware to exist before Authentik containers are running, prevents chicken-and-egg issues on first deploy.
- ForwardAuth address deliberately uses `http://authentik-server:9000` (not https) to avoid TLS redirect loop (PITFALLS.md Pitfall 2) — Traefik would redirect HTTP->HTTPS and break the auth check.
- proxy network created before docker_compose_v2 via community.docker.docker_network — guarantees network exists before Traefik tries to attach to it.
- pull: missing in docker_compose_v2 task — only pulls image if not present locally. Prevents unwanted upgrades when a pinned tag (traefik:v3.3) already exists.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## What Was NOT Built in This Plan

- Authentik containers (PostgreSQL, Redis, authentik-server, authentik-worker) — that is Plan 02
- Authentik groups (Admin, Editor, Viewer) and test users — that is Plan 02/03
- Semaphore project configuration — that is Plan 03
- Any application-layer services (FastAPI, Angular, etc.) — those are later phases

## Pitfalls Explicitly Avoided

- **Pitfall 1 (exposedByDefault: true):** Set `exposedByDefault: false` in traefik.yml.j2 providers.docker block
- **Pitfall 2 (HTTPS redirect loop on forwardAuth):** ForwardAuth address is `http://authentik-server:9000` (internal HTTP, not HTTPS)
- **Pitfall 3 (Missing authResponseHeaders):** All 11 X-authentik-* headers listed in middlewares.yml.j2
- **Pitfall 6 (docker_compose v1 module):** Uses community.docker.docker_compose_v2 throughout (tasks AND handlers)
- **Pitfall 10 (Secrets in plain text):** No passwords/keys in any committed file; group_vars pattern established for vault.yml in Plan 02

## Version Pins Used

- `traefik:v3.3` — pinned in group_vars/all.yml as traefik_image variable
- `community.docker >= 3.12.0` — minimum version ensuring docker_compose_v2 module availability

## Next Phase Readiness

- Plan 02 (Authentik role) can be built: proxy network exists, authentik-forward-auth middleware is defined in file provider
- Plan 02 must create `ansible/roles/authentik/` with its own docker-compose.yml.j2 referencing the proxy network as external
- The dashboard will return 500 on forwardAuth until Authentik server is live — this is expected and documented in plan objective
- Plan 03 (Semaphore) can reference `ansible/playbooks/deploy-infrastructure.yml` and `ansible/inventory/hosts.yml`

---
*Phase: 01-infrastruktur*
*Completed: 2026-03-27*

## Self-Check: PASSED

All 13 plan files exist. Both task commits (d45797b, 4864791) found in git history.
