# Research Agent — Architecture Decision Record v2
**Date:** 2026-04-17
**Status:** Active

---

## 1. Overview

A personal research ingestion and knowledge management system. Articles, PDFs, YouTube videos, and pasted text are saved to an Airtable database, enriched with AI-generated metadata, cross-linked to related prior articles, and exported into a structured local wiki viewed in Obsidian.

---

## 2. System Architecture

```
Browser (submit UI)
    │
    ├─ POST article payload
    │       │
    │       ▼
    │   n8n Workflow (Railway)
    │   re-search.up.railway.app/webhook/research-agent
    │       │
    │       ├─ Phase 1: Scrape / extract content
    │       │   • URL   → Jina Reader (r.jina.ai/{url})
    │       │   • File/Paste → content passed directly
    │       │   • YouTube → Supadata API (lang=en)
    │       │
    │       ├─ Groq Call 1 (llama-3.1-8b-instant)
    │       │   → title, summary, tags, key_claims, source_type
    │       │
    │       ├─ Airtable PATCH — save enriched record
    │       │
    │       └─ Phase 2: Related Items (non-blocking)
    │           • Fetch prior Airtable records (exclude new record)
    │           • Groq Call 2 → top 3 related articles + reason
    │           • Airtable PATCH — update related_items field
    │
    ├─ GET /health → localhost:3456 (wiki-server.js)
    └─ POST /update-wiki → spawns export-to-wiki.js
            │
            ▼
        wiki/raw/   ← one .md per Airtable record
            │
            ▼
        Claude Code (local)
            │
            ▼
        wiki/  (Obsidian vault)
          ├─ concepts/
          ├─ topics/
          ├─ entities/
          └─ sources/
```

---

## 3. Components

### 3.1 Submit UI — `index.html` (GitHub Pages)

**Hosted at:** `https://kevinwmeier.github.io/re-search/`
**Repo:** `github.com/kevinwmeier/re-search`

Three input tabs:
- **URL** — paste any article/YouTube URL
- **Upload File** — drag-and-drop PDF, .txt, .md, .html (PDF.js for client-side text extraction)
- **Paste Text** — paste raw content with optional title

Additional section:
- **Update Knowledge Base button** — checks if `wiki-server.js` is running on `localhost:3456`, triggers `export-to-wiki.js`, displays export log, prompts user to run Claude Code for wiki rebuild.

### 3.2 n8n Workflow — `research-agent-phase1.workflow.json`

**Hosted on:** Railway (`re-search.up.railway.app`)

**Phase 1 node chain:**
1. `Webhook` — receives POST payload
2. `IF YouTube URL` — routes YouTube vs. other
3. `Supadata` — fetches transcript (`lang=en` required)
4. `Jina Reader` — scrapes web articles
5. `Build Groq Request` — constructs prompt with content (truncated to 6,000 chars)
6. `Groq Call 1` — `llama-3.1-8b-instant`, extracts: title, summary, tags, key_claims, source_type
7. `Parse Groq Response` — extracts JSON from model output
8. `Create Airtable Record` — saves enriched record
9. `IF Airtable OK` — gates Phase 2

**Phase 2 node chain (non-blocking):**
10. `Fetch Prior Records` — fetches all records excluding new one
11. `Build Groq 2 Request` — up to 50 prior articles as context
12. `IF Skip Groq 2` — skips if no prior records exist
13. `Groq Call 2` — finds top 3 meaningfully related articles
14. `Parse Groq 2 Response` — extracts related array
15. `IF Has Related` — skips PATCH if no matches
16. `Update Related Items` — PATCHes `related_items` field on the new record
17. `Respond Success` — returns title, tags, summary, record_id to UI

### 3.3 Airtable Database

**Base ID:** `appgiz3nNzSPrOfpR`
**Table:** `Research Items`

| Field | Type | Description |
|---|---|---|
| `title` | Text | AI-extracted title |
| `url` | URL | Source URL |
| `date_added` | Date | Auto-timestamp |
| `source_type` | Single select | Article / PDF / Video / Podcast / Report / Other |
| `tags` | Multiple select | AI-generated topic tags |
| `summary` | Long text | 2–3 sentence AI summary |
| `key_claims` | Long text | Bullet-point key claims |
| `my_notes` | Long text | User's own notes (from submit form) |
| `related_items` | Long text | Phase 2 auto-linked related articles |

