# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

**hapaxromana** is the architectural specification and coordination space for a three-tier autonomous agent system. It currently contains design documents, not runnable code. Agent implementations live in separate repos.

## Architecture

```
TIER 1: Interactive    → Claude Code (command center, full MCP access)
TIER 2: On-Demand      → Pydantic AI agents invoked via CLI/Claude Code
TIER 3: Autonomous     → systemd services + n8n scheduled workflows
```

All tiers share: **LiteLLM** (:4000) for model routing, **Qdrant** (:6333) for vector memory, **Langfuse** (:3000) for observability. Tier 2/3 agents route all LLM calls through LiteLLM. Exception: Claude Code (Tier 1) makes direct API calls to Anthropic, not through LiteLLM.

Full design: `agent-architecture.md`

## Related Repos

| Repo | Purpose |
|------|---------|
| `~/projects/ai-agents/` | Pydantic AI agent implementations (Tier 2) + cockpit API backend |
| `~/projects/cockpit-web/` | React SPA web dashboard (cockpit frontend) |
| `~/projects/hapax-vscode/` | VS Code extension — chat sidebar, RAG search, management commands |
| `~/projects/obsidian-hapax/` | **Archived** — replaced by hapax-vscode |
| `~/llm-stack/` | Docker Compose for all infrastructure services |

## Tier 2 Agents (Pydantic AI)

Invocation pattern from `~/projects/ai-agents/`:
```bash
uv run python -m agents.<name> --flag value
```

### Implemented

| Agent | LLM? | Purpose |
|-------|------|---------|
| `research` | Yes | RAG-backed research with Qdrant tools |
| `code_review` | Yes | Code review with operator context |
| `profiler` | Yes | Operator profile extraction/curation from all sources (13 dimensions) |
| `health_monitor` | No | Deterministic health checks (17 groups, 75 checks), auto-fix, `--history` |
| `introspect` | No | Infrastructure manifest generator |
| `drift_detector` | Yes | Docs vs reality comparison, `--fix` mode |
| `activity_analyzer` | No* | Langfuse/health/drift telemetry aggregation (*optional `--synthesize`) |
| `briefing` | Yes | Daily operational briefing — telemetry + calendar schedule + Drive activity + email stats |
| `scout` | Yes | Horizon scanner — evaluates stack components against external landscape |
| `management_prep` | Yes | 1:1 prep (calendar + Gmail thread context), team snapshots, management overview |
| `meeting_lifecycle` | Yes | Meeting prep (48h calendar lookahead trigger), post-meeting processing, transcript ingestion, weekly review |
| `digest` | Yes | Content/knowledge digest — service-aware document grouping from RAG sources |
| `knowledge_maint` | No* | Qdrant vector DB hygiene — dedup, stale pruning, stats (*optional `--summarize`) |
| `demo` | Yes | Audience-tailored demo generator — slides, screenshots, voice-cloned video |
| `demo_eval` | Yes | Demo output evaluator — LLM-as-judge with self-healing loop |
| `gdrive_sync` | No | Google Drive incremental sync to RAG pipeline |
| `gcalendar_sync` | No | Google Calendar sync to RAG pipeline |
| `gmail_sync` | No | Gmail metadata sync to RAG pipeline |
| `youtube_sync` | No | YouTube subscriptions/likes sync to RAG pipeline |
| `claude_code_sync` | No | Claude Code transcript sync to RAG pipeline |
| `obsidian_sync` | No | Obsidian vault sync to RAG pipeline |
| `chrome_sync` | No | Chrome history + bookmarks sync to RAG pipeline |
| `audio_processor` | No | Ambient audio processing — VAD, classification, diarization, transcription to RAG |

### Planned

sample-curator, draft, midi-programmer.

### Shared Utilities

| Module | Purpose |
|--------|---------|
| `shared/takeout/` | Google Takeout ZIP processor — 14 services, 13 parsers, dual-path output |
| `shared/llm_export_converter.py` | Claude.ai / Gemini data export → markdown converter |
| `shared/management_bridge.py` | Deterministic management fact extraction from vault (zero LLM) |
| `shared/transcript_parser.py` | VTT/SRT/speaker-labeled transcript parsing (stdlib-only) |
| `shared/config.py` | Model aliases, embedding, Qdrant client, batch embedding |
| `shared/google_auth.py` | Google OAuth2 credential management (Drive, Calendar, Gmail, YouTube) |
| `shared/calendar_context.py` | Calendar event formatting for agent context injection |
| `shared/notify.py` | Unified notifications (ntfy + desktop) |
| `shared/vault_writer.py` | Vault egress (writes to `30-system/`, `10-work/`, `32-bridge/`) |

