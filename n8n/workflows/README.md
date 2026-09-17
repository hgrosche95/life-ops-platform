# n8n-Workflows

Als JSON exportierte n8n-Workflows (Phase 3, siehe `PLAN.md`). Jeder Workflow
wurde per n8n-CLI importiert und einmal end-to-end ausgeführt, bevor er hier
committet wurde - kein unbestätigter Code.

## jobboerse-match-telegram.json

Beobachtet einen RSS-Feed mit Stellenanzeigen, gleicht die ersten drei
Einträge über `job-application-skill`s `POST /job-postings/match` gegen einen
hinterlegten Lebenslauf ab und schickt bei ≥50% erfüllten Anforderungen eine
Telegram-Nachricht.

**Warum nur die ersten 3 Einträge (`Limit`-Node):** Der RSS-Feed liefert oft
20+ Einträge; jede Anfrage an `match_job_posting` ist ein LLM-Aufruf. Mit
Groqs kostenlosem Free-Tier (8.000 Tokens/Minute) reißt das bei zu vielen
Einträgen das Rate-Limit, selbst mit dem eingebauten 429-Retry
(`RetryingLlmProvider`) - live beim Testen aufgetreten.

### Import

```bash
docker cp jobboerse-match-telegram.json life-ops-n8n:/tmp/workflow.json
docker exec life-ops-n8n n8n import:workflow --input=/tmp/workflow.json
```

### Vor dem ersten echten Lauf einzustellen (im n8n-UI)

- **Node "Lebenslauf"**: `resumeText` und `applicantName` durch deine echten
  Daten ersetzen (aktuell `TODO`-Platzhalter).
- **Node "Telegram-Nachricht"**: `chatId` eintragen (`TODO_TELEGRAM_CHAT_ID`) -
  siehe unten, wie man sie bekommt. Die Telegram-Credential selbst muss unter
  diesem Namen existieren: **Bewerbungshelfer Telegram Bot**.
- **Nodes "Login"/"match_job_posting"**: Die URL `http://host.docker.internal:3100`
  geht davon aus, dass `job-application-skill/apps/api` nativ auf dem Host auf
  Port 3100 läuft (nicht containerisiert - dafür gibt es noch kein
  Dockerfile). Username/Passwort im "Login"-Node müssen zu `AUTH_USERNAME`/
  `AUTH_PASSWORD_HASH` in dessen `.env` passen.

### Telegram-Bot + Credential einrichten

1. In Telegram `@BotFather` → `/newbot` → Token notieren.
2. Dem Bot eine beliebige Nachricht schicken (Bots dürfen sonst nicht antworten).
3. Chat-ID ermitteln: `https://api.telegram.org/bot<TOKEN>/getUpdates` →
   `message.chat.id`.
4. Credential per CLI anlegen (Token landet nicht im Klartext im Workflow-JSON):
   ```bash
   echo '[{"id":"telegram-bewerbungshelfer","name":"Bewerbungshelfer Telegram Bot","type":"telegramApi","data":{"accessToken":"<TOKEN>"}}]' > cred.json
   docker cp cred.json life-ops-n8n:/tmp/cred.json
   docker exec life-ops-n8n n8n import:credentials --input=/tmp/cred.json
   ```

### Getestet

Einmal end-to-end mit echtem RSS-Feed, echtem LLM-Abgleich (Groq) und echtem
Telegram-Versand gelaufen - zwei von drei Testanzeigen überschritten die
Schwelle, beide Nachrichten kamen an. Danach Chat-ID und Lebenslauf-Felder
wieder auf Platzhalter zurückgesetzt, bevor committet wurde.

## knowledge-ingest-watch.json

Stößt stündlich (und manuell) `POST /ingest` im `ai-trip-planer`-RAG-Service
an (`services/rag`, Branch `feat/rag-collections`).

**Warum kein echter Commit-Check gegen die GitHub-API:** Ursprünglich gebaut
mit `GET /repos/.../commits?path=data/knowledge`, einem Code-Node, der den
neuesten Commit-SHA mit `$getWorkflowStaticData('global')` gegen den vorigen
Lauf verglich, und einem IF-Node davor. Live getestet (Workflow aktiviert,
6 automatische Läufe im 1-Minuten-Takt beobachtet, direkt in der n8n-SQLite-
DB nachgesehen): `staticData` blieb bei jedem Lauf `null` - diese n8n-Version
(2.38.7) hat ein neueres, anderes internes Versionierungssystem
(`workflow_published_version`-Tabelle), in dem die klassische
`$getWorkflowStaticData`-Persistenz zwischen Läufen nicht wie erwartet
funktioniert.

