# Project Research Summary

**Project:** KB-Pipeline — Traefik + Authentik Infrastructure (Phase 1)
**Domain:** Infrastructure — Reverse Proxy + Identity Provider on Docker Compose, managed by Ansible/Semaphore
**Researched:** 2026-03-27
**Confidence:** MEDIUM (training data through August 2025; external verification unavailable at research time)

## Executive Summary

KB-Pipeline Phase 1 is a self-hosted infrastructure layer: Traefik v3 as a TLS-terminating reverse proxy with Docker label-based service discovery, and Authentik as the identity provider handling Forward Authentication for all protected routes. The entire stack is deployed and managed via Ansible (community.docker collection) with Semaphore UI providing a Git-backed execution environment. This combination is well-established in home-lab and small-production deployments, with a mature integration pattern between Traefik's forwardAuth middleware and Authentik's embedded outpost. The approach is fully reproducible and idempotent when implemented correctly, which is the central operational requirement.

The recommended architecture uses two separate Docker Compose files (one for Traefik, one for Authentik + its dependencies), joined by a single external Docker network named `proxy`. Traefik owns and creates this network; Authentik attaches to it as an external consumer. PostgreSQL and Redis remain isolated on an internal Authentik-only network with no external exposure. Ansible roles deploy each concern independently, with strict ordering: Docker engine first, then Traefik (which creates the proxy network), then Authentik. All secrets are encrypted with ansible-vault and injected at deploy time via Semaphore's key store — never committed in plain text.

The dominant risks are all operational rather than architectural: circular auth loops (applying ForwardAuth to Authentik itself), container startup ordering (Authentik starting before PostgreSQL is ready), secrets leaking into Git, and Ansible playbooks failing idempotency due to Compose file hash drift. Every one of these is avoidable with known patterns that are well-documented in the research. The `AUTHENTIK_SECRET_KEY` must be generated once, stored in vault, and never changed after initial deployment — changing it invalidates all sessions. Phase 1 should be treated as complete only when the Ansible playbook runs idempotently twice from Semaphore with `changed=0` on the second run.

---

## Key Findings

### Recommended Stack

The stack is tightly coupled by design: each technology was chosen specifically for its integration with the others. Traefik v3 is the only viable choice if you want label-driven Docker service discovery with native ForwardAuth middleware support and active development (v2 is in maintenance mode). Authentik is preferred over Keycloak (too heavy) and Authelia (no user management UI) because it provides both the ForwardAuth endpoint and full user/group management in a single self-hosted package. Docker Compose v2 (the plugin) is the current standard; the Python `docker-compose` v1 binary is EOL and absent from modern Docker installs.

**Core technologies:**
- Traefik v3.3: reverse proxy, TLS termination, Docker label-based routing — only active-development option with native Authentik ForwardAuth integration
- Authentik 2024.12.x: OIDC/SSO + ForwardAuth identity provider — self-hosted, Docker-native, embedded outpost eliminates a separate proxy container
- PostgreSQL 16: Authentik's required persistent store — v16 is current LTS; SQLite is not supported by Authentik
- Redis 7: Authentik session cache and task queue — required by Authentik; no persistence needed for Phase 1
- Docker Compose v2 (plugin): container orchestration — v1 is deprecated; use `docker compose` subcommand not `docker-compose` binary
- Ansible 2.16.x + community.docker 3.12.x+: deployment automation — `docker_compose_v2` module (not `docker_compose`) wraps the v2 plugin idiomatically
- Semaphore UI 2.10.x+: Git-backed Ansible execution UI — lightweight AWX alternative, native vault password support

**Critical version notes:**
- Use `community.docker.docker_compose_v2` (not `docker_compose`) — the v1 module is deprecated and incompatible with Docker Compose v2
- Pull Authentik from `ghcr.io/goauthentik/server` — Docker Hub images are unofficial mirrors
- Pin Authentik to a specific `YYYY.MM.PATCH` tag — `latest` will auto-upgrade and break database migrations
- Omit the `version:` key from Compose files — Compose v2 treats it as obsolete and emits deprecation warnings

### Expected Features

