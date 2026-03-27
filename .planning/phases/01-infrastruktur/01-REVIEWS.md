---
phase: 1
reviewers: [claude-self]
reviewed_at: 2026-03-27T00:00:00Z
plans_reviewed: [01-01-PLAN.md, 01-02-PLAN.md, 01-03-PLAN.md]
note: codex unavailable (no OpenAI API key), gemini not installed, claude is current runtime — self-review performed
user_feedback_addressed:
  - "Redis not needed for current Authentik" — investigated; Redis still required but Valkey preferred
  - "Traefik 3.x version" — confirmed traefik:v3.3 is already pinned; v3 syntax verified with one concern
---

# Cross-AI Plan Review — Phase 1: Infrastruktur

> **Reviewer availability:** `gemini` not installed, `codex` missing OpenAI API key, `claude` is the current runtime.
> Self-review performed by Claude incorporating the user's two specific concerns.

---

## Review (Claude — self)

### 1. Summary

The Phase 1 plans are well-structured and demonstrate strong knowledge of the Traefik/Authentik stack. The decision to use file-provider middleware (not Docker labels) for ForwardAuth, the explicit circular-auth prevention (no ForwardAuth on the Authentik router), network isolation (`authentik_internal: true`), and vault-based secrets handling are all production-grade choices. The two user-flagged concerns are partially valid: **Traefik v3.3 is correct and already pinned**, but the **Redis dependency question is nuanced** — Authentik 2024.12.3 still requires Redis/Valkey externally (it is not bundled), however the plans should use **Valkey** instead of Redis to align with Authentik's upstream trajectory since 2024.8+. One minor Traefik v3 configuration concern exists around the entrypoint redirect key casing.

---

### 2. Strengths

- **Traefik v3.3 correctly pinned** — `traefik:v3.3` is an appropriate stable choice; config syntax is v3-compatible throughout
- **Circular auth prevention** — Authentik router labels explicitly omit ForwardAuth middleware; this is the most common failure mode and correctly prevented
- **File-provider middleware** — ForwardAuth defined in file provider (not Docker labels) ensures middleware is always available
- **Network isolation** — `authentik_internal: internal: true` correctly isolates PostgreSQL and Redis from external access
- **`traefik.docker.network=proxy` label** — Critical for multi-network containers; correctly included on authentik-server
- **`AUTHENTIK_LISTEN__PROXY_TRUSTED_CIDRS`** — Correctly set for X-Forwarded-For trust from Docker networks
- **Idempotency strategy** — `pull: missing`, `state: present` with `recreate: auto`, `creates:` guards — solid
- **Secrets management** — Ansible Vault with placeholder scaffold and explicit encryption steps; hygiene checks built into plans
- **Health dependency chain** — `depends_on: condition: service_healthy` on both PostgreSQL and Redis with proper healthchecks
- **`AUTHENTIK_COOKIE_DOMAIN`** — Apex domain without leading dot is correct for Authentik cookie behavior

---

### 3. Concerns

#### HIGH

**Redis vs Valkey for Authentik 2024.12.3**
The user's concern that "Redis is not needed" is **incorrect** — Authentik 2024.12.3 still requires an external Redis/Valkey instance (not bundled). However, Authentik began migrating from Redis to Valkey (Redis fork) starting with version **2024.8.0**. For 2024.12.3, the upstream documentation recommends `valkey/valkey:8-alpine` instead of `redis:7-alpine`. Using Redis 7 still works (Valkey is API-compatible), but it diverges from upstream recommendations.

**Required changes in `ansible/roles/authentik/templates/docker-compose.yml.j2`:**
```yaml
# Change:
  redis:
    image: docker.io/library/redis:7-alpine
    container_name: authentik-redis
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

# To:
  valkey:
    image: docker.io/valkey/valkey:8-alpine
    container_name: authentik-valkey
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
```
And update `AUTHENTIK_REDIS__HOST: redis` → `AUTHENTIK_REDIS__HOST: valkey`, plus all `redis:` → `valkey:` references in `depends_on`.

---

