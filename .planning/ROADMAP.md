# Roadmap: KB-Pipeline

## Overview

KB-Pipeline transforms an unstructured PowerPoint file silo into a searchable, reusable knowledge base. The journey starts with a hardened infrastructure layer (Traefik + Authentik, deployed via Ansible from Semaphore), then layers in the FastAPI backend with OIDC, the Angular frontend, the full processing pipeline (Gotenberg, MCP AI-Server, MinIO, Celery), Confluence export and PPTX download, and finally an observability stack. Each phase delivers a coherent, independently verifiable capability that the next phase builds on.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Infrastruktur** - Traefik + Authentik deployed via Ansible/Semaphore; HTTPS routing and ForwardAuth protecting all routes; idempotent from day one (completed 2026-03-27)
- [ ] **Phase 2: Backend & Auth Integration** - FastAPI REST API behind Traefik + Authentik OIDC; PostgreSQL, Redis, Celery workers operational
- [ ] **Phase 3: Frontend Application** - Angular + PrimeNG UI behind Traefik; OIDC login flow; users can navigate and search the knowledge base
- [ ] **Phase 4: Processing Pipeline** - Gotenberg PPTX-to-image conversion, MCP AI-Server, MinIO object storage, Confluence export, PPTX download
- [ ] **Phase 5: Observability** - Prometheus + Grafana consuming pre-enabled Traefik metrics; dashboards for all services

## Phase Details

### Phase 1: Infrastruktur
**Goal**: The infrastructure layer is live, secured, and reproducibly deployable — Traefik terminates HTTPS, Authentik protects every route, and the Ansible playbook runs idempotently from Semaphore with zero manual steps after the first run
**Depends on**: Nothing (first phase)
**Requirements**: TRFK-01, TRFK-02, TRFK-03, TRFK-04, TRFK-05, TRFK-06, AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, AUTH-06, AUTH-07, AUTH-08, DEPL-01, DEPL-02, DEPL-03, DEPL-04, DEPL-05
**Success Criteria** (what must be TRUE):
  1. Visiting any HTTP URL on the domain redirects to HTTPS with a valid TLS certificate — no browser trust error
  2. Visiting `[DOMAIN]/traefik` prompts an Authentik login; after login the Traefik dashboard is visible and no unauthenticated access is possible
  3. Visiting `[DOMAIN]/authentik` shows the Authentik admin UI; all three groups (Admin, Editor, Viewer) exist with one verified test user each
  4. Running the Semaphore pipeline a second time produces `changed=0` — the playbook is fully idempotent
  5. No secrets (passwords, secret keys) are present in plain text in the Git repository
**Plans**: TBD

Plans:
- [x] 01-01: Docker engine + Traefik role (proxy network, static/dynamic config, HTTPS, dashboard)
- [x] 01-02: Authentik role (PostgreSQL, Redis, server + worker, ForwardAuth provider, groups and test users)
- [x] 01-03: Ansible vault, Semaphore pipeline setup, idempotency verification

### Phase 2: Backend & Auth Integration
**Goal**: The FastAPI backend is reachable behind Traefik, authenticates users via Authentik OIDC, and exposes a working REST API backed by PostgreSQL with full-text search and Celery for async task processing
**Depends on**: Phase 1
**Requirements**: BACK-01, BACK-02, BACK-03, BACK-04
**Success Criteria** (what must be TRUE):
  1. The FastAPI REST API is reachable at `[DOMAIN]/api` and returns authenticated responses — unauthenticated requests are rejected
  2. The Authentik OIDC provider is configured with a live FastAPI client; token exchange works end-to-end
  3. PostgreSQL full-text search returns slide results for a test query
  4. A Celery task submitted via the API is picked up by a worker and completes — visible in task result or log
**Plans**: TBD

Plans:
- [ ] 02-01: FastAPI service, Traefik routing, Authentik OIDC provider + app client
- [ ] 02-02: PostgreSQL full-text search schema, Redis queue, Celery workers + Celery Beat

### Phase 3: Frontend Application
**Goal**: Users can open the Angular app in a browser, log in via Authentik OIDC, and navigate the knowledge base — the UI is served behind Traefik with the same ForwardAuth pattern established in Phase 1
**Depends on**: Phase 2
**Requirements**: FRONT-01, FRONT-02
**Success Criteria** (what must be TRUE):
  1. Visiting `[DOMAIN]/app` serves the Angular application; unauthenticated users are redirected to the Authentik login page
  2. After login, users land on the knowledge base view and can browse slides
  3. The OIDC token received from Authentik is used for authenticated API calls — no separate login prompt after SSO
**Plans**: TBD

Plans:
- [ ] 03-01: Angular + PrimeNG build, Traefik routing, OIDC login flow
**UI hint**: yes

### Phase 4: Processing Pipeline
**Goal**: Uploaded PowerPoint files are fully processed — converted to slide images by Gotenberg, stored in MinIO, enriched and translated by the MCP AI-Server, and exportable to Confluence or downloadable as a new PPTX
**Depends on**: Phase 3
**Requirements**: PROC-01, PROC-02, PROC-03, EXPO-01, EXPO-02
**Success Criteria** (what must be TRUE):
  1. A PPTX uploaded via the UI is converted to per-slide JPG images by Gotenberg and stored in MinIO — images are viewable in the knowledge base
  2. The MCP AI-Server translates slide text to English and reformulates content via its three tools (`translate_text`, `reformulate_text`, `generate_slide_content`) — callable from both the app and Claude Code
  3. An Editor can trigger a Confluence export and verify the published Knowledge Base page in Confluence
  4. A Viewer can select slides, compose a new presentation, and download it as a PPTX file
**Plans**: TBD

Plans:
- [ ] 04-01: Gotenberg integration + MinIO object storage + Celery processing pipeline
- [ ] 04-02: MCP AI-Server (SSE transport, three tools), Confluence export, PPTX download

### Phase 5: Observability
**Goal**: Operators can see the health and performance of all services in Grafana; Traefik metrics (pre-enabled in Phase 1) are collected by Prometheus and visualized — no guesswork when diagnosing issues
**Depends on**: Phase 1
**Requirements**: OBS-01
**Success Criteria** (what must be TRUE):
  1. The Grafana dashboard is accessible at `[DOMAIN]/grafana` and protected by Authentik ForwardAuth
  2. Traefik request rate, error rate, and latency are visible on a Grafana dashboard populated from Prometheus
  3. A simulated spike in traffic produces a visible change in Grafana metrics within 30 seconds
**Plans**: TBD

Plans:
- [ ] 05-01: Prometheus + Grafana deployment, Traefik metrics scrape config, dashboards
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 (Phase 5 can start after Phase 1)

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Infrastruktur | 3/3 | Complete   | 2026-03-27 |
| 2. Backend & Auth Integration | 0/2 | Not started | - |
| 3. Frontend Application | 0/1 | Not started | - |
| 4. Processing Pipeline | 0/2 | Not started | - |
| 5. Observability | 0/1 | Not started | - |
