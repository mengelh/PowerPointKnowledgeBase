# Architecture Research

**Domain:** Reverse Proxy + Identity Provider infrastructure (Traefik + Authentik on Docker Compose, managed by Ansible)
**Researched:** 2026-03-27
**Confidence:** HIGH (Traefik v3, Authentik 2024.x patterns well-established; no web access available to verify latest patch versions)

---

## Standard Architecture

### System Overview

```
                        Internet / Client Browser
                                   |
                             [ HTTPS :443 ]
                                   |
                    ┌──────────────▼──────────────┐
                    │         Traefik v3           │
                    │   (Reverse Proxy + TLS)      │
                    │                              │
                    │  EntryPoints: :80, :443      │
                    │  Providers:   Docker, File   │
                    │  ACME:        Let's Encrypt  │
                    └──┬──────────────┬────────────┘
                       │              │
         ┌─────────────▼──┐    ┌──────▼───────────────┐
         │  Authentik     │    │  Future App Services  │
         │  (server)      │    │  (FastAPI, Angular)   │
         │  :9000 / :9443 │    └──────────────────────┘
         └───────┬────────┘
                 │ depends-on
         ┌───────┴────────────────────┐
         │                            │
   ┌─────▼──────┐            ┌────────▼───────┐
   │ PostgreSQL │            │     Redis      │
   │  (authdb)  │            │  (authredis)   │
   └────────────┘            └────────────────┘
         │                            │
         └────────────┬───────────────┘
                      │ (same dependencies)
              ┌───────▼────────┐
              │ Authentik      │
              │ Worker         │
              │ (background    │
              │  tasks)        │
              └────────────────┘
```

**Forward Auth flow (request-time auth check):**

```
Client → Traefik → ForwardAuth middleware → Authentik (server :9000/outpost.goauthentik.io/…)
                                                │
                              401 Unauthorized  │  200 OK
                                    │           │
                             Redirect to        │
                             Authentik login    │
                                            ┌───▼───────────────┐
                                            │  Upstream Service  │
                                            │  (e.g. Traefik     │
                                            │   Dashboard)       │
                                            └───────────────────┘
```

---

## Component Responsibilities

| Component | Responsibility | Communicates With |
|-----------|----------------|-------------------|
| Traefik | TLS termination, HTTP routing, middleware (ForwardAuth, headers, ratelimit) | All backend services via Docker labels; Authentik for auth checks |
| Authentik Server | UI, OAuth2/OIDC provider, ForwardAuth endpoint, admin API | PostgreSQL, Redis, Authentik Worker |
| Authentik Worker | Background jobs: event logs, policy evaluation, certificate rotation, email | PostgreSQL, Redis |
| PostgreSQL (authdb) | Persistent storage for Authentik (users, groups, flows, policies) | Authentik Server, Authentik Worker |
| Redis (authredis) | Session cache, message broker between server and worker | Authentik Server, Authentik Worker |

---

## Docker Compose File Structure

### Recommended Split: Two Compose Files

**Rationale:** Traefik is infrastructure shared across all future services. Authentik is one application on top of that infrastructure. Separating them follows the principle of independent lifecycle management — Authentik can be restarted/upgraded without touching Traefik config.

```
/opt/kb-pipeline/
├── traefik/
│   ├── docker-compose.yml          # Traefik service only
│   └── config/
│       ├── traefik.yml             # Static config (entrypoints, providers, ACME)
│       └── dynamic/
│           └── tls.yml             # Dynamic TLS options (ciphersuites, min version)
│
└── authentik/
    └── docker-compose.yml          # authentik-server, authentik-worker, postgresql, redis
```

**Alternative (single file):** Acceptable for Phase 1 when there are only two top-level concerns. Prefer split files for Phase 2+ when app services are added.

### Shared External Network

Both compose files must share one external Docker network (`proxy`) so Traefik can reach Authentik containers via Docker service discovery:

```yaml
# traefik/docker-compose.yml
networks:
  proxy:
    name: proxy
    driver: bridge

# authentik/docker-compose.yml
networks:
  proxy:
    external: true        # Traefik-owned network
  authentik_internal:
    driver: bridge        # Internal-only: server <-> worker <-> db <-> redis
```

---

## Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│  Docker Network: proxy  (external, traefik-owned)           │
│                                                             │
│  ┌─────────┐        ┌──────────────────┐                   │
│  │ Traefik │◄──────►│ authentik-server │                   │
│  └─────────┘        └──────────────────┘                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Docker Network: authentik_internal  (compose-internal)     │
│                                                             │
│  ┌──────────────────┐   ┌──────────────────┐               │
│  │ authentik-server │   │ authentik-worker  │               │
│  └────────┬─────────┘   └────────┬──────────┘               │
│           │                      │                          │
│     ┌─────▼──────┐    ┌──────────▼─────┐                   │
│     │ postgresql  │    │     redis      │                   │
│     └────────────┘    └────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

**Isolation rule:** PostgreSQL and Redis are NOT on the `proxy` network. Only `authentik-server` (and `authentik-worker`) need to reach the outside-facing network. Database containers have zero exposure.

---

## Traefik: Static vs Dynamic Configuration

### Static Configuration (`traefik.yml`)

Set once at startup. Restart required to change.

```yaml
# traefik/config/traefik.yml
global:
  checkNewVersion: false
  sendAnonymousUsage: false

api:
  dashboard: true          # Enabled; access controlled via ForwardAuth
  insecure: false          # Never expose dashboard without auth

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: "admin@example.com"
      storage: "/certificates/acme.json"
      tlsChallenge: {}      # or httpChallenge if port 80 is reachable

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false    # CRITICAL: services must opt-in via labels
    network: proxy             # Default network for service discovery
  file:
    directory: "/config/dynamic"
    watch: true                # Hot-reload dynamic config files

log:
  level: INFO

accessLog: {}
```

### Dynamic Configuration (`config/dynamic/*.yml`)

Hot-reloaded without restart. Use for:
- TLS options (cipher suites, minimum version)
- Middleware definitions that are shared across services (e.g., the ForwardAuth middleware itself, security headers)
- Routers/services for non-Docker targets

```yaml
# traefik/config/dynamic/middlewares.yml
http:
  middlewares:
    authentik-forward-auth:
      forwardAuth:
        address: "http://authentik-server:9000/outpost.goauthentik.io/auth/traefik"
        trustForwardHeader: true
        authResponseHeaders:
          - X-authentik-username
          - X-authentik-groups
          - X-authentik-email
          - X-authentik-name
          - X-authentik-uid
          - X-authentik-jwt
          - X-authentik-meta-jwks
          - X-authentik-meta-outpost
          - X-authentik-meta-provider
          - X-authentik-meta-app
          - X-authentik-meta-version

    security-headers:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        contentTypeNosniff: true
        browserXssFilter: true
        referrerPolicy: "strict-origin-when-cross-origin"
        frameDeny: true
```

**Decision rule:** If a config value requires Traefik restart to take effect → static. If it can be updated on-the-fly → dynamic. Middleware definitions belong in dynamic config because they can be modified without downtime.

---

## Authentik Outpost Configuration (Forward Auth)

Authentik uses the concept of an **Outpost** — a sub-service (embedded or standalone) that handles the actual ForwardAuth requests from Traefik. For Phase 1, the **embedded outpost** inside `authentik-server` is sufficient.

### How It Works

1. Traefik receives request for a protected route
2. Traefik's ForwardAuth middleware POSTs to `http://authentik-server:9000/outpost.goauthentik.io/auth/traefik`
3. Authentik checks session cookie / token
4. If valid: returns 200 + user headers → Traefik forwards to upstream
5. If invalid: returns 401 → Traefik redirects to Authentik login flow

### Authentik Configuration in Docker Labels

```yaml
# Applied to any service that needs protection
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.my-service.rule=Host(`example.com`) && PathPrefix(`/my-service`)"
  - "traefik.http.routers.my-service.entrypoints=websecure"
  - "traefik.http.routers.my-service.tls.certresolver=letsencrypt"
  - "traefik.http.routers.my-service.middlewares=authentik-forward-auth@file"
```

### Traefik Dashboard Protection (Phase 1 Target)

The Traefik API/dashboard is exposed via a special router in dynamic config (not Docker labels, since Traefik itself is not a container tracked by its own Docker provider):

```yaml
# traefik/config/dynamic/dashboard.yml
http:
  routers:
    traefik-dashboard:
      rule: "PathPrefix(`/traefik`)"
      service: api@internal
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt
      middlewares:
        - authentik-forward-auth
```