**Traefik v3 redirect key casing** *(potential silent misconfiguration)*
In Traefik v3, the HTTP redirect key under `entryPoints` changed capitalization:
```yaml
# v2 / potentially broken in v3:
redirections:
  entryPoint:     # capital P
    to: websecure

# v3 correct:
redirections:
  entrypoint:     # all lowercase
    to: websecure
```
The plan uses `entryPoint:` (capital P). This may silently fail to apply the redirect in Traefik v3, breaking requirement TRFK-03. **Verify against Traefik v3 documentation before executing.**

---

#### MEDIUM

**`authentik-worker` runs as `user: root`**
The worker mounts `/var/run/docker.sock` and runs as root for embedded outpost management. This is Authentik's recommended approach for the embedded outpost but is a security consideration. Acceptable for a test VM; document as known trade-off.

**No `become_password` handling documented**
The playbook uses `become: true` but Plan 03 doesn't explicitly address whether the `deploy` user has passwordless sudo. If not, the Semaphore task template needs a become password configuration.

#### LOW

- **`traefik:v3.3`** — May not be the latest 3.x patch by execution time; consider checking for a newer stable release.
- **Traefik dashboard at `/traefik` subpath** — The dashboard SPA may fail to load CSS/JS without `api.basepath: /traefik` in the static config. Test asset loading after deployment.
- **AUTH-03/AUTH-05-08 are manual** — Consider adding `AUTHENTIK_BOOTSTRAP_PASSWORD` env var to automate initial admin setup.

---

### 4. Suggestions

1. **Replace Redis with Valkey** — Update the Authentik Compose template to use `valkey/valkey:8-alpine`, rename the service to `valkey`, update the healthcheck to `valkey-cli ping`, and update `AUTHENTIK_REDIS__HOST` to `valkey`. Also update `redis_data` volume name to `valkey_data`.

2. **Verify Traefik v3 redirect casing** — Before executing Plan 01-01, confirm whether `entryPoint:` or `entrypoint:` is correct for Traefik v3. The failing redirect would not crash Traefik but would leave HTTP endpoints accessible, violating TRFK-03.

3. **Add `api.basepath`** — In `traefik.yml.j2`, consider:
   ```yaml
   api:
     dashboard: true
     insecure: false
     basepath: /traefik
   ```
   This helps Traefik serve dashboard assets correctly at the subpath.

4. **Bootstrap automation** — Add optional Authentik bootstrap vars:
   ```yaml
   AUTHENTIK_BOOTSTRAP_PASSWORD: "{{ vault_authentik_bootstrap_password }}"
   AUTHENTIK_BOOTSTRAP_EMAIL: "admin@{{ domain }}"
   ```
   Reduces manual post-deploy steps for AUTH-03.

5. **Clarify `expose:` vs network isolation** — The interfaces block mentions `expose:` but the actual template uses network isolation only. This is correct but the comment should be clarified to avoid confusion.

---

### 5. Risk Assessment: **MEDIUM**

The plan is technically sound with strong security practices. Two issues could cause problems at deployment:

1. **Redis→Valkey**: Works with Redis 7, but diverges from upstream Authentik 2024.8+ recommendations. Low operational risk; medium alignment risk.
2. **Traefik v3 redirect casing**: Could silently break HTTPS-only enforcement (TRFK-03). Medium operational risk.

With these two issues addressed, the plan is ready to execute. The highest-risk concern (circular ForwardAuth loop) is correctly handled.

---

## Consensus Summary

*(Single reviewer — no consensus applicable)*

### Key Findings

1. **Redis is still required** — but should be replaced with Valkey for Authentik 2024.12.3 alignment
2. **Traefik v3.3 is correctly used** — one potential syntax issue with redirect key casing to verify
3. **All other plan decisions are sound** — circular auth prevention, network isolation, secrets management, idempotency patterns

### Action Items Before Execution

| Priority | Action | Plan |
|----------|--------|------|
| HIGH | Replace `redis:7-alpine` with `valkey/valkey:8-alpine` in Authentik Compose template | 01-02 |
| HIGH | Verify `entryPoint:` vs `entrypoint:` casing in Traefik v3 redirect config | 01-01 |
| MEDIUM | Add `api.basepath: /traefik` to Traefik static config | 01-01 |
| LOW | Consider Authentik bootstrap env vars for AUTH-03 automation | 01-02 |