All Phase 1 features map to infrastructure validation: the stack must be deployed, authenticated end-to-end, and proven idempotent. There are no application-layer features in Phase 1.

**Must have (table stakes — Phase 1 is not complete without all of these):**
- Traefik TLS termination (Let's Encrypt ACME, HTTP→HTTPS redirect, `exposedByDefault: false`)
- Traefik Docker provider active with shared `proxy` external network
- Traefik dashboard reachable and protected by Authentik ForwardAuth
- Authentik server + worker running (backed by PostgreSQL + Redis with healthcheck-gated `depends_on`)
- Authentik reachable at `[DOMAIN]/authentik` (path-prefix routing via `AUTHENTIK_WEB__PATH` env var)
- Authentik embedded outpost configured as ForwardAuth provider (proxy provider type, not OIDC)
- Traefik ForwardAuth middleware (`trustForwardHeader: true`, full `authResponseHeaders` list)
- Groups Admin, Editor, Viewer created; one test user per group; end-to-end login verified
- Ansible playbook deploys from zero (fresh VM) and runs idempotently (second run: `changed=0`)
- Semaphore pipeline executes the playbook successfully against the test VM

**Should have (Phase 1.x — add once P1 items are verified):**
- Docker Compose healthchecks on all services (PostgreSQL: `pg_isready`, Redis: `redis-cli ping`)
- Traefik access logs in structured JSON format
- Traefik Prometheus metrics endpoint enabled (costs 3 config lines; consumed in a later monitoring phase)
- Secrets in Ansible vault before first commit (treat as P1 for security but listed here to distinguish from functional requirements)

**Defer (Phase 2+):**
- Authentik OIDC provider — configure only when the first OIDC client application (FastAPI backend) exists to test against it
- Grafana + Prometheus monitoring stack — full observability is its own dedicated phase
- Authentik LDAP outpost — only if legacy system integration is explicitly required
- DNS-01 ACME challenge / wildcard certs — only if wildcard certificates become necessary
- External managed PostgreSQL — only at production deployment phase
- Watchtower / automated container updates — explicitly prohibited; breaks Ansible idempotency

**Anti-features (explicitly do not build in Phase 1):**
- `api.insecure: true` on Traefik dashboard
- `exposedByDefault: true` on Docker provider
- ForwardAuth middleware applied to the Authentik router (creates circular auth dependency)
- Traefik Pilot / enterprise features (Pilot is deprecated)
- Multi-tenancy in Authentik

### Architecture Approach

The architecture follows a layered model: Traefik is the public-facing infrastructure layer; Authentik is an application on top of that infrastructure; future app services (FastAPI, Angular) will be applications on top of both. This layering justifies separate Compose files with independent lifecycles. All inter-service communication uses Docker internal DNS (container service names as hostnames), never IP addresses. The ForwardAuth data flow is: Browser → Traefik EntryPoint :443 → ForwardAuth middleware → Authentik outpost at `http://authentik-server:9000/outpost.goauthentik.io/auth/traefik` → (200 + user headers OR 401 + redirect to login). Authentik must not have ForwardAuth applied to its own router — this is the single most common misconfiguration.

**Major components:**
1. Traefik — TLS termination, HTTP routing via Docker labels + file provider, ForwardAuth middleware chain execution; owns the `proxy` Docker network
2. Authentik Server — ForwardAuth endpoint (embedded outpost on :9000), admin UI, OAuth2/OIDC provider, user/group management; attached to `proxy` + `authentik_internal` networks
3. Authentik Worker — background jobs (event logs, policy evaluation, certificate rotation); attached to `authentik_internal` only
4. PostgreSQL — Authentik persistent store; `authentik_internal` network only, no external port binding
5. Redis — Authentik session cache + task broker; `authentik_internal` network only, no external port binding
6. Ansible roles (docker, traefik, authentik) — deploy and configure the above in dependency order; use `community.docker.docker_compose_v2` with `state: present` for idempotency
7. Semaphore UI — Git-backed Ansible execution; stores vault password and SSH key; triggers playbook runs

**Key architectural patterns:**
- `exposedByDefault: false` + opt-in via `traefik.enable=true` label — prevents accidental service exposure
- Traefik dashboard router defined in dynamic file config (not Docker labels) because Traefik is not tracked by its own Docker provider
- ForwardAuth middleware defined once in dynamic file config (`middlewares.yml`), referenced by all protected routers via `@file` suffix
- Authentik router has no auth middleware; all other routers reference `authentik-forward-auth@file`
- Ansible: template Compose files via Jinja2, notify handlers on change, use `community.docker.docker_compose_v2` with `state: present`; never use `down -v` (destroys volumes)
- Secrets: all in `group_vars/all/vault.yml` encrypted with ansible-vault; Semaphore injects vault password at runtime

### Critical Pitfalls

The research identified 11 pitfalls; all are preventable in Phase 1 design. The top 5 by impact:

1. **Authentik outpost not on the shared Docker network** — Traefik cannot resolve `authentik-server` hostname; every protected route returns 500. Prevent by creating the `proxy` external network as an Ansible `community.docker.docker_network` task before any Compose deployment, and attaching both Traefik and `authentik-server` to it.

2. **ForwardAuth middleware applied to the Authentik router** — creates a circular auth dependency; users can never log in. Prevent by explicitly omitting `authentik-forward-auth` middleware from the Authentik router labels; apply it only to downstream service routers.

3. **`AUTHENTIK_SECRET_KEY` not set statically** — Authentik generates a random key at each container start; all sessions are invalidated on every playbook run (container recreate). Prevent by generating the key once with `openssl rand -hex 32`, storing it in ansible-vault, and injecting it via the Compose template. Never change it after initial deployment.

4. **Ansible non-idempotency (containers recreated on every run)** — violates the core reproducibility requirement and causes unnecessary downtime. Prevent by using `community.docker.docker_compose_v2` with `state: present` and `recreate: auto`; template Compose files with `notify` handlers; pin all image tags to specific versions.

5. **Secrets committed to Git in plain text** — critical security failure for an internet-accessible VM. Prevent by encrypting `group_vars/all/vault.yml` with ansible-vault before the first commit; storing the vault password in Semaphore's key store; verifying with `git grep -i "password\|secret\|key"` returning no plain-text credentials.

Additional pitfalls worth noting:
- **TLS redirect loop on forwardAuth address** — always use internal HTTP URL (`http://authentik-server:9000/...`), never the public HTTPS URL
- **Missing `trustForwardHeader` and `authResponseHeaders`** — without these, upstream services receive no user identity headers after auth; copy the complete header list from STACK.md
- **Authentik cookie domain mismatch** — set `AUTHENTIK_COOKIE_DOMAIN` to the apex domain; prevents auth loops when services are on different paths
- **Container startup ordering without healthchecks** — use `depends_on: condition: service_healthy` with `pg_isready` and `redis-cli ping` healthchecks; Authentik crash-loops otherwise on cold start
- **Semaphore SSH key type mismatch** — store as "SSH Key" type, not "login with password"; test manual SSH from the Semaphore runner before first playbook run

---

## Implications for Roadmap

The research makes the phase structure unambiguous: there is one Phase 1 (infrastructure foundation) with a single acceptance criterion — the Ansible playbook deploys and runs idempotently from Semaphore. All subsequent phases add application services on top of this foundation. The order within Phase 1 is strictly determined by hard dependencies: Docker must be installed before anything else; Traefik must create the `proxy` network before Authentik can attach to it.

### Phase 1: Infrastructure Foundation (Traefik + Authentik + Ansible/Semaphore)

**Rationale:** Nothing else can be deployed until Traefik is routing with TLS and Authentik is protecting routes. This is the load-bearing layer. All 11 identified pitfalls apply to this phase — it is the most complex phase in terms of integration surface area despite involving zero application code.

**Delivers:**
- HTTPS routing for all future services (TLS via Let's Encrypt)
- Authentication gate for all internal tools (Authentik ForwardAuth)
- Traefik dashboard protected and accessible at `[DOMAIN]/traefik`
- Authentik admin UI accessible at `[DOMAIN]/authentik`
- Three user groups (Admin, Editor, Viewer) with verified test users
- Fully automated, idempotent deployment from Semaphore

**Addresses (from FEATURES.md):** All P1 table stakes features

**Avoids (from PITFALLS.md):**
- Circular ForwardAuth loop (Authentik router has no auth middleware)
- Secret key invalidation (static `AUTHENTIK_SECRET_KEY` in vault)
- Startup ordering failures (healthcheck-gated `depends_on`)
- Non-idempotency (correct Ansible module + no `--force-recreate`)
- Secrets in Git (ansible-vault from first commit)

**Deployment order within Phase 1:**
1. Semaphore project setup (SSH key, vault password, inventory)
2. Ansible role: `docker` — installs Docker Engine, Compose plugin, `python3-docker`
3. Ansible role: `traefik` — creates `proxy` network, deploys Traefik with static + dynamic config
4. Ansible role: `authentik` — deploys PostgreSQL, Redis, authentik-server, authentik-worker; configures ForwardAuth provider
5. Manual post-deploy: Authentik initial setup wizard, create groups and test users
6. Verification: incognito window test of ForwardAuth redirect; second Semaphore run confirms `changed=0`

### Phase 2: Application Services (FastAPI Backend)

**Rationale:** Once the infrastructure layer is verified idempotent, application services can be added. The first app service introduces the Authentik OIDC provider (which was deliberately deferred from Phase 1 — no OIDC provider without a client app to validate it against).

**Delivers:** FastAPI backend accessible behind Traefik + Authentik, receiving user identity headers or OIDC tokens

**Uses:** Shared `proxy` network (external, Traefik-owned); Authentik OIDC provider configured for the first time; existing Ansible patterns extended with a new role

**Implements:** `proxy` network attachment pattern for application services; OIDC provider + application pair in Authentik

**Avoids:** Configuring OIDC provider without a live client (explicitly deferred in FEATURES.md)

### Phase 3: Frontend Application (Angular)

**Rationale:** Frontend deployment follows backend — the Angular app needs an API to communicate with. Traefik routing for static assets and API proxying are straightforward extensions of established Phase 1 patterns.

**Delivers:** Angular frontend accessible at `[DOMAIN]/app` or similar, routed by Traefik

**Uses:** Existing ForwardAuth middleware; Traefik Docker label pattern already established

### Phase 4: Observability Stack (Prometheus + Grafana)

**Rationale:** Traefik Prometheus metrics endpoint is enabled in Phase 1 (zero cost), ready for consumption. The full observability stack (Prometheus + Grafana + alerting) is its own lifecycle and should not block application delivery.

**Delivers:** Metrics collection and dashboards for Traefik and application services

**Uses:** Pre-enabled Prometheus endpoint on Traefik; Authentik-protected Grafana dashboard

### Phase Ordering Rationale

- Phase 1 before all others: hard dependency — Traefik `proxy` network must exist before any other service can join it; Authentik must be running before any ForwardAuth-protected route is meaningful
- Phase 2 before Phase 3: API-first — frontend depends on a working backend endpoint
- Phase 4 last: observability is infrastructure for operations, not a user-facing feature; Traefik metrics endpoint is already enabled so nothing blocks Phase 4 from starting early in parallel if desired
- OIDC provider configuration is strictly Phase 2+ because testing it requires a live OIDC client application (FastAPI); configuring it in Phase 1 produces unverified configuration that can silently drift

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1 (Authentik subpath routing):** `AUTHENTIK_WEB__PATH` env var name has changed across Authentik versions — verify the exact env var name against current Authentik docs before implementing. LOW confidence on this specific detail. The FEATURES.md note flags this explicitly.
- **Phase 1 (ACME challenge selection):** TLS challenge vs HTTP-01 vs DNS-01 depends on whether the target VM has port 443 reachable from the public internet. Verify network topology before choosing ACME method.
- **Phase 2 (Authentik OIDC provider setup):** OIDC provider configuration in Authentik is more complex than proxy providers; the specific redirect URIs, client ID/secret flow, and group claims configuration will benefit from a focused research pass when Phase 2 is planned.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Traefik core setup):** Traefik v3 Docker label patterns, static/dynamic config split, and ForwardAuth middleware are extremely well-documented with HIGH confidence. No additional research needed.
- **Phase 1 (Ansible `docker_compose_v2` module):** Module behavior is stable and well-understood. The key decisions (module name, `state: present`, no `--force-recreate`) are documented in STACK.md with HIGH confidence.
- **Phase 3 (Angular Traefik routing):** Static asset routing is trivial; Traefik label pattern is identical to Phase 2.
- **Phase 4 (Prometheus + Grafana):** Standard Docker Compose deployment behind Traefik; endpoint is pre-enabled.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | Core technology choices are HIGH confidence (well-established patterns); specific patch versions need verification at release pages — Traefik, Authentik, community.docker, Semaphore all update frequently |
| Features | MEDIUM | Feature list derived from PROJECT.md requirements (HIGH confidence primary source) + Authentik/Traefik docs (MEDIUM); `AUTHENTIK_WEB__PATH` env var name is LOW confidence — must verify before implementation |
| Architecture | HIGH | Component boundaries, network topology, and data flow are well-established patterns; static/dynamic Traefik config split and ForwardAuth flow are documented with multiple community confirmations |
| Pitfalls | MEDIUM | All 11 pitfalls are derived from known failure modes in Traefik + Authentik deployments; recovery strategies are verified; specific Semaphore vault password configuration syntax is LOW confidence — verify against current Semaphore docs |

**Overall confidence:** MEDIUM

### Gaps to Address

- **`AUTHENTIK_WEB__PATH` exact env var name:** Research flags this as LOW confidence; this must be verified against current Authentik documentation (https://docs.goauthentik.io/) before writing the Compose template. If the env var name has changed in recent versions, Option B (Traefik stripprefix middleware) is the fallback — but it has known redirect flow issues documented in FEATURES.md.

- **ACME challenge method:** The research recommends TLS challenge (`tlsChallenge`) as the default but notes that the actual method depends on whether port 443 is directly reachable from Let's Encrypt servers. DNS-01 is the fallback for NAT-ed VMs. This must be confirmed against the actual network topology of the test VM before Phase 1 implementation begins.

- **Authentik outpost URL format for Traefik provider:** The ForwardAuth address suffix (`/auth/traefik` vs `/auth/nginx`) must match the Authentik version's actual endpoint path. FEATURES.md notes this as MEDIUM confidence and flags it for verification against current Authentik outpost documentation.

- **Semaphore vault password injection mechanism:** The exact environment variable name and configuration method for passing the ansible-vault password to a Semaphore task is LOW confidence. Verify in current Semaphore documentation before configuring the project.

- **community.docker minimum version:** The `docker_compose_v2` module requires community.docker >= 3.6.0. Verify the version available via `ansible-galaxy collection install community.docker` returns >= 3.12.x as recommended.

---

## Sources

### Primary (HIGH confidence)
- PROJECT.md requirements for KB-Pipeline Phase 1 — feature requirements and acceptance criteria
- Traefik v3 official documentation patterns (static/dynamic config, Docker provider, ForwardAuth middleware) — architecture decisions
- Authentik official Docker Compose install pattern — component structure and network design

### Secondary (MEDIUM confidence)
- Training data: Traefik v2/v3, Authentik 2024.x, Ansible community.docker 3.x, Semaphore 2.x docs through August 2025
- Community deployment patterns for Traefik + Authentik ForwardAuth — well-represented in GitHub issues and blog posts through training cutoff
- Ansible community.docker collection changelog — module deprecation and v1/v2 distinction

### Tertiary (LOW confidence — verify before implementing)
- `AUTHENTIK_WEB__PATH` exact env var name — verify at https://docs.goauthentik.io/
- Authentik outpost URL format for Traefik (`/auth/traefik` suffix) — verify against current Authentik outpost docs
- Semaphore vault password injection configuration — verify at https://docs.semaphoreui.com/
- Current patch versions for all stack components — verify at respective release pages

---
*Research completed: 2026-03-27*
*Ready for roadmap: yes*