Der Ersatz braucht keinen Zustand: `ingest_knowledge_base()`
(`services/rag/src/rag_service/ingest.py`) vergleicht pro Datei ohnehin einen
sha256-Hash und überspringt Unverändertes - ein stündlicher unbedingter
`/ingest`-Aufruf ist dadurch bereits günstig und sicher (kein Embedding-Aufruf
bei keiner Änderung), ohne von einem n8n-internen Mechanismus abzuhängen.

### Import

```bash
docker cp knowledge-ingest-watch.json life-ops-n8n:/tmp/workflow.json
docker exec life-ops-n8n n8n import:workflow --input=/tmp/workflow.json
```

Geht davon aus, dass `services/rag` nativ auf dem Host auf Port 8001 läuft
(`uv run uvicorn rag_service.main:app --port 8001`) - `host.docker.internal`
im HTTP-Request-Node erreicht den Host aus dem n8n-Container heraus.

### Getestet

Per `n8n execute` gegen die echte lokale Wissensbasis (4 Reiseziel-Dokumente)
gelaufen - alle vier korrekt als unverändert übersprungen (`skipped`), keine
Embedding-Kosten. Die eigentliche "neu"/"aktualisiert"-Logik ist identischer
Code zur bereits in Phase 2 getesteten CLI (`rag-ingest`), hier zusätzlich
über den neuen `/ingest`-HTTP-Endpunkt bestätigt erreichbar.

## langfuse-usage-warning.json

Fragt täglich Langfuses Metrics-API ab und schickt eine Telegram-Warnung,
wenn die geschätzte monatliche Nutzung ≥80% des Free-Tier-Limits (50.000
Units) erreicht.

**Wie die Auslastung berechnet wird:** Langfuse hat (Stand 2026) keinen
dedizierten "aktuelle Nutzung vs. Limit"-Endpunkt - das ist sogar ein offener
Feature-Request in ihrem GitHub. Laut ihrer eigenen Definition ist 1 Unit =
1 Trace + 1 Observation + 1 Score pro Abrechnungszeitraum. Der Workflow bildet
das über die v2-Metrics-API selbst nach:

```
GET /api/public/v2/metrics?query={
  "view": "observations",
  "metrics": [
    {"measure": "traceId", "aggregation": "uniq"},
    {"measure": "count", "aggregation": "count"},
    {"measure": "countScores", "aggregation": "sum"}
  ],
  "fromTimestamp": "<Monatserster 00:00 UTC>",
  "toTimestamp": "<jetzt>"
}
```

Gültige `view`/`measure`/`aggregation`-Werte sind in Langfuses Doku kaum
auffindbar - ermittelt, indem bewusst ungültige Werte geschickt wurden: die
Fehlermeldung zählt jeweils alle gültigen Optionen auf.

**Das ist eine Annäherung, keine garantiert exakte Langfuse-Kennzahl** - so
auch in der Telegram-Nachricht selbst formuliert, damit die Warnung nicht als
offizielle Zahl missverstanden wird.

### Import

```bash
docker cp langfuse-usage-warning.json life-ops-n8n:/tmp/workflow.json
docker exec life-ops-n8n n8n import:workflow --input=/tmp/workflow.json
```

### Vor dem ersten echten Lauf einzustellen

- **Node "Telegram-Warnung"**: `chatId` eintragen (`TODO_TELEGRAM_CHAT_ID`),
  Credential **Bewerbungshelfer Telegram Bot** wiederverwendet (siehe oben).
- **Credential "Langfuse Basic Auth"** (`langfuse-basic-auth`) per CLI anlegen:
  ```bash
  echo '[{"id":"langfuse-basic-auth","name":"Langfuse Basic Auth","type":"httpBasicAuth","data":{"user":"<PUBLIC_KEY>","password":"<SECRET_KEY>"}}]' > cred.json
  docker cp cred.json life-ops-n8n:/tmp/cred.json
  docker exec life-ops-n8n n8n import:credentials --input=/tmp/cred.json
  ```
- Bei US-Region-Projekt die URL im "Langfuse Nutzung abfragen"-Node von
  `cloud.langfuse.com` auf `us.cloud.langfuse.com` ändern.

### Getestet

Mit echten Langfuse-Keys end-to-end gelaufen: `units: 0` bei leerem Projekt
(korrekt, kein voreiliges `warn`), danach mit künstlich auf `true` gesetztem
`warn` erneut gelaufen, um den Telegram-Zweig zu beweisen - echte Nachricht
kam an. Danach beide Testwerte (`warn`, `chatId`) wieder zurückgesetzt.

## publish-platform-status.json

