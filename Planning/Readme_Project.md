Below is a **Python-first “opencode vibe code” plan** that replaces the fragile n8n expression/merge plumbing with a small, testable service—while keeping the *same core behavior* from your earlier n8n design (Slack capture → AI classify → Obsidian write → Inbox log → Slack confirmation). This plan also bakes in the **emoji-path contract** (never guess vault paths; always resolve via registry) that came out of your vault analysis.  

---

## Executive decision (CTO view)

**Pivoting to Python is the right move** *if* your goal is reliability + evolvability.

n8n is great until you need: cross-node state, robust fallbacks, idempotent upserts, deterministic formatting, and “no nulls ever” guarantees. Your current failures are classic “workflow glue” issues: missing merges, `null` registry context, path resolution edge cases, and expression syntax escaping. Your own docs already call out the need for tighter contracts (path registry; safer defaults; avoid “guessing”).

Python lets you:

* lock behavior with **TDD** (unit + contract tests),
* make **idempotency** real (dedupe by Slack `ts`),
* enforce the **path registry contract** centrally,
* and grow into scheduling assistant features without rewiring a graph every time.

---

## Target MVP behavior (Definition of Done)

**MVP = “Capture pipeline works end-to-end without manual babysitting.”**

1. Slack message in `#sb-inbox` triggers service
2. Service classifies into `{people|projects|ideas|needs_review}` using your schema (confidence threshold) 
3. Service loads `path_registry_new.json` and resolves emoji folder paths (never guesses) 
4. Writes/updates in Obsidian via Local REST API:

   * **Projects** → append task block into the correct Project note (or create note from template)
   * **Reminders** → append into today’s Daily note (Tasks plugin format)
   * **People** → append/update person note
   * **Ideas** → create idea note
5. Appends an **Inbox Log** entry (audit trail) in `zzz_System/Logs/Inbox` 
6. Replies in Slack thread with what happened + link/path

Everything above is covered by tests.

---

## Phase 0 — Project scaling first (your request)

### 0.1 Install MCP Launchpad (dev acceleration)

MCP Launchpad install (per upstream): ([GitHub][1])

```bash
uv tool install https://github.com/kenneth-liao/mcp-launchpad.git
mcpl list
```

It reads MCP config from `./mcp.json` (project) or `~/.claude/mcp.json` (global). ([GitHub][1])

### 0.2 Create `mcp.json` (project root)