### Authentik Self-Route (no ForwardAuth on Authentik itself)

Authentik must be reachable without ForwardAuth — otherwise users can never log in:

```yaml
# In authentik/docker-compose.yml labels on authentik-server:
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.authentik.rule=PathPrefix(`/authentik`)"   # or Host()+PathPrefix
  - "traefik.http.routers.authentik.entrypoints=websecure"
  - "traefik.http.routers.authentik.tls.certresolver=letsencrypt"
  # NO authentik-forward-auth middleware here
  - "traefik.http.services.authentik.loadbalancer.server.port=9000"
```

---

## Ansible Role / Directory Structure

### Recommended Layout

```
ansible/
├── inventory/
│   ├── hosts.yml                   # Target VM(s) with connection params
│   └── group_vars/
│       ├── all.yml                 # Non-secret shared vars
│       └── all/
│           └── vault.yml           # ansible-vault encrypted secrets
│
├── playbooks/
│   └── deploy-infrastructure.yml  # Top-level playbook for Phase 1
│
└── roles/
    ├── docker/                     # Ensure Docker + Docker Compose installed
    │   └── tasks/main.yml
    │
    ├── traefik/
    │   ├── tasks/
    │   │   └── main.yml
    │   ├── templates/
    │   │   ├── docker-compose.yml.j2
    │   │   ├── traefik.yml.j2      # Static config template
    │   │   └── middlewares.yml.j2  # Dynamic config template
    │   ├── vars/
    │   │   └── main.yml            # Role-level defaults
    │   └── handlers/
    │       └── main.yml            # restart traefik
    │
    └── authentik/
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   ├── docker-compose.yml.j2
        │   └── env.j2              # Authentik .env file
        ├── vars/
        │   └── main.yml
        └── handlers/
            └── main.yml            # restart authentik-server, authentik-worker
```

### Variable / Secret Handling

| Variable Type | Location | Example |
|---------------|----------|---------|
| Structural (non-secret) | `group_vars/all.yml` | `domain: kb.example.com`, `deploy_base_dir: /opt/kb-pipeline` |
| Infrastructure config | `roles/traefik/vars/main.yml` | `traefik_log_level: INFO`, `acme_email: admin@example.com` |
| Secrets | `group_vars/all/vault.yml` (ansible-vault) | `authentik_secret_key`, `pg_password`, `pg_user` |

**Vault strategy:** Use `ansible-vault encrypt_string` for individual values, or a single `vault.yml` file encrypted with `ansible-vault encrypt`. Semaphore has native vault password support — store the vault password in Semaphore's key store, not in the repo.

### Idempotency Patterns in Tasks

```yaml
# roles/traefik/tasks/main.yml (conceptual)
- name: Create traefik config directory
  ansible.builtin.file:
    path: "{{ deploy_base_dir }}/traefik/config/dynamic"
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Template static config
  ansible.builtin.template:
    src: traefik.yml.j2
    dest: "{{ deploy_base_dir }}/traefik/config/traefik.yml"
    owner: root
    mode: '0644'
  notify: restart traefik

- name: Deploy docker-compose for traefik
  ansible.builtin.template:
    src: docker-compose.yml.j2
    dest: "{{ deploy_base_dir }}/traefik/docker-compose.yml"
    owner: root
    mode: '0644'
  notify: restart traefik

- name: Ensure traefik is running
  community.docker.docker_compose_v2:
    project_src: "{{ deploy_base_dir }}/traefik"
    state: present
```

**Key module:** `community.docker.docker_compose_v2` (for Docker Compose v2 / the `docker compose` plugin). The older `docker_compose` module targets v1 (`docker-compose`). Verify which is installed on the VM.

---

## TLS Certificate Strategy

### Recommended: Let's Encrypt ACME (TLS Challenge)

For a publicly reachable domain (like `semaphore.home-engelhardt.uk` suggests this VM is internet-accessible):

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      email: "{{ acme_email }}"
      storage: "/certificates/acme.json"
      tlsChallenge: {}
```

Mount `acme.json` as a Docker volume. Traefik writes certificates there and reloads automatically.

### Alternative: HTTP-01 Challenge

If port 443 has issues but 80 is open:

```yaml
      httpChallenge:
        entryPoint: web
