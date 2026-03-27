# KB-Pipeline

## What This Is

KB-Pipeline ist eine Web-Anwendung, die PowerPoint-Präsentationen automatisch in eine strukturierte, durchsuchbare Knowledge Base umwandelt. Präsentationen werden über einen Docker-geteilten Ordner auf der VM oder direkt per UI-Upload bereitgestellt, in einzelne Slides zerlegt, mit Metadaten angereichert, per AI ins Englische übersetzt und als navigierbare Wissenssammlung nach Confluence publiziert. Anwender können aus dem Slide-Bestand neue Präsentationen zusammenstellen und als PPTX herunterladen. Alle AI-Operationen (Übersetzung, Neuformulierung, Slide-Generierung) sind in einen eigenständigen MCP-Server ausgelagert, der sowohl von der App als auch aus Claude Code / Claude Desktop nutzbar ist.

## Core Value

Hunderte Unternehmens-Präsentationen werden von einem unzugänglichen Datei-Silo zu einer einheitlichen, durchsuchbaren und wiederverwendbaren Knowledge Base — sodass niemand mehr eine Folie suchen oder doppelt erstellen muss.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Phase 1 — Infrastruktur (Traefik + Authentik)**

- [ ] Traefik läuft als Reverse Proxy auf der Test-VM und terminiert HTTPS
- [ ] Authentik läuft und ist über `[DOMAIN]/authentik` erreichbar
- [ ] Traefik Dashboard ist über `[DOMAIN]/traefik` erreichbar — geschützt durch Authentik Forward Auth
- [ ] Authentik: Gruppe **Admin** angelegt (Rechte: System-Konfiguration — Taxonomie, Zielgruppen, Templates, PPTX-Quellen, Authentik-Rollen, Export-Konfiguration, MCP-Server Einstellungen)
- [ ] Authentik: Gruppe **Editor** angelegt (Rechte: Slides anreichern, AI-Verarbeitung starten/reviewen, BG-Shapes prüfen, Präsentationen bauen, Exports starten)
- [ ] Authentik: Gruppe **Viewer** angelegt (Rechte: Knowledge Base durchsuchen, Slides browsen, Präsentationen herunterladen)
- [ ] Je ein Testnutzer pro Gruppe angelegt und funktionsfähig
- [ ] Deployment via Ansible Playbook über Semaphore reproduzierbar und idempotent

**Spätere Phasen (Backlog)**

- [ ] FastAPI Backend (REST API + SSE)
- [ ] Angular Frontend (PrimeNG)
- [ ] PostgreSQL mit Full-Text Search
- [ ] Redis Queue + Celery Workers + Celery Beat
- [ ] Gotenberg (PPTX → JPG)
- [ ] MCP AI-Server (SSE Transport): translate_text, reformulate_text, generate_slide_content
- [ ] MinIO (Object Storage)
- [ ] Confluence-Export

### Out of Scope

- Kubernetes / k3s — bewusste Entscheidung für Docker Compose; kein Cluster-Overhead für diese Projekt-Größe
- Produktions-Deployment — Phase 1 zielt ausschließlich auf die Test-Umgebung
- Vollständiger App-Stack in Phase 1 — nur Traefik + Authentik; restliche Services folgen in späteren Phasen

## Context

### Technische Architektur (Zielbild)

```
┌──────────────────────────────────────────────────┐
│              Traefik (Reverse Proxy)              │
└────┬──────────────┬──────────────┬───────────────┘
     │              │              │
  Angular        Authentik      FastAPI
  (PrimeNG)      (OIDC/SSO)    (REST API + SSE)
     │              │              │
     └──────────────┴──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   PostgreSQL           Redis            Celery Workers
   (+ Full-Text         (Queue)          + Celery Beat
    Search)                              (Scheduler)
                           │
              ┌────────────┼────────────┐
              │            │            │
          Gotenberg    MCP AI-Server  MinIO
          (PPTX→JPG)   (SSE Transport) (Object Storage)
                       ├─ translate_text
                       ├─ reformulate_text
                       └─ generate_slide_content
```

### Deployment-Infrastruktur

- **Test-VM:** Existiert, Domain bekannt
- **Deployment-Tool:** Semaphore UI — `https://semaphore.home-engelhardt.uk`
- **Automation:** Ansible Playbooks (von Semaphore ausgeführt)
- **Secrets:** Semaphore API Key in `.planning/secrets.local` (gitignored)
- **Container-Runtime:** Docker Compose

### Nutzerrollen

| Rolle | Berechtigungen |
|-------|----------------|
| Admin | System-Konfiguration: Taxonomie, Zielgruppen, Templates, PPTX-Quellen, Authentik-Rollen, Export-Konfiguration, MCP-Server Einstellungen |
| Editor | Slides anreichern, AI-Verarbeitung starten und reviewen, BG-Shapes überprüfen, Präsentationen bauen, Exports starten |
| Viewer | Knowledge Base durchsuchen, Slides browsen, Präsentationen herunterladen |

## Constraints

- **Tech Stack:** Docker Compose — kein Kubernetes; bewusste Entscheidung für Einfachheit
- **Deployment:** Ansible via Semaphore — kein direktes SSH auf die VM im CI-Prozess
- **Phase 1 Scope:** Nur Traefik + Authentik; kein App-Code in dieser Phase
- **Sprache:** UI-Beschreibungstexte werden per AI ins Englische übersetzt (MCP-Server)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Docker Compose statt Kubernetes | Kein Cluster-Overhead für Projekt-Größe; einfacheres Ops | — Pending |
| Ansible via Semaphore | Reproduzierbares, auditierbares Deployment ohne direkten VM-Zugriff | — Pending |
| Authentik als OIDC/SSO | Zentrales IAM für alle Services; Traefik Forward Auth Integration | — Pending |
| MCP-Server als eigenständiger Service | AI-Operationen von App-Code entkoppelt; direkt aus Claude Code nutzbar | — Pending |
| Traefik Dashboard hinter Authentik | Kein öffentlicher Zugriff auf Infrastruktur-UIs | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-27 after initialization*
