# Experiment Response Monitor

**Need (one sentence):** Research assistants running live Qualtrics experiment sessions currently find out about incomplete, inattentive, or timed-out participant responses only when they clean data after the fact — this workflow catches and alerts on those responses the moment they come in.

## Architecture

Qualtrics (polled on a schedule) → export job started → wait for export to complete → download + unzip + parse responses → split into individual items → duplicate check against the log → validation (incomplete / failed attention check / duration outlier / condition mismatch / clean) → log every response → Slack alert on anything flagged.

See `docs/architecture-diagram.png` (or the project proposal PDF) for the full labeled diagram.

A parallel error workflow (Error Trigger → Slack) is attached via workflow Settings → Error Workflow, so any unhandled failure in the main workflow also alerts separately with the workflow name and a link to the failed execution.

## Running it (self-hosted via Docker)

1. Clone this repo.
2. Copy `.env.example` to `.env` and fill in real values:
   - `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` — any values, this is a fresh local database.
   - `N8N_ENCRYPTION_KEY` — generate one with `openssl rand -base64 32`.
3. Start everything:
   ```bash
   docker compose up -d
   ```
4. n8n will be available at `http://localhost:5678`. On first run, create an owner account.
5. In the n8n editor, import the workflow files from `workflows/` (Menu → Import from File), one for each workflow (main + error handler).
6. Re-add credentials inside this instance: Google Sheets OAuth2, Slack OAuth2, and the Qualtrics API Header Auth credential (`X-API-TOKEN`) — credentials are never stored in the exported JSON or in this repo, so they must be re-entered per instance.
7. Data persists across restarts via the named Docker volumes (`postgres_data`, `n8n_data`) — `docker compose down` followed by `docker compose up -d` will not lose your workflows or n8n's internal data.

**Note on the live demo:** the live demo for this project runs on n8n Cloud (for credential/OAuth stability during presentation), using the exact same workflow JSON committed in `workflows/` in this repo. This docker-compose setup is provided so the system can be run fully self-hosted, and was tested to start cleanly from a fresh clone.

## How to test

- Each workflow keeps its original manual/mock trigger (`Edit Fields` test node) alongside the real Qualtrics-polling trigger, so you can fire a test run on demand without waiting on real survey data — see the "Trigger fireable on demand" note in the workflow itself.
- A clean mock response should log to the sheet/database with no Slack alert.
- A response with `attention_check` set to anything other than `"correct"`, or `duration_sec` outside 60–1800, should log *and* trigger a Slack alert with a plain-English reason.
- Submitting the same `response_id` twice should produce exactly one logged row and one alert (idempotency check via the duplicate-check `If` node).
- Executions tab shows recent runs; a deliberately broken credential/URL triggers the attached error workflow, visible as a separate Slack alert with the workflow name and execution link.

## Screenshots

_Add screenshots here before submitting: canvas overview, a flagged Slack alert, a clean sheet/DB row, and the Executions tab showing a green run._

## Workflow files

- `workflows/experiment-response-monitor-vrac.json` — main workflow, pointed at the real VRAC experiment survey (used to demo realistic batch volume)
- `workflows/experiment-response-monitor-demo.json` — same workflow, duplicated and pointed at a short dedicated demo survey (used for the live round-trip demo)
- `workflows/experiment-monitor-error-handler.json` — error workflow, attached to both of the above via Settings → Error Workflow

See `Experiment Response Monitor - Project Proposal.pdf` in this repo for the full written proposal and architecture diagram.
