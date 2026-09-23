# life-ops-platform — Bauplan

> Ursprünglicher Plan aus der Entstehung des Projekts, nicht der Ist-Zustand.
> Abweichungen: `agentic-rogue-like` wurde kein integrierter Dienst, sondern nur
> Ideengeber für den Validierungs-Loop. Den aktuellen Stand beschreibt die
> [README](../README.md).

Dach über drei bestehende Repos: ai-trip-planner, job-application-skill,
agentic-rogue-like. Zwei Zugänge (Claude per MCP, n8n per REST) zu drei
Fachdiensten, die eine RAG-Basis, eine Postgres, ein Langfuse-Projekt teilen.

## Entscheidungen
- MCP nur für Claude (Modell entdeckt Tools live); REST für n8n (Workflows sind
  vordefiniert, kein MCP-Overhead nötig).
- Eine RAG-Instanz mit `collection`-Feld (travel/jobs) statt drei Services.
- Encounter-Agent liefert ein Pattern (LangGraph-Retry), kein wörtliches Tool.
- Eigenes Meta-Repo statt Erweiterung eines bestehenden Fachprojekts.

## Phase 0 — Grundgerüst
docker-compose.yml mit n8n (Image von n8n.io, kein eigener Build). Referenziert
ai-trip-planners fertige GHCR-Images statt lokalem Checkout. Fertig, wenn n8n
lokal läuft und per HTTP-Request-Node einen bestehenden ai-trip-planner-Endpunkt
erfolgreich aufruft.

## Phase 1 — Bewerbungshelfer als Dienst
job-application-skill bekommt REST-Endpunkte (Vorbild: apps/api) für
erstelle_lebenslauf.py/setup_bewerbungsordner.py, plus MCP-Server nach exaktem
Vorbild packages/mcp-server (registerTools, stdio+HTTP, Bearer-Auth).
Tools: match_job_posting, draft_cover_letter, list_applications.
Fertig, wenn Claude Desktop darüber einen Anschreiben-Entwurf erzeugt.

## Phase 2 — RAG um Collections erweitern
Document-Tabelle bekommt `collection`-Feld; rag-ingest bekommt --collection-Flag;
neues Tool search_career_knowledge nutzt dieselbe Pipeline wie
search_travel_knowledge. Fertig, wenn dieselbe /search-Route mit
collection=jobs nie Reise-Inhalte mischt.

## Phase 3 — n8n-Automationen
(a) Jobbörse/RSS beobachten -> match_job_posting -> Telegram/Slack bei Treffer.
(b) data/knowledge/-Commits beobachten -> rag-ingest automatisch anstoßen.
(c) Langfuse-API abfragen, bei Kosten-/Rate-Limit-Nähe warnen.
Fertig, wenn alle drei einmal end-to-end gelaufen sind, als JSON exportiert.

## Phase 4 — Geteiltes Tracing & CI/CD
Bewerbungshelfer bekommt @langfuse/tracing nach agent.service.ts-Vorbild, ins
selbe Langfuse-Projekt. ci.yml/deploy.yml aus ai-trip-planner kopiert.

## Phase 5 — Encounter-Agent-Pattern übertragen
LangGraph-Retry (generieren -> validieren -> bei Verstoß erneut) für
Anschreiben-Ton-/Längen-Check nachgebaut.

## Phase 6 (Stretch) — Azure-Deployment
Bewerbungshelfer als zweite Container App, container-app.bicep als Vorlage.

## Phase 7 (Stretch) — Portfolio-Anbindung
portfolio-page bekommt scripts/generate-platform-status.ts (Zwilling von
generate-deploy-info.ts) -> src/data/platform-status.json, gerendert von
PlatformStatus.tsx. Architektur-Diagramm als echter @xyflow/react-Graph
(NodeGraph.tsx-Vorbild) statt Bild. Ein n8n-Workflow committet die JSON direkt
ins portfolio-page-Repo statt einen neuen öffentlichen Endpunkt zu bauen.

## Minimal beeindruckende Version, wenn Zeit knapp wird
Phase 0 + 1 + 3(a) reichen für eine überzeugende Demo. Phase 4, 5, 6 sind
Politur, zuerst weglassen. Vollständiges Dokument mit Diagrammen:
https://claude.ai/code/artifact/f48cdc77-459c-40cb-ae8b-b284069821ac