This is a **good starting config** for your stack. Fill in env vars in `.env` (no secrets in git).

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    },
    "git": {
      "command": "npx",
      "args": ["-y", "@cyanheads/git-mcp-server"],
      "env": {
        "GITMCP_REPO_PATH": "${GITMCP_REPO_PATH}"
      }
    },
    "linear": {
      "command": "npx",
      "args": ["-y", "@linear/mcp"],
      "env": {
        "LINEAR_API_KEY": "${LINEAR_API_KEY}"
      }
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_SIGNING_SECRET": "${SLACK_SIGNING_SECRET}"
      }
    },
    "google_calendar": {
      "command": "npx",
      "args": ["-y", "@gongrzhe/server-calendar-autoauth-mcp"],
      "env": {
        "GOOGLE_CALENDAR_ID": "${GOOGLE_CALENDAR_ID}"
      }
    },
    "obsidian": {
      "command": "uvx",
      "args": ["mcp-obsidian"],
      "env": {
        "OBSIDIAN_REST_API_BASE": "${OBSIDIAN_REST_API_BASE}",
        "OBSIDIAN_REST_API_TOKEN": "${OBSIDIAN_REST_API_TOKEN}"
      }
    }
  }
}
```

Notes:

* MCP Launchpad expects `${VAR}` env substitution and supports project `.env`. ([GitHub][1])
* `context7` is commonly used as an MCP docs helper and includes “opencode” config examples. ([GitHub][1])
* Slack MCP availability differs by server; official Slack guidance also notes limited access in some cases—so having a fallback MCP server option is smart. ([Slack Developer Docs][2])
* If you prefer a different Obsidian MCP, there are multiple community options; pick one that matches your REST API plugin. ([GitHub][3])

### 0.3 Repo scaffold (Python, TDD-first)

Use `uv` + `pyproject.toml` from day one.

**Recommended libraries**

* Core: `fastapi`, `uvicorn`, `pydantic`, `httpx`, `python-dotenv`, `tenacity`
* Slack: `slack-bolt`
* Observability: `structlog` (or `loguru`)
* Templates: `jinja2`
* Dates: `python-dateutil`
* File watching (post-MVP): `watchdog`
* Testing: `pytest`, `pytest-asyncio`, `pytest-cov`, `respx` (mock httpx), `freezegun`
* Quality gates: `ruff`, `mypy`, `pre-commit`

(Your earlier requirements already emphasize cost-effective LLM usage; keep that but implement it as a policy module in Python.)

### 0.4 Security hygiene (do this immediately)

Your uploaded planning artifacts include secrets/tokens. Rotate them and move all credentials to `.env` + secret store.

---

## Phase 1 — MVP build plan (TDD + Linear)

### Linear structure (what to create)

**Initiative:** Second Brain Python MVP
**Epics:**

1. Project Scaling & Tooling
2. Capture Pipeline MVP
3. Obsidian Writers & Templates
4. Audit Log + Idempotency
5. QA Automation (sub-agent owned)
6. Post-MVP: Vault Crawl + Registry Sync
7. Scheduling Assistant (Calendar layer)

Each ticket should include:

* Acceptance criteria
* Tests required
* Failure modes

### Service architecture (simple, not overbuilt)

**One Python service** (FastAPI) with modules:

* `ingest/slack_events.py`

  * verifies Slack signatures
  * extracts `text`, `user`, `channel`, `ts`, `thread_ts`
* `classify/llm.py`

  * builds prompt
  * parses strict JSON
  * enforces confidence threshold
* `registry/path_registry.py`

  * loads `path_registry_new.json`
  * resolves emoji paths deterministically
* `obsidian/client.py`

  * wraps Obsidian Local REST calls (`list`, `read`, `write`, `append`)
* `writers/{projects,people,ideas,daily,inbox_log}.py`

  * “render task block” (Tasks plugin-compatible)
  * “append safely” (with anchors)
* `dedupe/store.py`

  * sqlite (or tiny local file) keyed by Slack `ts` to prevent duplicates
* `reply/slack_reply.py`

  * thread reply with summary and resolved path

This mirrors your prior n8n workflow stages—just without the fragile per-node state juggling. 

### TDD approach (what to test first)

Start with tests that lock the contracts:

1. **Classification parsing**

   * strips code fences
   * rejects invalid JSON
   * normalizes destination casing (`projects` not `Projects`)
2. **Registry resolution**

   * given `destination=projects` and `project_area=house`, returns correct `Projects/🏠 House` style path from registry
   * never returns null; falls back to `PROJECT_SECOND_BRAIN_ROOT` (or needs_review)
     (This is the bug that’s biting your n8n flow today.) 
3. **Writers produce valid markdown blocks**

   * project task block includes starter step + DoD + next action
   * reminder tasks go to Daily note
4. **Inbox log entry is append-only and includes Slack ts**
5. **Idempotency**

   * same Slack `ts` processed twice → second run is a no-op (or updates status)

---

## Phase 2 — Post-MVP (your vault crawl / registry sync idea)

Create a Linear Epic: **“Vault crawl + registry sync”**.

Two implementation options:

1. **Simplest (recommended): scheduled crawl**

   * daily or hourly: call Obsidian API `GET /vault/` then walk directories and regenerate registry
   * avoids file watcher edge cases and VM filesystem quirks

2. **Watcher-based (your idea): watchdog → webhook to service**

   * `watchdog` monitors vault folder
   * on change, call `POST /admin/refresh-registry`
   * service re-crawls + updates `path_registry_new.json`

Put this behind a debounce (e.g., 2–5 seconds) so you don’t thrash during bulk edits.

Your earlier analysis already treats the registry as a critical contract—automating it is a clean post-MVP win. 

---

## Phase 3 — Scheduling assistant integration (after capture is stable)

Once capture is reliable, add the “calendar layer”:

* New classification outcomes:

  * `kind: reminder` → Daily note + optional calendar time-block suggestion
  * `kind: project_task` with due date → backplan + propose 2 schedule options
* Slack UX:

  * buttons: “Schedule this”, “Break into steps”, “Snooze”, “Done”
* Guardrails: don’t write calendar events unless explicitly approved (matches your assistant spec)

This is easiest when your task + calendar logic lives in the same codebase (again: reason to pivot).

---

## Agent plan (opencode “vibe code” execution)

You said you want a QA sub-agent and MCP launchpad hosting MCPs. Here’s the agent lineup I’d run:

1. **Tech Lead Agent (you + opencode)**

   * owns architecture decisions, defines contracts, keeps scope tight

2. **Integration Agent: Slack**

   * Slack events, signature verification, thread replies, button interactions

3. **Integration Agent: Obsidian**

   * REST client + robust append semantics + template rendering

4. **Registry Agent**

   * reads registry, resolves paths, post-MVP crawler

5. **LLM Agent**

   * prompt + JSON parsing + confidence policy + model routing (cheap-by-default per your requirements)

6. **QA Agent (hard gatekeeper)**

   * enforces TDD, adds regression tests for every bug, runs `pytest -q`, coverage thresholds, mocks HTTP via `respx`

7. **Linear Agent**

   * creates epics/issues, keeps acceptance criteria crisp, closes the loop between bugs ↔ tests

Because you’re using MCP launchpad, all agents can discover tools via `mcpl search` and execute them via `mcpl call`. ([GitHub][1])

