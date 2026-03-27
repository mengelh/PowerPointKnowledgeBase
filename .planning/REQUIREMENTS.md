# Requirements: KB-Pipeline

**Defined:** 2026-03-27
**Core Value:** Hunderte Unternehmens-Präsentationen werden von einem unzugänglichen Datei-Silo zu einer einheitlichen, durchsuchbaren und wiederverwendbaren Knowledge Base — sodass niemand mehr eine Folie suchen oder doppelt erstellen muss.

---

## v1 Requirements (Phase 1 — Infrastruktur)

### Traefik

- [ ] **TRFK-01**: Traefik läuft als Reverse Proxy mit Docker-Provider (`exposedByDefault: false`); Services opt-in per Label
- [ ] **TRFK-02**: TLS über Self-signed-Zertifikat auf der Test-VM; alle Routen HTTPS-only
- [ ] **TRFK-03**: HTTP → HTTPS Redirect aktiv — kein Plaintext-Zugriff möglich
- [ ] **TRFK-04**: Traefik Dashboard erreichbar unter `[DOMAIN]/traefik`, geschützt durch Authentik ForwardAuth
- [ ] **TRFK-05**: Traefik Access Logs in JSON-Format aktiviert
- [ ] **TRFK-06**: Prometheus Metrics Endpoint aktiviert (für späteres Monitoring bereit)

### Authentik

- [ ] **AUTH-01**: Authentik Server + Worker laufen stabil; PostgreSQL und Valkey als Dependencies mit `service_healthy` Healthchecks
- [ ] **AUTH-02**: Authentik erreichbar unter `[DOMAIN]/authentik` via `AUTHENTIK_WEB__PATH` Umgebungsvariable
- [ ] **AUTH-03**: Authentik Initial-Setup abgeschlossen — Admin-Superuser erstellt, initiales Passwort geändert
- [ ] **AUTH-04**: Authentik ForwardAuth Provider + Embedded Outpost konfiguriert; Traefik kann Authentik als Auth-Backend nutzen
- [ ] **AUTH-05**: Gruppe **Admin** in Authentik angelegt (RBAC: System-Konfiguration — Taxonomie, Zielgruppen, Templates, PPTX-Quellen, Authentik-Rollen, Export-Konfiguration, MCP-Server Einstellungen)
- [ ] **AUTH-06**: Gruppe **Editor** in Authentik angelegt (RBAC: Slides anreichern, AI-Verarbeitung starten/reviewen, BG-Shapes prüfen, Präsentationen bauen, Exports starten)
- [ ] **AUTH-07**: Gruppe **Viewer** in Authentik angelegt (RBAC: Knowledge Base durchsuchen, Slides browsen, Präsentationen herunterladen)
- [ ] **AUTH-08**: Je ein Testnutzer pro Gruppe angelegt; Login-Flow für jeden Nutzer end-to-end verifiziert (Authentik → ForwardAuth → geschützte Route)

### Deployment

- [ ] **DEPL-01**: Docker Compose Healthchecks auf allen Services (PostgreSQL, Redis, Authentik-Server, Traefik) — `depends_on` mit `condition: service_healthy`
- [ ] **DEPL-02**: Ansible Playbook deployt den vollen Stack (Traefik + Authentik) von Null auf einer frischen VM reproduzierbar
- [ ] **DEPL-03**: Ansible Playbook ist idempotent — zweiter Durchlauf via Semaphore zeigt keine Änderungen
- [ ] **DEPL-04**: Secrets (`AUTHENTIK_SECRET_KEY`, DB-Passwort, weitere Credentials) in Ansible Vault verschlüsselt — nicht im Git-Repo im Klartext
- [ ] **DEPL-05**: Semaphore-Pipeline führt das Ansible Playbook erfolgreich gegen die Test-VM aus; Ergebnis in Semaphore-UI verifiziert

---

## v2 Requirements (Backlog — spätere Phasen)

### Backend

- **BACK-01**: FastAPI REST API + SSE läuft hinter Traefik
- **BACK-02**: Authentik OIDC Provider konfiguriert für FastAPI (kein OIDC-Setup ohne App-Client in Phase 1)
- **BACK-03**: PostgreSQL mit Full-Text Search für Slides
- **BACK-04**: Redis Queue + Celery Workers + Celery Beat (Task-Scheduler)

### Frontend

- **FRONT-01**: Angular + PrimeNG Frontend hinter Traefik
- **FRONT-02**: OIDC-Login-Flow in Angular (via Authentik)

### Processing

- **PROC-01**: Gotenberg (PPTX → JPG Konvertierung)
- **PROC-02**: MCP AI-Server (SSE Transport): `translate_text`, `reformulate_text`, `generate_slide_content`
- **PROC-03**: MinIO Object Storage für Slide-Bilder und Medien

### Export

- **EXPO-01**: Confluence-Export der Knowledge Base
- **EXPO-02**: PPTX-Download zusammengestellter Präsentationen

### Observability

- **OBS-01**: Grafana + Prometheus Monitoring Stack (Traefik Metrics bereits aktiviert in Phase 1)

---

## Out of Scope

| Feature | Reason |
|---------|--------|
| Authentik OIDC Provider (Phase 1) | Kein App-Client vorhanden — unverifizierbarer Config-Drift; kommt mit FastAPI-Phase |
| Authentik LDAP Outpost | Nicht geplant; ForwardAuth + OIDC deckt alle Anwendungsfälle ab |
| Let's Encrypt / ACME | Test-VM nutzt Self-signed Zertifikat; kein öffentlicher Port 80/443 für ACME benötigt |
| Automatische Container-Updates (Watchtower) | Bricht Ansible-Idempotenz; stattdessen kontrollierte Updates via Playbook |
| Kubernetes / k3s | Kein Cluster-Overhead; Docker Compose ist ausreichend für diese Projekt-Größe |
| Grafana + Prometheus Stack (Phase 1) | Dedizierte Observability-Phase; Metrics Endpoint wird vorbereitet, aber nicht konsumiert |
| Traefik TCP/UDP Routing | Kein Service nutzt Raw-TCP/UDP; unnötige Attack Surface |
| Authentik Multi-Tenancy | Single-Domain-Betrieb; kein Nutzen |
| External Managed PostgreSQL | Phase 1 ist Test-VM; containerisiertes PostgreSQL ausreichend |
| Produktions-Deployment | Phase 1 zielt ausschließlich auf die Test-Umgebung |

---

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| TRFK-01 | Phase 1 | Pending |
| TRFK-02 | Phase 1 | Pending |
| TRFK-03 | Phase 1 | Pending |
| TRFK-04 | Phase 1 | Pending |
| TRFK-05 | Phase 1 | Pending |
| TRFK-06 | Phase 1 | Pending |
| AUTH-01 | Phase 1 | Pending |
| AUTH-02 | Phase 1 | Pending |
| AUTH-03 | Phase 1 | Pending |
| AUTH-04 | Phase 1 | Pending |
| AUTH-05 | Phase 1 | Pending |
| AUTH-06 | Phase 1 | Pending |
| AUTH-07 | Phase 1 | Pending |
| AUTH-08 | Phase 1 | Pending |
| DEPL-01 | Phase 1 | Pending |
| DEPL-02 | Phase 1 | Pending |
| DEPL-03 | Phase 1 | Pending |
| DEPL-04 | Phase 1 | Pending |
| DEPL-05 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 19 total
- Mapped to phases: 19
- Unmapped: 0 ✓

---
*Requirements defined: 2026-03-27*
*Last updated: 2026-03-27 after initial definition*
