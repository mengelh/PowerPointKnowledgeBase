# Stack Research

**Domain:** Infrastructure — Reverse Proxy + Identity Provider + Docker Compose + Ansible/Semaphore
**Researched:** 2026-03-27
**Confidence:** MEDIUM (training data through August 2025; external verification unavailable — versions flagged below)

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Traefik | v3.3.x (latest stable) | Reverse proxy, TLS termination, routing | Native Docker label-based auto-discovery; v3 is the current major, v2 is in maintenance mode. EntryPoint/Router/Middleware model maps cleanly to Authentik Forward Auth. |
| Authentik | 2024.12.x or 2025.x (latest stable) | OIDC/SSO, Forward Auth, group-based authz | Self-hosted, Docker-native, best-in-class Traefik Forward Auth integration. Includes embedded outpost for proxy auth without a separate agent. |
| PostgreSQL | 16.x | Authentik database | Authentik requires PostgreSQL; v16 is current LTS, well-supported by Authentik. |
| Redis | 7.x | Authentik cache/broker | Required by Authentik for task queue and session cache. |
| Docker Compose | v2 (plugin, `docker compose`) | Container orchestration | v2 is the standard; v1 (`docker-compose` binary) is deprecated and removed from Docker Desktop 4.x+. Use `services:` top-level, no `version:` key needed. |
| Ansible | 2.16.x / 2.17.x | Deployment automation | Stable, widely used. community.docker collection requires ansible-core >=2.15. |
| community.docker | 3.x (3.12.x+) | Ansible Docker Compose module | Provides `community.docker.docker_compose_v2` — the correct module for Compose v2. The older `community.docker.docker_compose` (v1) is deprecated. |
| Semaphore UI | 2.10.x+ | Ansible execution UI, Git-backed inventory | Web UI for Ansible, replaces AWX for small/home-lab setups. Runs itself as a Docker container. Supports project-scoped Git repos, secrets vault, task scheduling. |

### Supporting Libraries / Tools

| Library / Tool | Version | Purpose | When to Use |
|----------------|---------|---------|-------------|
| Traefik `crowdsec` plugin | latest | Optional brute-force / IP reputation middleware | Add in later phases when internet-facing. Not needed for Phase 1 test-VM. |
| `htpasswd` / `traefik.basicauth` middleware | built-in | Traefik dashboard fallback auth during bootstrap | Use only during initial bootstrapping before Authentik is live. Remove once Forward Auth is active. |
| Ansible `ansible-vault` | built-in | Encrypt secrets in playbook vars | Use for all credentials committed to Git (DB passwords, Authentik bootstrap secret). |
| `python3-docker` (pip) | latest | Required on the managed host for community.docker | Must be installed on the target VM; community.docker modules call the Docker SDK under the hood. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `docker compose config` | Validate Compose file syntax before deployment | Run locally or in Ansible `command:` task as a pre-check. |
| `ansible-lint` | Lint Ansible playbooks | Catches deprecated module names (`docker_compose` → `docker_compose_v2`), YAML style issues. |
| `yamllint` | Lint YAML files | Keep Compose files clean; avoids Ansible parse errors. |
| Semaphore CLI (`semaphore`) | Trigger runs via API | Useful for CI hooks later; not needed for Phase 1 manual deployments. |

---

## Key Configuration Patterns

### Traefik v3 — Docker Compose Labels (Canonical Pattern)

```yaml
# docker-compose.yml — Traefik service
services:
  traefik:
    image: traefik:v3.3
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      # Let's Encrypt (use for internet-facing; swap for internal CA on private test-VM)
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "letsencrypt:/letsencrypt"
    labels:
      - "traefik.enable=true"
      # Dashboard router — protected by Authentik Forward Auth middleware
      - "traefik.http.routers.traefik-dashboard.rule=PathPrefix(`/traefik`)"
      - "traefik.http.routers.traefik-dashboard.entrypoints=websecure"
      - "traefik.http.routers.traefik-dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.traefik-dashboard.service=api@internal"
      - "traefik.http.routers.traefik-dashboard.middlewares=authentik-auth@docker"
```

