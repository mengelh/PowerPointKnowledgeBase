# Feature Research

**Domain:** Infrastructure — Reverse Proxy + Identity Provider (Traefik + Authentik on Docker Compose)
**Researched:** 2026-03-27
**Confidence:** MEDIUM (training knowledge through Aug 2025; external verification unavailable in this session)

---

## Feature Landscape

### Table Stakes (Must Work Before Any Other Service Deploys)

These are non-negotiable. Without each one, the infrastructure layer is not functional. No
application service can be deployed safely on top of a missing table stake.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Traefik TLS termination (Let's Encrypt / ACME)** | All traffic must be HTTPS; HTTP → HTTPS redirect required | MEDIUM | Needs valid domain + port 80/443 reachable. ACME HTTP-01 or DNS-01 challenge. Use `certificatesResolvers` in static config. |
| **Traefik Docker provider** | Service discovery from `docker-compose.yml` labels — zero manual routing config per service | LOW | Must set `exposedByDefault: false`; each container opts in via `traefik.enable=true` label. |
| **Traefik entrypoints (web + websecure)** | Port 80 and 443 listening, HTTP → HTTPS redirect middleware on `web` entrypoint | LOW | Define in static config (`traefik.yml` or CLI flags). Redirect as default middleware on entrypoint. |
| **Traefik Docker network isolation** | All proxied services share one dedicated Docker network with Traefik; other networks remain internal | LOW | Create `proxy` network externally; services not needing Traefik stay off it. Critical for security. |
| **Authentik server + worker running** | Authentik requires both `authentik-server` and `authentik-worker` containers; worker handles background tasks | MEDIUM | Worker needs same secrets/env as server. Requires PostgreSQL + Redis as hard dependencies. |
| **Authentik PostgreSQL dependency** | Authentik persists all identity data in PostgreSQL; without it Authentik cannot start | LOW | Use official `postgres:16` or `17` image. Dedicated volume for data persistence. |
| **Authentik Redis dependency** | Authentik uses Redis for session cache, task queue, and real-time events | LOW | Use `redis:alpine`. No persistence required for Phase 1 (sessions can be lost on restart). |
| **Authentik reachable at `/authentik` path prefix** | Operators need to reach Authentik UI to configure groups, flows, and providers | MEDIUM | Authentik supports `AUTHENTIK_WEB__PATH` env var for path prefix OR Traefik stripprefix middleware. Path-prefix approach is cleaner. Verify current Authentik docs on subpath support — this has changed across versions. |
| **Authentik outpost (embedded or external)** | Forward Auth requires an outpost to handle auth challenges. Embedded outpost is part of Authentik server for small deployments | MEDIUM | Embedded outpost is the default for single-node setups. No separate container needed. |
| **Traefik ForwardAuth middleware pointing to Authentik** | Every protected route must pass auth check through Authentik before proxying | MEDIUM | Middleware config: `http.middlewares.authentik.forwardAuth.address = http://authentik-server:9000/outpost.goauthentik.io/auth/nginx`. Headers `X-Forwarded-*` must be passed correctly. |
| **Traefik dashboard reachable and protected** | Dashboard exposes routing state — must exist but must not be publicly accessible | LOW | Enable via `api.dashboard: true` in static config. Apply `authentik` ForwardAuth middleware. Route via `traefik.http.routers.traefik` label. |
| **Authentik Admin group created** | Phase 1 explicitly requires Admin, Editor, Viewer groups with test users | LOW | Create via Authentik admin UI or bootstrap flow post-install. Groups are used downstream for RBAC claims in OIDC tokens. |
| **Authentik Editor group created** | See Admin group — same rationale | LOW | Same mechanism. |
| **Authentik Viewer group created** | See Admin group — same rationale | LOW | Same mechanism. |
| **Test user per group, login verified** | Proves end-to-end: Authentik auth flow → Traefik ForwardAuth → protected route works | LOW | Manual verification step. Part of acceptance criteria, not automated in Phase 1. |
| **Ansible playbook deploys full stack idempotently** | Reproducibility is a hard requirement (Semaphore CI, no direct SSH) | HIGH | Playbook must handle: Docker network creation, volume creation, image pull, `docker compose up -d`, and wait for healthchecks. Idempotency requires checking state before acting. |
| **Semaphore can execute playbook against test VM** | Deployment pipeline must be verified as functional in Phase 1 | MEDIUM | Requires Semaphore project + inventory + SSH key configured. Playbook must not fail on re-run. |

---

### Differentiators (Adds Robustness — Phase 1 Nice-to-Have)

These are not required for the infrastructure to function, but they meaningfully improve
operational quality and reduce future pain.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Traefik access logs** | Audit trail for all HTTP requests. Valuable for debugging auth failures during later app development | LOW | `accessLog: {}` in static config. Structured JSON format preferred. |
| **Traefik metrics endpoint (Prometheus)** | Enables future monitoring without retroactive instrumentation | LOW | `metrics.prometheus: {}` in static config. Not consumed in Phase 1 but costs nothing to enable. |
| **Authentik error brand/logo customization** | Makes auth pages look intentional rather than default. Reduces confusion for test users | LOW | Authentik tenant settings. Not functional but improves operator experience. |
| **Docker Compose health checks on all services** | Ansible `wait_for` task becomes more reliable; dependent services start in correct order | MEDIUM | Each service needs `healthcheck` block. Traefik, Authentik server, PostgreSQL, Redis each have different check commands. |
| **Traefik static config via file (`traefik.yml`)** | Separates static config from runtime labels; easier to audit and version-control | LOW | Alternative is CLI flags in `command:` block. File is more readable and Ansible-templatable. |
| **Authentik secret key in environment variable, injected via Ansible vault** | Prevents secret leakage in Docker Compose files committed to git | LOW | Use `AUTHENTIK_SECRET_KEY` env var. Ansible vault encrypts the value. Generate once with `openssl rand -hex 32`. |
| **Dedicated Docker volumes for PostgreSQL and Authentik media** | Data survives container recreate during Ansible re-runs | LOW | Named volumes in `docker-compose.yml`. Ansible must not `down -v` — use `up -d --remove-orphans` instead. |
| **Traefik rate limiting middleware (global)** | Prevents accidental DoS during development; protects Authentik login endpoint | LOW | `http.middlewares.ratelimit.rateLimit` — apply to Authentik login router. Not critical but cheap. |

---

### Anti-Features (Explicitly Do NOT Build in Phase 1)

| Feature | Why Requested | Why Problematic for Phase 1 | Alternative |
|---------|---------------|----------------------------|-------------|
| **Authentik OIDC Provider for app SSO** | App services will eventually use OIDC. Seems logical to configure now. | No app services exist in Phase 1. Configuring OIDC providers without a client app to test them creates unverified config that may drift. | Add OIDC provider setup when the first app service (FastAPI backend) is added in a later phase. |
| **Authentik LDAP outpost** | LDAP is sometimes wanted for legacy compatibility | LDAP outpost is a separate container, adds operational complexity, and is not needed for any planned service. | Authentik OIDC/ForwardAuth covers all planned use cases. |
| **Traefik Let's Encrypt DNS-01 challenge via external DNS provider API** | More robust than HTTP-01 for wildcard certs | Requires API credentials for DNS provider (Cloudflare, Route53, etc.). Adds secret management complexity. HTTP-01 works fine for single-domain test VM. | Use HTTP-01 ACME in Phase 1. Migrate to DNS-01 only if wildcard certs become necessary. |
| **Watchtower or automated container updates** | "Keeps things up to date automatically" | Automated updates break reproducibility. Ansible idempotency becomes unreliable if containers are updated out-of-band. | Pin image versions. Update deliberately via Ansible playbook changes. |
| **Grafana + Prometheus monitoring stack** | Metrics endpoint is enabled — monitoring stack seems natural | Full observability stack (Prometheus + Grafana + alerting) is a project of its own. Adds 3+ containers, persistent storage, and dashboard config. Not needed to validate infrastructure layer. | Enable Traefik metrics endpoint (cost: 3 config lines). Add monitoring stack as a dedicated phase later. |
| **Traefik TCP/UDP routing** | Traefik supports non-HTTP protocols | No planned service uses raw TCP/UDP. Enables attack surface without benefit. | HTTP/HTTPS routing only. Add TCP routing when a service explicitly requires it. |
| **Authentik multi-tenancy / multiple tenants** | Authentik supports tenant isolation | Single tenant (one domain, one company). Multi-tenancy adds configuration complexity with zero benefit. | Default single-tenant Authentik setup. |
| **External PostgreSQL (managed DB)** | Production best practice | Phase 1 is test-VM only. Managed DB adds cost, network dependency, and setup complexity out of scope. | Containerized PostgreSQL with named volume. Revisit at production phase. |
| **Traefik pilot / enterprise features** | Traefik Pilot provides cloud dashboard | Traefik Pilot has been deprecated. Do not configure it. | Local Traefik dashboard (protected by Authentik) is sufficient. |

---

## Feature Dependencies

```
[Traefik TLS termination]
    └──requires──> [Valid domain pointing to test VM]
    └──requires──> [Port 80 open for ACME HTTP-01 challenge]

[Traefik ForwardAuth middleware]
    └──requires──> [Authentik outpost reachable by Traefik]
                       └──requires──> [Authentik server running]
                                          └──requires──> [PostgreSQL running]
                                          └──requires──> [Redis running]

[Traefik dashboard protected]
    └──requires──> [Traefik ForwardAuth middleware]
    └──requires──> [Authentik admin user exists and can log in]

[Authentik groups (Admin/Editor/Viewer)]
    └──requires──> [Authentik server running and initial setup complete]

[Test user login verified]
    └──requires──> [Authentik groups created]
    └──requires──> [Traefik ForwardAuth middleware applied to at least one route]

[Ansible playbook idempotent]
    └──requires──> [All services have health checks or known startup times]
    └──requires──> [Secrets injected via Ansible vault, not hardcoded]

[Semaphore execution]
    └──requires──> [Ansible playbook written and tested locally first]
    └──requires──> [SSH key added to Semaphore project]
```

### Dependency Notes

- **Authentik requires PostgreSQL + Redis before starting:** Docker Compose `depends_on` with `condition: service_healthy` must be set. Without healthcheck conditions, Authentik server may start before PostgreSQL is ready, causing startup failures.
- **Traefik ForwardAuth requires Authentik outpost URL:** The outpost URL in the ForwardAuth middleware config is a static address. If Authentik moves (IP or port change), the middleware breaks. Use Docker service name (`authentik-server`) not IP.
- **Traefik dashboard protection requires ForwardAuth before dashboard is publicly reachable:** The router applying ForwardAuth must be configured before the Traefik service is exposed. Do not expose dashboard without middleware in the same commit.
- **Ansible idempotency requires named volumes:** If Ansible runs `docker compose down -v`, it destroys Authentik's PostgreSQL data. The playbook must only use `up -d --remove-orphans`.

---

## MVP Definition (Phase 1 Infrastructure)

### Launch With (Phase 1 Complete)

- [ ] Traefik running, TLS terminating via Let's Encrypt — HTTPS works for all routes
- [ ] Traefik HTTP → HTTPS redirect active — no plaintext access
- [ ] Traefik Docker provider active — services added by label
- [ ] Traefik dashboard reachable at `[DOMAIN]/traefik` — protected by Authentik ForwardAuth
- [ ] Authentik running (server + worker) — backed by PostgreSQL + Redis
- [ ] Authentik reachable at `[DOMAIN]/authentik`
- [ ] Authentik initial setup complete — admin superuser created
- [ ] Authentik ForwardAuth provider + outpost configured
- [ ] Traefik ForwardAuth middleware pointing to Authentik outpost
- [ ] Groups Admin, Editor, Viewer created in Authentik
- [ ] One test user per group, login flow verified end-to-end
- [ ] Ansible playbook deploys full stack from zero (fresh VM) — verified
- [ ] Ansible playbook is idempotent — second run changes nothing — verified via Semaphore

### Add After Validation (Phase 1.x or Phase 2 prep)

- [ ] Authentik OIDC provider configured — add when FastAPI backend is introduced
- [ ] Traefik access logs structured JSON — add when first app service is deployed and debugging is needed
- [ ] Traefik Prometheus metrics — add when monitoring phase begins
- [ ] Docker Compose healthchecks on all services — add if Ansible `wait_for` proves flaky

### Future Consideration (Phase 3+)

- [ ] Grafana + Prometheus monitoring stack — dedicated observability phase
- [ ] Authentik LDAP outpost — only if legacy system integration is required
- [ ] DNS-01 ACME challenge — only if wildcard certificates become necessary
- [ ] External managed PostgreSQL — only at production deployment phase

---

## Feature Prioritization Matrix

| Feature | Operator Value | Implementation Cost | Priority |
|---------|---------------|---------------------|----------|
| Traefik TLS + HTTPS | HIGH | MEDIUM | P1 |
| Traefik Docker provider + HTTP→HTTPS redirect | HIGH | LOW | P1 |
| Authentik server + PostgreSQL + Redis | HIGH | MEDIUM | P1 |
| Authentik ForwardAuth provider + outpost | HIGH | MEDIUM | P1 |
| Traefik ForwardAuth middleware | HIGH | MEDIUM | P1 |
| Traefik dashboard protected | HIGH | LOW | P1 |
| Authentik groups + test users | HIGH | LOW | P1 |
| Ansible playbook idempotent | HIGH | HIGH | P1 |
| Semaphore pipeline verified | HIGH | LOW | P1 |
| Authentik subpath routing | MEDIUM | MEDIUM | P1 (required by spec) |
| Docker named volumes for data persistence | MEDIUM | LOW | P1 |
| Secrets via Ansible vault | MEDIUM | LOW | P1 |
| Docker Compose healthchecks | MEDIUM | MEDIUM | P2 |
| Traefik access logs | MEDIUM | LOW | P2 |
| Traefik Prometheus metrics endpoint | LOW | LOW | P2 |
| Authentik branding customization | LOW | LOW | P3 |
| Traefik rate limiting | LOW | LOW | P3 |
| Grafana + Prometheus stack | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for Phase 1 launch (infrastructure cannot be declared done without it)
- P2: Should have — add once P1 items are verified working
- P3: Nice to have — future phase

---

## Key Implementation Notes

### Authentik Subpath Routing

The PROJECT.md spec requires Authentik at `[DOMAIN]/authentik` (not a subdomain). Authentik's
support for subpath routing has historically been inconsistent. Two approaches:

**Option A — `AUTHENTIK_WEB__PATH` environment variable** (HIGH confidence, native support as of
Authentik 2023.x): Set `AUTHENTIK_WEB__PATH=/authentik` in the Authentik server container. Traefik
proxies to `/authentik` without stripping the prefix.

**Option B — Traefik stripprefix middleware**: Strip `/authentik` before forwarding. This breaks
Authentik's internal redirect flows because Authentik generates redirects based on the path it
believes it's at.

**Recommendation:** Use Option A. Verify `AUTHENTIK_WEB__PATH` is still the correct env var name
against current Authentik docs before implementing. (LOW confidence on exact env var name — verify.)

### ForwardAuth: Single vs Domain Mode

Authentik ForwardAuth supports two modes:

- **Single application mode:** Auth check only for specific routes. Each protected service needs its
  own ForwardAuth middleware pointing to that application's outpost URL.
- **Domain mode (catch-all):** One middleware protects everything under the domain. Simpler for
  Phase 1 where all services are internal tools.

**Recommendation:** Use domain mode for Phase 1. Traefik dashboard + Authentik admin UI are the only
two services — a single ForwardAuth middleware on a catch-all router is appropriate. When app
services with public endpoints are added, switch to per-application ForwardAuth.

### Traefik Dashboard Routing

Traefik's internal dashboard is served by the Traefik API. It does not run as a Docker container,
so it cannot use Docker labels for routing in the standard way. It requires a router defined in
the dynamic file config or via the static `api` config + a dedicated router label on a dummy
container (the `traefik` container itself). The correct pattern:

```yaml
# In traefik docker-compose service labels:
traefik.http.routers.traefik-dashboard.rule=Host(`domain`) && PathPrefix(`/traefik`)
traefik.http.routers.traefik-dashboard.middlewares=authentik@file,strip-traefik-prefix@file
traefik.http.routers.traefik-dashboard.service=api@internal
traefik.http.routers.traefik-dashboard.tls.certresolver=letsencrypt
```

This is a known Traefik pattern — routers can reference the internal `api@internal` service.
(HIGH confidence — Traefik v2/v3 documented behavior.)

---

## Sources

- Traefik v3 documentation (https://doc.traefik.io/traefik/) — training knowledge through Aug 2025; MEDIUM confidence
- Authentik documentation (https://docs.goauthentik.io/) — training knowledge through Aug 2025; MEDIUM confidence
- Authentik ForwardAuth + Traefik integration guides — community-verified pattern; MEDIUM confidence
- PROJECT.md requirements for KB-Pipeline Phase 1 — HIGH confidence (primary source)
- External web access unavailable during this research session — verify `AUTHENTIK_WEB__PATH` and current Authentik outpost URL format before implementation

---

*Feature research for: Traefik + Authentik Phase 1 Infrastructure (KB-Pipeline)*
*Researched: 2026-03-27*
