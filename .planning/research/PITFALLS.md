# Pitfalls Research

**Domain:** Traefik + Authentik + Docker Compose + Ansible/Semaphore infrastructure
**Researched:** 2026-03-27
**Confidence:** MEDIUM (training data through August 2025; WebSearch/WebFetch unavailable; verify against current official docs before implementation)

---

## Critical Pitfalls

### Pitfall 1: Authentik Outpost Not on the Same Docker Network as Traefik

**What goes wrong:**
The Authentik embedded outpost (or a dedicated proxy outpost) cannot be reached by Traefik at the `forwardAuth` address. Traefik receives connection refused or DNS resolution errors. Every protected route returns 500 or a blank page instead of the login redirect.

**Why it happens:**
Traefik and Authentik are defined in separate `docker-compose.yml` files (one for infra, one for apps) or in different Compose projects with distinct default networks. The outpost container name is used as the `forwardAuth` address but Traefik cannot resolve it because it lives in a different network namespace.

**How to avoid:**
- Create a single shared external Docker network (`proxy` or `traefik`) in advance: `docker network create proxy`.
- Declare this network as `external: true` in every Compose file that needs to communicate.
- Attach both Traefik and the Authentik server (and outpost) to this shared network.
- Use the container service name as the hostname only when containers share a network. Use the explicit network alias otherwise.
- Ansible task: create the network before any `docker compose up` task, using `community.docker.docker_network` with `state: present` (idempotent).

**Warning signs:**
- `dial tcp: lookup authentik-server on ...: no such host` in Traefik logs.
- Traefik dashboard shows the forwardAuth middleware as unhealthy/errored.
- HTTP 500 on any protected route immediately after stack start.

**Phase to address:** Phase 1 — Traefik + Authentik deployment

---

### Pitfall 2: TLS Redirect Loop on the Authentik /outpost.goauthentik.io Endpoint

**What goes wrong:**
Requests to Authentik's outpost endpoint (`/outpost.goauthentik.io/...`) get caught in a redirect loop: Traefik redirects HTTP → HTTPS, but the internal forwardAuth call from Traefik to the outpost is already on HTTP (internal Docker network). The result is either an infinite redirect or a 301 loop that browsers refuse to follow.

**Why it happens:**
Two common causes:
1. A global HTTP→HTTPS redirect middleware is applied to ALL routes including the outpost's internal route.
2. The `forwardAuth.address` points to the HTTPS URL (external domain) instead of the internal container HTTP address.

**How to avoid:**
- The `forwardAuth.address` must always use the **internal HTTP address** of the outpost container, e.g. `http://authentik-server:9000/outpost.goauthentik.io/...`. Never point it at the public HTTPS URL.
- Apply the HTTP→HTTPS redirect middleware only to the `web` (port 80) entrypoint, not globally to all routers.
- Use a separate `websecure` entrypoint (port 443) without the redirect middleware applied at the entrypoint level — let it be the TLS termination point.
- Verify: `traefik.http.routers.authentik.entrypoints=websecure` on the Authentik container labels; do not add the redirect middleware to this router.

**Warning signs:**
- Browser reports "too many redirects" (ERR_TOO_MANY_REDIRECTS) on first navigation.
- `curl -v http://[domain]/outpost.goauthentik.io/...` shows repeated 301s.
- Traefik access logs show the same request cycling through multiple times.

**Phase to address:** Phase 1 — Traefik TLS and middleware configuration

---

### Pitfall 3: Missing `trustForwardHeader` and Incorrect `authResponseHeaders` in forwardAuth Middleware

**What goes wrong:**
After successful Authentik authentication, the protected backend never receives the user identity headers (`X-authentik-username`, `X-authentik-groups`, etc.). Services that depend on these headers for authorization behave as if no user is logged in, or Traefik strips/ignores the response headers from the outpost.

**Why it happens:**
Traefik's `forwardAuth` middleware by default does not forward response headers from the auth server to the upstream backend. `authResponseHeaders` must explicitly list every header Authentik sets. Additionally, if `trustForwardHeader: false` (the default), Authentik may not trust `X-Forwarded-For` and `X-Forwarded-Proto`, breaking redirect URL construction.

**How to avoid:**
- In the forwardAuth middleware definition, set `trustForwardHeader: true` so Authentik receives the correct scheme/host for redirect URLs.
- Explicitly list `authResponseHeaders`:
  ```yaml
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
  ```