```

### Alternative: DNS-01 Challenge (Wildcard certs)

If the VM is not directly internet-accessible (behind NAT) but DNS is managed by a supported provider. Requires DNS provider API credentials in the static config.

### Internal / Self-Signed (avoid for Phase 1)

Only warranted if the VM truly has no public DNS resolution. Creates browser trust issues and requires CA distribution. Avoid unless necessary.

**Recommendation:** Use TLS Challenge (tlsChallenge) — requires only that port 443 is reachable from the ACME server (Let's Encrypt). No port 80 needed. Simpler than DNS challenge for a single domain.

---

## Data Flow

### Authentication Request Flow

```
Browser (GET /traefik/dashboard)
  │
  ▼
Traefik EntryPoint :443
  │ matches router rule: PathPrefix(`/traefik`)
  ▼
Middleware: authentik-forward-auth
  │ ForwardAuth POST → http://authentik-server:9000/outpost.goauthentik.io/auth/traefik
  │
  ├─ [No session / invalid token]
  │    401 → Traefik redirects → https://<domain>/authentik/flows/default-authentication/
  │               Browser logs in via Authentik UI
  │               Authentik sets session cookie
  │               Redirect back to /traefik/dashboard
  │
  └─ [Valid session]
       200 + headers (X-authentik-username, X-authentik-groups, …)
         │
         ▼
       Traefik forwards request to: api@internal (Traefik dashboard)
         │
         ▼
       Response → Browser
```

### Ansible Deployment Flow

```
Semaphore UI triggers playbook
  │
  ▼
Ansible connects to VM via SSH
  │
  ├─ Role: docker  → ensures Docker Engine + Compose plugin installed
  │
  ├─ Role: traefik
  │   ├─ Creates directories
  │   ├─ Templates config files (traefik.yml, middlewares.yml)
  │   ├─ Templates docker-compose.yml
  │   ├─ Creates proxy network (if not exists)
  │   └─ docker compose up -d (idempotent)
  │
  └─ Role: authentik
      ├─ Creates directories
      ├─ Templates .env (from vault vars)
      ├─ Templates docker-compose.yml
      └─ docker compose up -d
```

---

## Build Order (Phase Dependencies)

```
1. Docker role          ← prerequisite for everything
       │
       ▼
2. Traefik role         ← creates proxy network; must exist before authentik
       │                   attaches to it
       ▼
3. Authentik role       ← attaches authentik-server to proxy network;
                           configures ForwardAuth middleware