#### Takeout Processor (`shared/takeout/`)

Processes Google Takeout ZIP exports into the profiler + RAG pipeline. Dual data paths:
- **Unstructured** (emails, notes, documents) → markdown with YAML frontmatter → `~/documents/rag-sources/takeout/{service}/` → RAG watchdog + profiler LLM extraction
- **Structured** (search queries, purchases, calendar, location) → JSONL → deterministic ProfileFact mapping (zero LLM cost, `profiler_bridge.py`)

```bash
uv run python -m shared.takeout --list-services ~/Downloads/takeout.zip
uv run python -m shared.takeout ~/Downloads/takeout.zip --services chrome,keep --since 2025-01-01
uv run python -m shared.takeout ~/Downloads/takeout.zip --resume    # Resume interrupted run
uv run python -m shared.takeout --progress ~/Downloads/takeout.zip  # Show progress
```

14 services across 3 tiers (chrome, search, keep, youtube, calendar, contacts | gmail, drive, chat | maps, photos, purchases, tasks, gemini). Progress tracking enables resume for large exports. Profiler `--auto` flow loads structured facts automatically.

## Tier 3 Services

### Systemd Timers (host)

| Timer/Service | Schedule | Purpose |
|---------------|----------|---------|
| `rag-ingest.service` | Always running | Watches ~/documents/rag-sources/ |
| `health-monitor.timer` | Every 15 min | Auto-fix + desktop notification on failures |
| `profile-update.timer` | Every 6h | Incremental operator profile update |
| `meeting-prep.timer` | Daily 06:30 | Auto-generate 1:1 prep docs |
| `digest.timer` | Daily 06:45 | Content digest — 15 min before briefing |
| `daily-briefing.timer` | Daily 07:00 | Morning briefing + notification |
| `scout.timer` | Weekly Wed 10:00 | Horizon scan — external fitness evaluation |
| `drift-detector.timer` | Weekly Sun 03:00 | Documentation drift detection |
| `manifest-snapshot.timer` | Weekly Sun 02:30 | Infrastructure state snapshot |
| `knowledge-maint.timer` | Weekly Sun 04:30 | Qdrant dedup, stale pruning, stats |
| `llm-backup.timer` | Weekly Sun 02:00 | Full stack backup |
| `vram-watchdog.timer` | Every 30 min | GPU VRAM pressure monitoring |
| `obsidian-webui-sync.timer` | Every 6h | Obsidian vault → Open WebUI sync |
| `audio-recorder.service` | Always on | Continuous mic recording (Blue Yeti, ffmpeg) |
| `audio-processor.timer` | Every 30min | Audio segmentation + transcription + RAG (GPU) |
| `audio-archiver.timer` | Daily 03:00 | Archive raw audio to Google Drive (rclone) |
| VS Code + GitDoc | Always running | Vault sync via git auto-commit + push |

### Sync Pipeline Container (`hapax-sync-pipeline`)

7 RAG sync agents run in Docker via supercronic. Managed by `docker compose` in `~/projects/ai-agents/`.

| Agent | Prod Schedule | Purpose |
|-------|---------------|---------|
| `gdrive_sync` | Every 2h | Google Drive RAG sync |
| `gcalendar_sync` | Every 30min | Google Calendar RAG sync |
| `gmail_sync` | Every 1h | Gmail metadata RAG sync |
| `youtube_sync` | Every 6h | YouTube subscriptions/likes sync |
| `claude_code_sync` | Every 2h | Claude Code transcript RAG sync |
| `obsidian_sync` | Every 30min | Obsidian vault RAG sync |
| `chrome_sync` | Every 1h | Chrome history + bookmarks sync |

`CYCLE_MODE` env var selects `crontab.prod` or `crontab.dev` (reduced frequencies).

## Multi-Channel Access