- Reference the Authentik documentation for the current complete list of headers for the proxy provider type in use.

**Warning signs:**
- Login succeeds (cookie set), but the application receives no user headers.
- Authentik admin UI shows the outpost as healthy but applications log "anonymous user" on every request.
- The redirect after login goes to the wrong URL (mismatched scheme http vs https).

**Phase to address:** Phase 1 — forwardAuth middleware configuration

---

### Pitfall 4: Authentik Cookie Domain Mismatch Causing Authentication Loops

**What goes wrong:**
The Authentik session cookie is set with a domain that does not match the domain of the protected services. Users log in successfully, get redirected back to the application, but are immediately redirected to login again. The loop continues indefinitely.

**Why it happens:**
Authentik sets the session cookie on the exact domain returned in the redirect. If the `AUTHENTIK_COOKIE_DOMAIN` environment variable is not set (or set incorrectly), and services are on different subdomains (e.g. `traefik.example.com`, `app.example.com`, `authentik.example.com`), the cookie will only be valid for the specific subdomain that issued it rather than `.example.com`.

**How to avoid:**
- Set `AUTHENTIK_COOKIE_DOMAIN=example.com` (apex domain without leading dot — Authentik prepends the dot internally) in the Authentik server environment variables in docker-compose.
- Ensure all protected services use the same apex domain.
- When using a subdirectory path (e.g. `[DOMAIN]/authentik`) instead of a subdomain, cookie domain scoping is less of an issue but the `AUTHENTIK_WEB_ROOT` / path prefix configuration must still be correct.
- Verify the `Set-Cookie` header in the browser DevTools after first login.

**Warning signs:**
- Browser enters a login redirect loop (login page → redirect → login page).
- Browser DevTools shows a `Set-Cookie` header with a domain value that doesn't match the current page's domain.
- Authentik logs show repeated new session creation for the same user.

**Phase to address:** Phase 1 — Authentik environment configuration

---

### Pitfall 5: Authentik Proxy Provider vs. OIDC Provider Confusion for the Traefik Dashboard

**What goes wrong:**
The Traefik dashboard is protected using an OIDC provider configuration in Authentik, but Traefik forwardAuth only supports the **proxy provider** (forward auth) pattern. The OIDC flow requires the application to implement the OIDC client — Traefik's dashboard does not. The result is that the dashboard is either unauthenticated or authentication simply does not work.

**Why it happens:**
Authentik offers both proxy providers (forward auth — handled entirely by Authentik/Traefik, no app changes needed) and OIDC providers (app handles token exchange). For services like the Traefik dashboard that have no built-in auth, **proxy provider in forward auth mode** is required.

**How to avoid:**
- For the Traefik dashboard: create a **proxy provider** in Authentik with mode "Forward auth (single application)".
- Set the external host to the exact URL of the Traefik dashboard, including path if using path-based routing.
- Create an application in Authentik backed by this proxy provider and assign it to the Admin group only.
- For future services with their own OIDC client support (FastAPI, Angular), use OIDC providers instead.

**Warning signs:**
- Traefik dashboard shows no login prompt and is accessible without credentials (auth bypass).
- Authentik logs show no access events for dashboard requests.
- Authentik application is listed with an OIDC provider type but the application has no embedded client.

**Phase to address:** Phase 1 — Authentik application and provider setup

---

### Pitfall 6: Ansible `docker compose up` Idempotency — Re-creating Containers Every Run

**What goes wrong:**
The Ansible playbook uses `ansible.builtin.shell: docker compose up -d` or `community.docker.docker_compose_v2` without proper change detection. Every playbook run tears down and recreates containers (even when nothing changed), causing unnecessary downtime and breaking the "idempotent" requirement.

**Why it happens:**
`docker compose up -d` will recreate containers if the Compose file hash changes or if environment variables differ. When Ansible templates the docker-compose.yml file from a Jinja2 template, minor whitespace differences or variable reordering can change the hash. Also, `--force-recreate` used in playbooks "to be safe" defeats idempotency.

**How to avoid:**
- Use the `community.docker.docker_compose_v2` module (requires Docker Compose v2 plugin) with `state: present` — this is the idiomatic idempotent approach.
- Use `recreate: auto` (the default) which only recreates when configuration has actually changed.
- Avoid `--force-recreate` in production playbooks; reserve for explicit "reset" tags.
- Template the Compose file with `ansible.builtin.template` and use `notify` + handler to trigger `docker compose up -d` only when the file actually changed.
- Pin image tags in the Compose file — `latest` tag causes needless recreations when images update between runs.