Sammelt täglich (und manuell) das letzte Ergebnis der drei obigen Workflows
über n8ns eigene Executions-API und committet eine Zusammenfassung als
`src/data/platform-status.json` in `portfolio-page` - Grundlage für ein
Status-Widget auf der `life-ops-platform`-Projektseite.

**Warum ein einzelner Aggregator statt dass jeder Workflow selbst committet:**
Ein Commit nach `portfolio-page`s `main` löst über `deploy.yml` ein volles
Rebuild/Redeploy der Seite aus. Drei unabhängige Schreiber hätten Merge-
Konflikt-Risiko und ein Redeploy bei jedem RSS-/Stunden-/Tages-Lauf bedeutet.
Ein Aggregator-Lauf pro Tag heißt: ein kontrollierter Redeploy-Rhythmus, ein
GitHub-Schreibrecht (fein-granularer PAT, nur `contents:write` auf
`portfolio-page`) statt drei.

**Wie das letzte Ergebnis pro Workflow ausgelesen wird:** `GET
/api/v1/executions?workflowId=...&limit=1&includeData=true` liefert die
komplette letzte Ausführung inklusive `data.resultData.runData` - ein Objekt
pro Node-Name mit dessen Output-Historie. Verifiziert gegen echte
Ausführungen (2026-09-16): `runData[nodeName]` ist ein Array von *Läufen*
dieses Nodes, nicht ein Array pro Item. Pro Ziel-Workflow wird der Output
eines bestimmten, bekannten Nodes ausgelesen (`Bewertung` bei
`jobboerse-match-telegram`, `rag-ingest anstoßen` bei
`knowledge-ingest-watch`, `Auslastung berechnen` bei
`langfuse-usage-warning`) und daraus ein Klartext-Ergebnis gebaut - keine
n8n-internen IDs oder Rohdaten im Widget.

**Erstanlage vs. Update:** `GET .../contents/src/data/platform-status.json`
liefert beim allerersten Lauf einen 404 (Datei existiert noch nicht) -
`neverError: true` verhindert, dass das den Workflow abbricht, und der
`PUT`-Body lässt `sha` dann einfach weg (legt die Datei neu an). Bei jedem
weiteren Lauf liefert derselbe GET ein `sha`, das der `PUT` mitschicken muss,
sonst lehnt GitHub das Update ab.

### Import

```bash
docker cp publish-platform-status.json life-ops-n8n:/tmp/workflow.json
docker exec life-ops-n8n n8n import:workflow --input=/tmp/workflow.json
```

### Vor dem ersten echten Lauf einzustellen

- **Credential "n8n API Key"** (`n8n-api-key`, `httpHeaderAuth`): ein n8n-API-
  Key aus dem n8n-UI (Settings → n8n API), als `Authorization: Bearer <KEY>`.
  ```bash
  echo '[{"id":"n8n-api-key","name":"n8n API Key","type":"httpHeaderAuth","data":{"name":"Authorization","value":"Bearer <KEY>"}}]' > cred.json
  docker cp cred.json life-ops-n8n:/tmp/cred.json
  docker exec life-ops-n8n n8n import:credentials --input=/tmp/cred.json
  ```
- **Credential "GitHub PAT (portfolio-page)"** (`github-portfolio-pat`,
  `httpHeaderAuth`): ein fein-granularer GitHub-PAT, nur für `portfolio-page`
  freigegeben, nur `Contents: Read and write`.
  ```bash
  echo '[{"id":"github-portfolio-pat","name":"GitHub PAT (portfolio-page)","type":"httpHeaderAuth","data":{"name":"Authorization","value":"Bearer <PAT>"}}]' > cred.json
  docker cp cred.json life-ops-n8n:/tmp/cred.json
  docker exec life-ops-n8n n8n import:credentials --input=/tmp/cred.json
  ```
- Geht davon aus, dass der n8n-Server selbst unter
  `http://host.docker.internal:5678` erreichbar ist - anders als bei den
  anderen drei Workflows braucht dieser hier also den laufenden Server
  während des Tests (`docker compose run --rm n8n execute` reicht nicht,
  wenn der Server währenddessen gestoppt ist).

### Getestet

Per `n8n execute` bei laufendem Server end-to-end gelaufen: alle drei
Ziel-Workflows lieferten ihr echtes letztes Ergebnis (u.a. "3 Anzeigen
geprüft, bester Treffer 5/5 bei Huzzle"), `SHA abrufen` traf den erwarteten
404 (Datei existierte noch nicht), `Committen` erzeugte einen echten Commit
in `portfolio-page` mit korrekter Parent-SHA. Die Erstanlage von
`src/data/platform-status.json` ist damit dieser Testlauf selbst.