**Views configured (manually in Airtable UI):**
- All Items (Grid, sorted by date_added desc)
- By Source Type (Grid, grouped by source_type)
- By Tag (Gallery)
- Recent (Grid, filtered to last 30 days)

### 3.4 Local Scripts

#### `export-to-wiki.js`
Fetches all Airtable records and writes one structured `.md` file per record to `wiki/raw/`. Handles pagination. Clears stale files before each run. Creates vault folder structure if missing.

#### `backfill-related-items.js`
Runs the Phase 2 Groq prompt against all existing Airtable records. Fills records missing `related_items`. Supports `--all` flag to reprocess everything. Rate-limit aware — parses "try again in Xs" from Groq errors and retries automatically.

#### `wiki-server.js`
Tiny HTTP server on `localhost:3456`. Provides:
- `GET /health` — returns `{ok: true}` so the browser button can detect if it's running
- `POST /update-wiki` — spawns `export-to-wiki.js`, streams stdout/stderr, returns `{ok, lines, error}`

Enables the "Update Knowledge Base" button in the browser UI to trigger local filesystem operations without leaving the browser.

### 3.5 Wiki (Obsidian + Claude Code)

**Vault location:** `wiki/` (relative to project root)

**Folder structure:**
```
wiki/
  raw/        ← exported from Airtable (read-only)
  concepts/   ← one page per concept/technology/framework
  topics/     ← broad cluster pages synthesising multiple concepts
  entities/   ← orgs, people, tools (created when appearing in 2+ sources)
  sources/    ← one page per source article
  _index.md   ← master index, always updated
  CLAUDE.md   ← schema and instructions for Claude Code
```

**Workflow:**
1. Run `node export-to-wiki.js` (or click button in UI)
2. Open Claude Code, `cd wiki/`
3. Tell Claude Code: *"I just added new sources to the raw/ folder. Please read them and build the wiki."*
4. Claude Code reads new raw files, creates/updates pages, maintains `_index.md`
5. Open Obsidian → Graph View to explore connections

---

## 4. Key Decisions

### ADR-001: Groq + llama-3.1-8b-instant (not 70b)
**Decision:** Use `llama-3.1-8b-instant` for both Groq calls.
**Rationale:** The 70b model consumed the free-tier daily token limit (100K) in a handful of submissions. The 8b model provides 500K tokens/day — sufficient for high-volume use. Quality is acceptable for structured metadata extraction.
**Trade-off:** Slightly lower quality on complex summarisation, but structured prompts with JSON output requirements mitigate this.

### ADR-002: Phase 2 is non-blocking
**Decision:** Related items linking runs after the HTTP response is sent to the user.
**Rationale:** Groq Call 2 adds ~2–5 seconds latency. The article is already saved; the user does not need to wait for cross-linking. If Phase 2 fails, the record still exists.
**Trade-off:** User sees no immediate indication of related items — they appear in Airtable and wiki after next export.

### ADR-003: Jina Reader for web scraping
**Decision:** Use `r.jina.ai/{url}` for article extraction.
**Rationale:** No infrastructure required. Returns clean markdown-formatted text ready for LLM consumption. Handles most paywalled and JavaScript-rendered pages.
**Trade-off:** Dependent on Jina's availability. Fails on some sites.

### ADR-004: Supadata for YouTube transcripts
**Decision:** Use Supadata API with `lang=en` parameter.
**Rationale:** YouTube Data API requires OAuth and quota management. Supadata is simpler to call from n8n. The `lang=en` parameter is required — without it, Supadata returns the first available language (which may be non-English for channels with auto-translated captions).

### ADR-005: Karpathy-style wiki over RAG
**Decision:** Build a persistent interlinked markdown wiki (read by Claude Code) instead of a vector database RAG system.
**Rationale:** RAG retrieves chunks at query time but doesn't synthesise knowledge across sources. The wiki approach (inspired by Andrej Karpathy's LLM wiki concept) has Claude Code read every article once and build persistent, human-readable pages that improve incrementally. Works with Obsidian's graph view for visual exploration.
**Trade-off:** Requires Claude Code runs to update the wiki; not fully automated. Wiki pages can drift if not rebuilt after new imports.