```

**Why this order:**
- The `proxy` external network must exist before any service tries to join it
- Traefik's dynamic config references `authentik-server` by hostname — Traefik must be up and watching for the container to appear
- Authentik's Docker labels are read by Traefik via the Docker provider automatically once the container starts

---

## Architectural Patterns

### Pattern 1: Labels as Configuration (Docker Provider)

**What:** Traefik reads router/middleware/service config directly from Docker container labels. No separate config file per service.
**When to use:** All services running as Docker containers.
**Trade-off:** Config lives next to the service (good for locality), but requires Traefik to have Docker socket access (security consideration — use socket proxy in production).

### Pattern 2: File Provider for Non-Container Targets

**What:** Traefik's `file` provider reads YAML/TOML from a directory. Used for the API dashboard router (since Traefik itself is not a Docker container in its own label space) and for shared middleware definitions.
**When to use:** When the target is `api@internal`, or for middleware shared across many services.
**Trade-off:** Decoupled from individual services, but requires SSH/Ansible to update the file on the VM.

### Pattern 3: External Network for Service Mesh

**What:** One shared `proxy` Docker network owned by Traefik. All routable services join this network. Backend-only services (DB, Redis) stay on isolated compose-internal networks.
**When to use:** Multi-compose-file setups where Traefik must reach services defined in different compose projects.
**Trade-off:** Services can potentially reach each other directly through the proxy network — acceptable for a small trusted environment; use firewall rules or a socket proxy in high-security contexts.

---

## Anti-Patterns

### Anti-Pattern 1: Exposing All Docker Services by Default

**What people do:** Set `exposedByDefault: true` in Traefik's Docker provider.
**Why it's wrong:** Every container gets a router, including databases, Redis, internal tools — unintended internet exposure.
**Do this instead:** `exposedByDefault: false` in static config. Services opt-in with `traefik.enable=true` label.

### Anti-Pattern 2: Putting ForwardAuth Middleware on Authentik Itself

**What people do:** Apply the `authentik-forward-auth` middleware to the Authentik router.
**Why it's wrong:** Creates a circular dependency — auth check calls Authentik, which needs auth, which calls Authentik, infinite loop. Users can never log in.
**Do this instead:** The Authentik router must have zero auth middleware. Only downstream services get ForwardAuth.

### Anti-Pattern 3: Storing Secrets in Docker Compose Files or Plain `vars/`

**What people do:** Put `AUTHENTIK_SECRET_KEY: "abc123"` directly in `docker-compose.yml.j2` or `vars/main.yml`.
**Why it's wrong:** Secrets committed to the repo or readable without vault.
**Do this instead:** All secrets in `group_vars/all/vault.yml` encrypted with ansible-vault. The template references `{{ vault_authentik_secret_key }}`. Vault password stored in Semaphore's credential store.

### Anti-Pattern 4: Using Traefik's `insecure: true` API

**What people do:** Enable `api: insecure: true` for easy dashboard access during development.
**Why it's wrong:** Dashboard is exposed on port 8080 without any authentication — anyone who can reach the VM port gets full Traefik config visibility.
**Do this instead:** `api: insecure: false` + ForwardAuth-protected router. The extra setup is done once in Phase 1.

### Anti-Pattern 5: Single Compose File for All Services

**What people do:** Put Traefik, Authentik, the app, databases all in one `docker-compose.yml`.
**Why it's wrong:** Can't restart Authentik without risking Traefik downtime; can't add app services independently; rollbacks affect unrelated services.
**Do this instead:** One compose file per concern (traefik/, authentik/, app/). Shared via the external `proxy` network.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Let's Encrypt | ACME TLS challenge via Traefik | acme.json must be a volume (not bind-mounted from tmpfs) to persist certs across restarts |
| Semaphore | Ansible execution environment | Vault password stored as Semaphore key; no plain-text secrets in playbooks |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Traefik ↔ Authentik Server | HTTP ForwardAuth over `proxy` network | Port 9000 (HTTP); use internal network hostname `authentik-server` |
| Traefik ↔ Future App Services | HTTP over `proxy` network via Docker labels | Services join proxy network; Traefik discovers via Docker provider |
| Authentik Server ↔ PostgreSQL | TCP 5432 over `authentik_internal` network | Not exposed on proxy network |
| Authentik Server ↔ Redis | TCP 6379 over `authentik_internal` network | Not exposed on proxy network |
| Authentik Server ↔ Worker | Shared DB + Redis (no direct HTTP) | Worker polls jobs from Redis/DB |

---

## Recommended File/Directory Structure on VM

```
/opt/kb-pipeline/
├── traefik/
│   ├── docker-compose.yml
│   ├── config/
│   │   ├── traefik.yml              # Static config
│   │   └── dynamic/
│   │       ├── middlewares.yml      # ForwardAuth + security headers
│   │       ├── dashboard.yml        # Traefik dashboard router
│   │       └── tls.yml              # TLS options (ciphers, minVersion)
│   └── certificates/
│       └── acme.json                # Let's Encrypt storage (chmod 600)
│
└── authentik/
    ├── docker-compose.yml
    ├── .env                         # Rendered by Ansible from vault vars
    ├── media/                       # Authentik uploaded media (logos, etc.)
    ├── certs/                       # Custom certs if needed
    └── custom-templates/            # Authentik UI overrides (optional)
```

---

## Sources

- Traefik v3 documentation (traefik.io/traefik/): static/dynamic config split, Docker provider, ForwardAuth middleware — HIGH confidence (well-established patterns, stable since v2)
- Authentik documentation (goauthentik.io/docs/): Docker Compose install, embedded outpost, ForwardAuth with Traefik — HIGH confidence (official install method is stable)
- Ansible community.docker collection: `docker_compose_v2` module — MEDIUM confidence (verify module name matches collection version installed; `community.docker` >= 3.x required for v2)
- Let's Encrypt ACME TLS challenge: supported by Traefik natively, no certbot needed — HIGH confidence

---

*Architecture research for: Traefik + Authentik infrastructure (KB-Pipeline Phase 1)*
*Researched: 2026-03-27*