| Channel | Transport | Ingress | Egress |
|---------|-----------|---------|--------|
| Desk | localhost | Web dashboard, chat, CLI | Web dashboard, notify-send |
| Remote | Tailscale | Open WebUI (https://pop-os-2.tailf9491.ts.net) | Chat responses |
| Mobile | Tailscale + Telegram | Telegram bot messages | ntfy push, Telegram replies |
| Knowledge | Git (vault repo) | Vault notes → RAG | System → vault markdown |
| Cloud | rclone (Google Drive) | Drive files → RAG | (read-only) |

**Capability matrix — what each channel can do:**

| Capability | Web (cockpit-web) | VS Code (hapax-vscode) | Mobile (Telegram) |
|-----------|-------------------|------------------------|-------------------|
| Health monitoring | Full (live refresh) | Status bar (cockpit API) | Alert push only |
| Agent execution | Full (all agents) | — | — |
| Chat / conversation | Full (streaming) | Full (LiteLLM/direct) | Quick capture only |
| Nudge management | Full (act/dismiss) | View nudges command | Daily digest |
| Profile visibility | Full (/profile) | View profile command | — |
| Knowledge search | Via research agent | RAG search (Qdrant) | — |
| Briefing access | Dashboard view | Git sync (30-system/) | Push notification |
| Scout reports | Full | — | — |

**Design rationale:** The web dashboard is the full command center — no capability is exclusive to another channel. VS Code is the knowledge interface (read + search + chat + management commands). Mobile is notification-driven with minimal interaction (quick capture via Telegram bot, briefing/health alerts via ntfy).

Key components: Tailscale (mesh VPN), ntfy (push notifications, Docker), n8n workflows (Telegram bot), `shared/notify.py` (unified notification dispatch), `shared/vault_writer.py` (vault egress).

## Vaults

Two-vault architecture bridges the Zscaler corporate boundary.

**Work vault:** `~/Documents/Work/` — git repo, syncs to corporate work laptop via git remote
**Personal vault:** `~/Documents/Personal/` — local only (no sync needed)

**Work vault structure:**
| Folder | Purpose |
|--------|---------|
| `00-inbox/` | Work captures (all devices) |
| `10-work/` | Management domain — people, meetings, projects, decisions, references, coaching, feedback, 1on1-prep |
| `30-system/` | System-managed (vault_writer output: briefings, digests, nudges, goals, management-overview, hapax-context) |
| `32-bridge/` | Bridge zone — prompts, guides |
| `40-calendar/` | Daily and weekly notes |
| `50-templates/` | Work templates |
| `90-attachments/` | Media and file attachments |

**Personal vault structure:**
| Folder | Purpose |
|--------|---------|
| `00-inbox/` | Personal captures |
| `20-personal/` | Personal domain — music, etc. |
| `50-templates/` | Personal templates |
| `90-attachments/` | Media and file attachments |

**Data flows:**
- System → Vault: `vault_writer.py` writes to Work vault (`30-system/`, `10-work/`) → GitDoc auto-commits → git push → all devices
- Vault → System: RAG ingestion from `~/documents/rag-sources/` → Qdrant
- Briefing agent writes to Work vault on `--save`

**Qdrant `documents` collection metadata:** Payloads include `source_service` (gdrive, gcalendar, gmail, youtube, takeout, proton, claude-code, obsidian, chrome) and `source_platform` tags, auto-detected by `ingest.py` from `rag-sources/` path patterns.

**VS Code extension:** `hapax-vscode` (`~/projects/hapax-vscode/`) — chat sidebar with streaming, Qdrant RAG search, 1:1 prep generation, team snapshots, decision capture, provider abstraction (LiteLLM/OpenAI/Anthropic). API keys resolved from VS Code settings → env vars → pass store. RAG features home-only; chat works anywhere via direct provider APIs.

## Model Aliases (via LiteLLM)

| Alias | Model | Use Case |
|-------|-------|----------|
| `fast` | claude-haiku | Cheap, quick tasks |
| `balanced` | claude-sonnet | Default for most agents |
| `reasoning` | deepseek-r1:14b | Complex reasoning (Ollama) |
| `coding` | qwen-coder-32b | Code generation (Ollama) |
| `local-fast` | qwen-7b | Lightweight local tasks |

Embedding: `nomic-embed-text-v2-moe` via Ollama (768d, requires `search_query:`/`search_document:` prefixes).

VRAM constraint: RTX 3090 = 24GB. One large Ollama model at a time.

## Document Authority Hierarchy

When documents disagree, the following precedence applies:

1. **Source code** — the implementation is always the ground truth
2. **`CLAUDE.md`** (this file) — canonical reference for the current system state
3. **`operations-manual.md`** — operational how-to, maintained alongside code
4. **`agent-architecture.md`** — architectural design reference, updated less frequently
5. **`README.md`** — public-facing summary, may lag behind

When updating documentation, propagate changes to all affected documents. The drift-detector agent checks for inconsistencies weekly.

## Conventions

- Python: `uv`, type hints mandatory, Pydantic models
- All LLM calls through LiteLLM, all traces to Langfuse
- Secrets via `pass` + `direnv`, never hardcoded
- Git: conventional commits, feature branches from `main`
- Agents are stateless per-invocation; persistent state in Qdrant or filesystem