### ADR-006: localhost bridge server for browser→filesystem
**Decision:** `wiki-server.js` runs on `localhost:3456` to let the GitHub Pages UI trigger local scripts.
**Rationale:** GitHub Pages is static — it cannot execute server-side code. The submit UI needs to trigger `export-to-wiki.js` which writes to the local filesystem. A tiny local HTTP server bridges this gap. Browsers allow `fetch()` to `http://localhost` from HTTPS pages (localhost is a trusted origin per W3C spec).
**Trade-off:** User must remember to start `wiki-server.js` before using the button. The UI detects and shows instructions if the server is not running.

### ADR-007: Content truncation at 6,000 characters
**Decision:** Truncate article content to 6,000 chars before sending to Groq.
**Rationale:** Metadata extraction (title, summary, tags, key_claims) does not require the full article. Truncating reduces token consumption significantly and stays well within rate limits.
**Trade-off:** Very long articles may have their key claims drawn only from the first portion of the content.

---

## 5. External Services & Credentials

| Service | Purpose | Notes |
|---|---|---|
| Airtable | Database | Base ID: `appgiz3nNzSPrOfpR` |
| Groq | LLM inference | Model: `llama-3.1-8b-instant` |
| Railway | n8n hosting | `re-search.up.railway.app` |
| GitHub Pages | Submit UI hosting | `kevinwmeier.github.io/re-search` |
| Jina Reader | Web scraping | `r.jina.ai/{url}` — no key required |
| Supadata | YouTube transcripts | `lang=en` param required |
| PDF.js | Client-side PDF extraction | CDN via `cdnjs.cloudflare.com` |

*Credentials are stored in `Research_Assistant.txt` (not committed) and hardcoded in scripts for local use.*

---

## 6. Repository Structure

```
kevinwmeier/re-search          ← GitHub Pages (submit UI)
  index.html                   ← Submit UI + Update Knowledge Base button

kevinwmeier/ai-projects        ← Script/toolchain repo
  Research_Agent/
    submit.html                ← Source for index.html above
    export-to-wiki.js          ← Airtable → wiki/raw/ export
    backfill-related-items.js  ← Phase 2 backfill for existing records
    wiki-server.js             ← localhost:3456 bridge server
    wiki-CLAUDE.md             ← Claude Code schema (copied into wiki/)
    research-agent-phase1.workflow.json  ← n8n workflow (import to Railway)
    wiki/                      ← Obsidian vault (not committed)
```

---

## 7. Runbook

### Save a new article
1. Open `https://kevinwmeier.github.io/re-search/`
2. Paste URL, upload file, or paste text → Submit
3. Record appears in Airtable within ~10 seconds
4. Related items field populates in background (~30 seconds)

### Update the wiki
1. In a terminal: `cd "C:\Users\kevin\OneDrive\Documents\AI_Projects\Research_Agent" && node wiki-server.js`
2. In the browser UI, click **⟳ Update Knowledge Base**
3. Switch to Claude Code, `cd wiki/`
4. Tell Claude Code: *"I just added new sources to the raw/ folder. Please read them and build the wiki."*
5. Open Obsidian → Graph View

### Backfill related items for existing records
```
cd C:\Users\kevin\OneDrive\Documents\AI_Projects\Research_Agent
node backfill-related-items.js          # fills only empty records
node backfill-related-items.js --all    # reprocesses all records
```
Note: Rate limited to ~8 records/minute on Groq free tier. Script handles rate limit retries automatically.

### Deploy workflow changes to n8n
1. Edit `research-agent-phase1.workflow.json`
2. Open Railway n8n instance
3. Import workflow JSON (replace existing)

---

## 8. Known Limitations

- Groq free tier: 500K tokens/day on 8b model. Heavy use days may hit limits.
- `wiki-server.js` must be manually started before using the browser button.
- Wiki pages must be manually rebuilt via Claude Code after new imports — not fully automated.
- Scanned PDFs (image-only) are not supported — PDF.js can only extract text-layer PDFs.
- YouTube videos without English captions/transcripts will fail to extract text.
- Related items are limited to the 50 most recent prior records for Groq context window reasons.