**Warning signs:**
- Ansible run output shows "changed" on the docker compose task even when no config changed.
- Container start timestamps show recreation on every playbook run.
- `docker compose ps` shows containers with very recent start times after every playbook execution.

**Phase to address:** Phase 1 — Ansible playbook design

---

### Pitfall 7: Secrets in Ansible Inventory or Plain-Text Variables Files

**What goes wrong:**
Authentik secret key, PostgreSQL passwords, and other credentials end up committed to the Git repository in plain text inside `group_vars/all.yml`, `host_vars/`, or directly in the Compose template. This is a critical security failure for any externally accessible VM.

**Why it happens:**
Getting the stack "working first" leads to hardcoding secrets in variables files. Semaphore makes it easy to store environment variables but developers forget to move secrets out of the repository.

**How to avoid:**
- Use **Ansible Vault** to encrypt the secrets file: `ansible-vault encrypt group_vars/vault.yml`. Store only the vault password in Semaphore (vault password file feature or environment variable `ANSIBLE_VAULT_PASSWORD_FILE`).
- Alternatively: use Semaphore's built-in "Secret Store" for vault passwords and reference them in the project.
- The Compose template should reference `{{ vault_authentik_secret_key }}` — variables loaded from the encrypted vault file.
- Add `group_vars/vault.yml` to `.gitignore` if not using vault encryption (last resort).
- The `.planning/secrets.local` pattern already used in this project is correct — extend it to Ansible vault.

**Warning signs:**
- `git log --all -S "password"` finds secrets in commit history.
- Plain-text passwords visible in `ansible-playbook --list-vars` output.
- Semaphore project variables have secrets stored as "env" type (visible) instead of "secret" type (masked).

**Phase to address:** Phase 1 — secrets management, before first commit of Ansible files

---

### Pitfall 8: Authentik Initial Setup — `AUTHENTIK_SECRET_KEY` Not Set or Changed After First Run

**What goes wrong:**
Authentik's `AUTHENTIK_SECRET_KEY` is used to sign sessions and tokens. If it is not explicitly set, Authentik generates a random one at startup. If the container is recreated (e.g. after a playbook run), a new key is generated, invalidating all existing sessions. Users are logged out and OIDC tokens issued to applications are immediately invalid.

**Why it happens:**
Documentation examples sometimes omit `AUTHENTIK_SECRET_KEY` or show it as optional. Developers assume the generated key persists — but it only persists as long as the container process is alive (not across container recreation).

**How to avoid:**
- Always generate a static `AUTHENTIK_SECRET_KEY` before first deployment: `openssl rand -hex 32`.
- Store it in Ansible Vault and inject it via the docker-compose.yml template.
- Never change it after initial deployment (changing it invalidates all sessions).
- Treat it with the same sensitivity as a signing key.

**Warning signs:**
- All users are logged out every time the Ansible playbook runs.
- OIDC-integrated applications report "invalid token signature" errors after playbook execution.
- Authentik logs show a new secret key generated on startup.

**Phase to address:** Phase 1 — Authentik initial configuration

---

### Pitfall 9: Semaphore SSH Key Handling — Key Not Available at Playbook Execution Time

**What goes wrong:**
Ansible playbooks run in Semaphore fail with `Permission denied (publickey)` when connecting to the target VM, even though the SSH key was added to the Semaphore project. This blocks all deployment automation.

**Why it happens:**
Three common sub-causes:
1. The key is stored in Semaphore as "login with password" type instead of "SSH" type.
2. The key has a passphrase but Semaphore is not configured to decrypt it (passphrase not stored).
3. The Semaphore runner user's `~/.ssh/known_hosts` does not include the target VM's host key, and `StrictHostKeyChecking` is not disabled, causing the first connection to hang waiting for user confirmation.

**How to avoid:**
- Store the SSH key in Semaphore as type "SSH Key" with the passphrase field if the key is passphrase-protected (or use a passphrase-free deployment key).
- Add `ansible_ssh_common_args: '-o StrictHostKeyChecking=no'` in the Semaphore inventory OR add the VM's host key to known_hosts in the Semaphore runner setup (more secure).
- Test SSH connectivity manually from the Semaphore runner host before running playbooks.
- Use a dedicated SSH key for Semaphore deployment (not a personal key) to enable key rotation without affecting developer access.

