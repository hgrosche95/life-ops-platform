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