**Why this matters for Phase 1:**
- `exposedbydefault=false` is critical — prevents accidental exposure of services that haven't opted in with `traefik.enable=true`.
- The dashboard uses `api@internal` as its service (built-in); no `traefik.http.services.X.loadbalancer` label needed for it.
- For a private test-VM without a public IP, swap `httpchallenge` for `tlschallenge` (DNS-01) or use a self-signed cert with `tls: {}` and no resolver.

**TLS for private/test VM (no Let's Encrypt):**
```yaml
# In Traefik static config or as CLI flags:
# No certresolver → Traefik generates a self-signed cert automatically
- "traefik.http.routers.traefik-dashboard.tls=true"
# Or provide a custom cert via dynamic config file provider
```

### Authentik Forward Auth Middleware (Traefik Label)

```yaml
# On any service that should be protected by Authentik:
labels:
  - "traefik.http.middlewares.authentik-auth.forwardauth.address=http://authentik-server:9000/outpost.goauthentik.io/auth/traefik"
  - "traefik.http.middlewares.authentik-auth.forwardauth.trustForwardHeader=true"
  - "traefik.http.middlewares.authentik-auth.forwardauth.authResponseHeaders=X-authentik-username,X-authentik-groups,X-authentik-email,X-authentik-name,X-authentik-uid,X-authentik-jwt,X-authentik-meta-jwks,X-authentik-meta-outpost,X-authentik-meta-provider,X-authentik-meta-app,X-authentik-meta-version"
```

**Critical detail:** The `forwardauth.address` must point to the embedded outpost inside the Authentik `server` container on port 9000. This is Authentik's built-in proxy outpost — no separate outpost deployment needed for simple Forward Auth setups.

### Authentik — Docker Compose Structure

```yaml
services:
  postgresql:
    image: docker.io/library/postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${PG_PASS}
      POSTGRES_USER: authentik
      POSTGRES_DB: authentik
    volumes:
      - database:/var/lib/postgresql/data

  redis:
    image: docker.io/library/redis:7-alpine
    restart: unless-stopped
    command: --save 60 1 --loglevel warning
    volumes:
      - redis:/data

  authentik-server:
    image: ghcr.io/goauthentik/server:2024.12.3
    restart: unless-stopped
    command: server
    environment: &authentik-env
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
    volumes:
      - ./media:/media
      - ./custom-templates:/templates
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.authentik.rule=PathPrefix(`/authentik`)"
      - "traefik.http.routers.authentik.entrypoints=websecure"
      - "traefik.http.routers.authentik.tls=true"
      - "traefik.http.services.authentik.loadbalancer.server.port=9000"

  authentik-worker:
    image: ghcr.io/goauthentik/server:2024.12.3
    restart: unless-stopped
    command: worker
    environment: *authentik-env
    volumes:
      - ./media:/media
      - ./certs:/certs
      - ./custom-templates:/templates
```

**Why `ghcr.io/goauthentik/server` not Docker Hub:** Authentik's official images are on GitHub Container Registry. Docker Hub images may exist as mirrors but the canonical source is `ghcr.io/goauthentik/server`.

**Why same image for server and worker:** Authentik packages server and worker in one image; the `command:` argument (`server` vs `worker`) determines the role. This simplifies upgrades — one image tag to update.

### Ansible Playbook Pattern — community.docker

```yaml
# requirements.yml
collections:
  - name: community.docker
    version: ">=3.12.0"

# tasks/deploy_compose.yml
- name: Deploy Docker Compose stack
  community.docker.docker_compose_v2:
    project_src: /opt/kb-pipeline
    state: present
    pull: always          # Always pull latest images matching tag
    remove_orphans: true  # Clean up removed services
  register: compose_result

- name: Show compose changes
  ansible.builtin.debug:
    var: compose_result
  when: compose_result.changed
```

**Critical: Use `docker_compose_v2`, not `docker_compose`.** The `community.docker.docker_compose` module wraps the deprecated Python `docker-compose` v1 binary. `docker_compose_v2` wraps the Docker CLI plugin (`docker compose`) — which is what all modern systems use.

**Prerequisites on managed host (add to a `setup` role):**
```yaml
- name: Install Docker and prerequisites
  ansible.builtin.package:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin   # provides 'docker compose' subcommand
      - python3-docker          # required by community.docker modules
    state: present
```

### Semaphore Integration Pattern

Semaphore organizes work into: **Projects → Repositories → Inventories → Environment (vars) → Task Templates → Tasks**.

For Phase 1 deployment:

1. **Repository:** Point to the Git repo containing Ansible playbooks (`/ansible/` directory).
2. **Inventory:** Static inventory with the test-VM IP/hostname; or a simple `hosts.ini` committed to the repo.
3. **Environment:** Store sensitive vars (pulled from Semaphore's built-in vault), injecting `PG_PASS`, `AUTHENTIK_SECRET_KEY` as extra-vars or via `ANSIBLE_VAULT_PASSWORD_FILE`.
4. **Task Template:** Reference the playbook file path in the repo (e.g., `playbooks/deploy-infra.yml`).

**Semaphore does not replace Ansible Vault** — store secrets in Semaphore's "Key Store" (type: `secret`) and reference them as extra-vars. For playbook-level vault encryption, Semaphore can inject the vault password from its Key Store.

---

## Installation

```bash
# On the Ansible control node (local machine or CI runner):
pip install ansible ansible-lint

# Install community.docker collection:
ansible-galaxy collection install community.docker

# On the managed host (handled by Ansible setup role):
# docker-ce, docker-compose-plugin, python3-docker
# (see package task above)
```

```bash
# Semaphore — run as Docker container on a separate management host or locally:
docker run -d \
  --name semaphore \
  -e SEMAPHORE_DB_DIALECT=bolt \
  -e SEMAPHORE_ADMIN_PASSWORD=changeme \
  -e SEMAPHORE_ADMIN_NAME=admin \
  -e SEMAPHORE_ADMIN_EMAIL=admin@example.com \
  -p 3000:3000 \
  semaphoreui/semaphore:latest
```

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Traefik v3 | Nginx Proxy Manager | If the team is unfamiliar with YAML labels and prefers a GUI for routing config. NPM lacks the programmatic label-driven auto-discovery that makes Ansible deployments clean. |
| Traefik v3 | Caddy | If automatic HTTPS with zero-config is the only requirement and no middleware (Forward Auth) is needed. Caddy's Forward Auth support exists but is less battle-tested with Authentik than Traefik's. |
| Authentik | Keycloak | If enterprise SAML federation or Java-ecosystem familiarity is required. Keycloak is heavier (JVM, more config surface) and its Docker Compose setup is more complex. |
| Authentik | Authelia | If you only need 2FA/MFA on top of existing LDAP. Authelia has no user management UI; Authentik is better for managing users/groups programmatically as we need in Phase 1. |
| community.docker.docker_compose_v2 | `ansible.builtin.command: docker compose up` | Shell commands are acceptable for one-off tasks but lack idempotency checks, status reporting, and structured output. Always prefer the module. |
| Semaphore UI | AWX / Automation Controller | If you need RBAC-heavy enterprise features, LDAP-backed logins for the Ansible UI, or a large team. AWX has higher operational overhead (Kubernetes-native). |
| PostgreSQL 16 | PostgreSQL 15 | 15 works fine; 16 adds logical replication improvements useful if we add replicas later. No functional difference for Phase 1. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `community.docker.docker_compose` (v1 module) | Wraps the deprecated Python `docker-compose` binary which is EOL. Not compatible with modern `docker compose` plugin installations. | `community.docker.docker_compose_v2` |
| `version: "3.9"` at top of Compose file | Docker Compose v2 ignores the `version:` key (it's obsolete); including it causes a deprecation warning and confuses readers about which spec is active. | Omit the `version:` key entirely — Compose v2 uses the current schema by default. |
| Traefik v2 | Traefik v2 enters maintenance mode; v3 is the active development line. v3 introduced breaking changes in middleware naming and CLI flag syntax — start with v3 to avoid a mandatory migration. | `traefik:v3.3` |
| Docker Hub `authentik` image (unofficial) | Several Docker Hub images are community mirrors, not officially maintained. Image provenance cannot be verified. | `ghcr.io/goauthentik/server:YYYY.MM.PATCH` |
| Exposing Docker socket without `ro` mount | Mounting `/var/run/docker.sock` without `:ro` gives Traefik write access to the Docker daemon — potential container escape. Traefik only needs read access for label discovery. | `- /var/run/docker.sock:/var/run/docker.sock:ro` |
| Pinning Authentik to `latest` tag | `latest` will auto-upgrade on `docker compose pull`, breaking Authentik which requires explicit database migrations between versions. | Pin to a specific `YYYY.MM.PATCH` tag and upgrade intentionally. |
| Storing secrets in Compose `.env` committed to Git | Leaks credentials into version history. | Use `ansible-vault` encrypted vars files; inject at deploy time via Semaphore environment. |

---

## Stack Patterns by Variant

**If the test-VM has no public internet exposure (private LAN only):**
- Use Traefik with `tls: {}` (no resolver) — Traefik generates a self-signed cert automatically.
- Or provision a wildcard cert from an internal CA and mount it as a Traefik file provider dynamic config.
- Do NOT use Let's Encrypt ACME with `httpChallenge` — it requires port 80 to be reachable from the internet.
- Consider `tlsChallenge` (DNS-01) with a supported DNS provider if the domain has public DNS even on a private IP.

**If forward auth needs to protect non-HTTP services (later phases):**
- Authentik also supports TCP/port-level proxying via its outpost, but this is not needed for Phase 1.
- Stay with HTTP Forward Auth for Phase 1.

**If multiple Docker Compose stacks are deployed (Phase 1 infra + Phase 2 app):**
- Use a shared Docker network (`external: true`) so that app services can reach Authentik's outpost at `http://authentik-server:9000`.
- Define the shared network in the infra stack; reference it as `external` in the app stack.

```yaml
# infra/docker-compose.yml
networks:
  proxy:
    name: proxy
    driver: bridge

# app/docker-compose.yml
networks:
  proxy:
    external: true
    name: proxy
```

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| Traefik v3.3 | Docker Compose v2 | Labels use `traefik.http.*` — same namespace as v2; middleware naming changed in v3 (no `@docker` suffix needed on middleware definitions, but required on middleware _references_). |
| Authentik 2024.12.x | PostgreSQL 16, Redis 7 | Authentik's Docker Compose example in docs uses these exact versions. |
| community.docker 3.x | ansible-core 2.15+ | The `docker_compose_v2` module was introduced in community.docker 3.6.0. Requires the `docker compose` CLI plugin (not the Python `docker-compose` package) on the managed host. |
| Semaphore 2.10.x | Ansible 2.16.x | Semaphore executes Ansible via subprocess; any Ansible version installed in the Semaphore container or on the runner host is supported. |
| Docker Compose v2 | Docker Engine 23.x+ | Compose v2 is bundled as `docker-compose-plugin` package in Docker CE repos. Verify `docker compose version` returns `v2.x.x`. |

---

## Sources

- Training data (Traefik v3 docs, Authentik docs, Ansible collection docs) — coverage through August 2025. **MEDIUM confidence on versions** — verify current patch versions at:
  - https://github.com/traefik/traefik/releases
  - https://github.com/goauthentik/authentik/releases
  - https://github.com/ansible-collections/community.docker/blob/main/CHANGELOG.rst
  - https://github.com/semaphoreui/semaphore/releases
- External verification unavailable at research time (WebSearch/WebFetch disabled).
- Architecture patterns for Authentik + Traefik Forward Auth are well-established community standards; HIGH confidence on pattern correctness.

---

*Stack research for: KB-Pipeline Phase 1 — Traefik + Authentik infrastructure on Docker Compose / Ansible / Semaphore*
*Researched: 2026-03-27*