**Warning signs:**
- Semaphore task log shows `UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: ..."}`.
- Task completes immediately with 0 hosts — inventory is parsed but connection fails silently.
- Repeating the task manually from CLI with the same key works fine (indicates key format or passphrase issue in Semaphore).

**Phase to address:** Phase 1 — Semaphore project setup

---

### Pitfall 10: Traefik Label Precedence — Middleware Applied in Wrong Order or Overridden by Other Labels

**What goes wrong:**
When Authentik forwardAuth middleware is defined but the order of middleware chain application is wrong, requests can bypass authentication. For example: a rate-limit or error-handler middleware defined at router level overrides the auth middleware; or the middleware name collision between multiple services causes one service's auth to silently apply to another.

**Why it happens:**
Traefik middleware is referenced by name and middleware names are namespaced by Docker provider labels. If two containers define a middleware with the same name (e.g. `traefik.http.middlewares.auth.forwardauth.address=...`), one silently overwrites the other. The chain is applied left-to-right but order is not always obvious from labels.

**How to avoid:**
- Name middlewares with service-specific prefixes: `traefik.http.middlewares.traefik-dash-auth.forwardauth.address=...`.
- Define the "global" Authentik forwardAuth middleware once — either in a static Traefik config file or as labels on the Traefik container itself, then reference it from all services by its fully qualified name.
- Use Traefik's dashboard to inspect the actual middleware chain applied to each router — verify it matches expectation before considering auth "done".
- Define the middleware chain explicitly on the router: `traefik.http.routers.myservice.middlewares=authentik-auth@docker,another-middleware@docker`.

**Warning signs:**
- Traefik dashboard shows the router with 0 or unexpected middlewares attached.
- A service is accessible without authentication despite having auth labels.
- A service returns 401/500 even though it shouldn't have auth (middleware name collision).

**Phase to address:** Phase 1 — Traefik label design

---

### Pitfall 11: Docker Compose Service Startup Order — Authentik Depends on PostgreSQL and Redis

**What goes wrong:**
Authentik fails to start or enters a crash-restart loop because PostgreSQL or Redis is not ready when Authentik starts its initialization. The stack starts correctly the first time but fails on cold reboots or playbook-triggered full restarts.

**Why it happens:**
Docker Compose `depends_on` with `condition: service_started` only waits for the container process to start, not for the service inside to be accepting connections. PostgreSQL may take several seconds to complete initialization on first run or after a dirty shutdown.

**How to avoid:**
- Use `depends_on` with `condition: service_healthy` for PostgreSQL and Redis.
- Define `healthcheck` blocks in the Compose file for both:
  - PostgreSQL: `pg_isready -U $POSTGRES_USER`
  - Redis: `redis-cli ping`
- The Authentik worker and server should both list `db` and `redis` in their `depends_on` with `service_healthy` condition.
- Set `restart: unless-stopped` on all Authentik services to self-heal transient startup failures.

**Warning signs:**
- Authentik logs contain `could not connect to server: Connection refused` during startup.
- `docker compose ps` shows Authentik in "Restarting" state shortly after stack start.
- The stack works after manually waiting 30 seconds and running `docker compose restart authentik-server`.

**Phase to address:** Phase 1 — docker-compose.yml design

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Using `image: authentik:latest` instead of pinned version | Always gets newest version | Authentik upgrade breaks config silently on next `docker compose pull` | Never — pin to specific version |
| Defining forwardAuth middleware inline on every service label | Works without shared config | Duplicate configuration; changing the outpost address requires updating every service | Only for single-service proof-of-concept; extract to shared middleware immediately |
| Storing the Ansible vault password in a plain-text file on the runner | Simplifies setup | Vault password file is as sensitive as the secrets themselves | Only on air-gapped runners with physical security |
| HTTP-only (no TLS) for internal services during "testing" | Faster initial setup | Credentials and session tokens transmitted in plaintext; hard to re-enable TLS later | Never on internet-accessible VMs |
| Skipping Authentik PostgreSQL and using SQLite | No PostgreSQL dependency | SQLite not supported for production Authentik; migration is destructive | Never — Authentik requires PostgreSQL |
| Running Traefik with `insecureSkipVerify: true` globally | Bypasses cert errors quickly | Disables all TLS verification including for upstream services | Only during local development; remove before test-VM deployment |

---

## Integration Gotchas

Common mistakes when connecting to external services.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Traefik + Authentik forwardAuth | Pointing `address` to external HTTPS URL | Use internal Docker DNS: `http://authentik-server:9000/outpost.goauthentik.io/...` |
| Authentik + PostgreSQL | Using `AUTHENTIK_POSTGRESQL__HOST=localhost` | Use Docker service name: `AUTHENTIK_POSTGRESQL__HOST=db` (or whatever the service is named) |
| Traefik + Let's Encrypt | Running ACME on a non-internet-accessible test VM | Use DNS-01 challenge instead of HTTP-01; or use a self-signed cert with `tls.certresolver` disabled for internal domains |
| Semaphore + Ansible Vault | Not setting `ANSIBLE_VAULT_PASSWORD_FILE` env in Semaphore | Add the vault password as a Semaphore "Secret" and reference it via environment variable in the project settings |
| Authentik outpost + Traefik | Not adding `X-Forwarded-For` trust settings | Set `AUTHENTIK_LISTEN__PROXY_TRUSTED_CIDRS` to include the Docker network CIDR |
| Ansible + Docker Compose v2 | Using `docker-compose` (v1, hyphenated) in shell tasks | Use `docker compose` (v2, space) or the `community.docker.docker_compose_v2` module; v1 is deprecated and absent from modern Docker installs |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Authentik embedded outpost for all services | Works fine for 2-3 apps | The embedded outpost has limited concurrency; becomes a bottleneck | > ~20 concurrent auth checks; deploy a dedicated proxy outpost container instead |
| Traefik dashboard with no request rate limit | No issue at 1-2 admins | Dashboard endpoint is expensive; bots/scanners generate load if forwardAuth misconfiguration allows access | External scanning; always apply rate-limit middleware to dashboard router |
| Using `AUTHENTIK_LOG_LEVEL=debug` permanently | Helpful during setup | Log volume causes disk fill on active instances | Any moderate usage; use `info` level for production/test environments |

---

## Security Mistakes

Domain-specific security issues beyond general web security.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Traefik API dashboard exposed on port 8080 without auth | Anyone on the network sees all routes, middleware, certs | Disable the insecure dashboard: set `api.insecure: false`; access only via Traefik-routed+Authentik-protected path |
| Authentik admin interface accessible without restricting source IP | Full user/group management exposed to internet | Use Authentik's own access policy to restrict admin interface to known IPs, or use Traefik IP whitelist middleware |
| `AUTHENTIK_SECRET_KEY` shorter than 32 bytes | Sessions can potentially be forged | Always generate with `openssl rand -hex 32` (produces 64 hex chars = 32 bytes) |
| PostgreSQL port 5432 bound to `0.0.0.0` in Compose | Database exposed to all interfaces including public | In Compose, bind DB to internal network only — do not publish port 5432 externally; use `expose` not `ports` |
| Redis port 6379 bound to `0.0.0.0` without password | In-memory data and queue accessible without auth | Same as PostgreSQL — keep Redis on internal network only; set `requirepass` if port must be published |
| Ansible playbook runs as `root` on target VM | Full system compromise if playbook is malicious or misconfigured | Use a dedicated deploy user with passwordless sudo only for specific commands, or Docker socket access only |
| Traefik missing `X-Frame-Options` and `X-Content-Type-Options` headers on dashboard | Potential clickjacking on the dashboard UI | Add a security headers middleware to the dashboard router |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Traefik HTTPS:** Traefik is running and routing — but verify TLS termination is actually working: `curl -v https://[domain]` must show a valid certificate, not a self-signed one and not HTTP.
- [ ] **Authentik Login:** Authentik login page loads — but verify the session cookie is set with the correct domain and that a second browser without the cookie is correctly redirected (not granted access).
- [ ] **forwardAuth on Traefik Dashboard:** Dashboard is behind Authentik label — but verify a request WITHOUT a valid session actually redirects to login (does not 401 or bypass). Test in an incognito window.
- [ ] **Ansible Idempotency:** Playbook runs once successfully — but verify it runs a SECOND time with `changed=0` on all tasks except potentially `Gathering Facts`.
- [ ] **Authentik Groups:** Groups "Admin", "Editor", "Viewer" exist — but verify that a test user in each group can access only the correct resources (group bindings applied to applications/policies, not just created).
- [ ] **Secret Rotation Safety:** Authentik secret key is set — but verify it is in Ansible Vault, not in the Compose template in plain text, and not in `git log`.
- [ ] **Service Restart Behavior:** Stack starts cleanly — but verify it recovers after `docker compose down && docker compose up -d` without manual intervention (healthchecks, depends_on ordering).
- [ ] **Semaphore Reproducibility:** One successful Semaphore run — but verify the run works from a clean state (delete and re-add the project; run from scratch) to confirm no implicit state dependency.

---

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Authentik secret key regenerated (sessions invalidated) | LOW | Users must re-login; no data lost. Set static key in vault and redeploy. |
| Cookie domain mismatch (auth loop) | LOW | Stop Authentik, correct `AUTHENTIK_COOKIE_DOMAIN`, restart. Browser: clear cookies and retry. |
| Wrong network (outpost unreachable) | LOW | Add missing network to the affected services in Compose file; `docker compose up -d` re-attaches containers to correct networks without full teardown. |
| Secrets committed to Git | HIGH | Rotate ALL committed secrets immediately. Use `git filter-repo` or BFG to purge history. Force-push. Notify affected parties. |
| PostgreSQL not reachable on startup (crash loop) | LOW | Add healthcheck + `depends_on: service_healthy`; `docker compose restart authentik-server authentik-worker`. |
| Traefik dashboard accessible without auth (middleware misconfiguration) | MEDIUM | Immediately add Traefik dashboard behind firewall or `api.insecure: false`; fix label; redeploy. Audit access logs for unauthorized access. |
| Ansible runs non-idempotent (containers recreated each run) | MEDIUM | Refactor to `community.docker.docker_compose_v2` with `state: present`; audit template for non-deterministic rendering (timestamps, random values). |
| Semaphore SSH connection fails | LOW | Add target VM host key to known_hosts on runner; verify key type matches (ed25519 vs rsa); test manual SSH from runner. |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Outpost not on shared Docker network | Phase 1: Docker network design | `docker network inspect proxy` shows both Traefik and Authentik containers |
| TLS redirect loop on forwardAuth address | Phase 1: Traefik middleware config | `curl -v http://authentik-server:9000/outpost.goauthentik.io/...` from Traefik container resolves without redirect |
| Missing `authResponseHeaders` | Phase 1: forwardAuth middleware definition | Protected service receives `X-authentik-username` header on authenticated request |
| Cookie domain mismatch | Phase 1: Authentik env vars | Login in one tab; open protected service in new tab — no re-login prompt |
| Proxy provider vs OIDC confusion | Phase 1: Authentik provider/application setup | Traefik dashboard shows login redirect, not 401 or open access |
| Ansible non-idempotency | Phase 1: Ansible playbook design | Two consecutive Semaphore runs both show `changed=0` |
| Secrets in plain text | Phase 1: Ansible vault setup (before first commit) | `git grep -i "password\|secret\|key"` returns no plain-text credentials |
| `AUTHENTIK_SECRET_KEY` not set | Phase 1: Authentik initial config | Container recreation does not log out existing users |
| Semaphore SSH key handling | Phase 1: Semaphore project setup | Semaphore task connects to VM without manual intervention |
| Traefik label name collision | Phase 1: Traefik label naming convention | Traefik dashboard shows correct middleware chain for each router |
| Container startup ordering | Phase 1: docker-compose.yml healthchecks | `docker compose down && docker compose up -d` recovers fully without manual restart |

---

## Sources

- Training data (Traefik documentation v2.x/v3.x patterns, Authentik documentation through 2025, Ansible community.docker collection documentation, Semaphore UI documentation) — MEDIUM confidence
- Authentik forward auth integration pattern: well-established, multiple community deployments documented in public GitHub issues and blog posts through training cutoff
- Traefik forwardAuth behavior: documented in official Traefik middleware docs; `trustForwardHeader` and `authResponseHeaders` behavior is stable across v2 and v3
- Ansible `community.docker.docker_compose_v2` idempotency: documented in Ansible Galaxy collection; v1 vs v2 `docker-compose` binary distinction is a well-known migration issue
- Semaphore SSH key handling: documented in Semaphore project docs and GitHub issues
- NOTE: WebSearch and WebFetch were unavailable during this research session. Verify against official documentation before implementation, particularly for Authentik version-specific header names and Semaphore vault password configuration.

---
*Pitfalls research for: Traefik + Authentik + Docker Compose + Ansible/Semaphore (KB-Pipeline Phase 1)*
*Researched: 2026-03-27*
