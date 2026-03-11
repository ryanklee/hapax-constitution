# Domain 5: Intelligence Agents — Audit Findings

**Auditor:** Claude Opus 4.6
**Date:** 2026-03-02
**Scope:** 6 source files (2,120 LOC), 3 test files (834 LOC), 1 related test file (387 LOC)

## Inventory

| File | LOC | Test File | Test LOC | Test Coverage |
|------|-----|-----------|----------|---------------|
| `agents/research.py` | 181 | — | — | **None** |
| `agents/code_review.py` | 120 | — | — | **None** |
| `agents/briefing.py` | 385 | `tests/test_briefing.py` | 208 | Partial (schemas, formatters, notification only) |
| `agents/scout.py` | 544 | — | — | **None** |
| `agents/digest.py` | 385 | `tests/test_digest.py` | 345 | Good (schemas, formatters, collectors, notification) |
| `agents/management_prep.py` | 505 | `tests/test_management_prep.py` | 281 | Good (data collection, formatters, mocked LLM) |
| — | — | `tests/test_management.py` | 387 | Good (management data collectors) |
| **Total** | **2,120** | | **1,221** | |

## Agent Architecture Summary

| Agent | Model | System Prompt | Context Tools | Output Type | Error Handling |
|-------|-------|---------------|---------------|-------------|----------------|
| research | `balanced` (claude-sonnet) | Dynamic (goals + instructions) | Yes | Unstructured string | None |
| code_review | `balanced` (claude-sonnet) | Static fragment + instructions | Yes | Unstructured string | None |
| briefing | `fast` (claude-haiku) | Static prompt | Yes | `Briefing` (Pydantic) | None around LLM call |
| scout | `balanced` (claude-sonnet) | Static prompt | Yes | `Recommendation` (Pydantic) | Per-component try/except |
| digest | `fast` (claude-haiku) | Static prompt | **No** | `Digest` (Pydantic) | None around LLM call |
| management_prep | `balanced` / `fast` (3 agents) | Static prompts (3) | Yes (all 3 agents) | 3 Pydantic models | None around LLM call |

---

## Completeness Findings

### C-5.1: Digest agent missing context tool registration [medium]

**File:** `agents/digest.py:179-183`

The digest agent does NOT register context tools. All other 5 agents in this domain do.

```python
# digest.py:179-183
digest_agent = Agent(
    get_model("fast"),
    system_prompt=SYSTEM_PROMPT,
    output_type=Digest,
)
# No context tool registration follows
```

Compare with briefing.py:109-112, scout.py:235-238, management_prep.py:136-140, research.py:73-75, code_review.py:58-60 — all register context tools via the standard pattern:

```python
from shared.context_tools import get_context_tools
for _tool_fn in get_context_tools():
    agent.tool(_tool_fn)
```

The digest agent is the only one that skips this. This means the digest agent cannot look up operator constraints, patterns, or profile data on demand. Since the digest is content-focused rather than operator-focused, this may be intentional — but it breaks the stated convention that "every LLM agent should register context tools."

### C-5.2: Three agents have zero test coverage [high]

**Files:** `agents/research.py`, `agents/code_review.py`, `agents/scout.py`

These three agents have no test files at all:

- **research.py** (181 LOC): Qdrant search tools, interactive REPL, system prompt builder — all untested.
- **code_review.py** (120 LOC): Model override logic, input routing (file/diff/stdin) — untested.
- **scout.py** (544 LOC): The most complex agent with web search, rate limiting, component registry YAML parsing, LLM evaluation, usage map building from Langfuse, notification, formatting — **zero tests for 544 lines**.

Scout is especially concerning given its external API dependency (Tavily) and the per-component error handling logic that should be verified.

### C-5.3: No test for briefing generate_briefing pipeline [medium]

**File:** `tests/test_briefing.py`

The test file covers schemas (4 tests), formatters (7 tests), and notifications (4 tests), totaling 15 tests. The core `generate_briefing()` function — which orchestrates activity collection, health checks, scout/digest report loading, goals section building, and LLM synthesis — has **zero** test coverage.

This is the most important function in the file (lines 115-241, 126 LOC). It handles multiple fallible data sources (Langfuse, health, scout JSON, digest JSON, goals) and should be tested at minimum with mocked dependencies.

### C-5.4: No test for digest generate_digest pipeline [medium]

**File:** `tests/test_digest.py`

Similar to briefing: the test file covers schemas, formatters, collectors, and notifications (28 tests). The `generate_digest()` function itself (lines 186-242) which orchestrates data collection and LLM synthesis is not tested.

### C-5.5: Scout notification uses subprocess instead of shared.notify [low]

**File:** `agents/scout.py:467-483`

Scout's `send_notification()` directly calls `subprocess.run(["notify-send", ...])` instead of using the unified `shared.notify.send_notification()` that briefing.py:325-342 and digest.py:325-341 both use.

```python
# scout.py:476-483
try:
    subprocess.run(
        ["notify-send", "--app-name=LLM Stack", "Horizon Scan", f"{summary}\n{body}"],
        timeout=5,
        capture_output=True,
    )
except (subprocess.TimeoutExpired, FileNotFoundError, OSError):
    pass
```

This means scout notifications are desktop-only (no ntfy push) while briefing and digest use the unified channel that sends to both ntfy and desktop. Inconsistent notification behavior.

### C-5.6: Code review agent re-creates agent on model override without context tools [medium]

**File:** `agents/code_review.py:86-92`

When `--model` is specified on CLI, the agent is recreated but context tools are NOT re-registered:

```python
if args.model != "balanced":
    global agent
    agent = Agent(
        get_model(args.model),
        deps_type=ReviewDeps,
        system_prompt=SYSTEM_PROMPT,
    )
    # Context tools NOT registered on new agent
```

The original agent (line 51-55) has context tools registered (lines 58-60), but the recreated agent does not. This means `--model fast` produces an agent without operator context tools.

---

## Correctness Findings

### R-5.1: Code review system prompt concatenation may crash on None [medium]

**File:** `agents/code_review.py:33`

```python
SYSTEM_PROMPT = get_system_prompt_fragment("code-review") + """\
You are a senior code reviewer...
```

If `get_system_prompt_fragment()` returns `None` or empty string, the concatenation would either crash (if None) or produce a valid but potentially poorly-formatted prompt (if empty). Looking at the implementation in `shared/operator.py:128+`, the function returns `str`, so it will always return a string. However, if the operator.json file is missing or malformed, it could return an empty string, producing a prompt that starts immediately with the instructions without any operator identity context.

This is safe but fragile — research.py:41-42 handles this more defensively:

```python
fragment = get_system_prompt_fragment("research")
if fragment:
    parts.append(fragment)
```

### R-5.2: Scout staleness check for scout-report.json is incomplete [medium]

**File:** `agents/briefing.py:137-156`

The briefing agent loads the scout report and comments "Load scout report if recent (< 7 days old)" but does NOT actually check the age:

```python
# Load scout report if recent (< 7 days old)
scout_section = ""
if SCOUT_REPORT.exists():
    try:
        scout_data = json.loads(SCOUT_REPORT.read_text())
        scout_ts = scout_data.get("generated_at", "")
        # NOTE: scout_ts is extracted but NEVER compared to current time
        actionable = [
            r for r in scout_data.get("recommendations", [])
            if r.get("tier") in ("adopt", "evaluate")
        ]
```

The `scout_ts` is used in the section header text but there is no `datetime.fromisoformat(scout_ts)` comparison against `datetime.now()` to actually enforce the 7-day freshness window. A 6-month-old scout report would still be included in the briefing.

### R-5.3: Scout eval_agent index() call will crash on invalid tier values [low]

**File:** `agents/scout.py:444-447`

```python
for rec in sorted(
    report.recommendations,
    key=lambda r: ["adopt", "evaluate", "monitor", "current-best"].index(r.tier)
    if r.tier in ["adopt", "evaluate", "monitor", "current-best"] else 99,
):
```

This is actually safe due to the ternary guard (`if r.tier in ... else 99`), but the ternary syntax is ambiguous. Due to Python operator precedence, this parses as:

```python
key=lambda r: (list.index(r.tier)) if (r.tier in list) else 99
```

Which is correct. No bug, but the complex lambda is hard to verify at a glance.

### R-5.4: Briefing overwrites stats from LLM with computed stats [correct, not a bug]

**File:** `agents/briefing.py:235-239`

```python
result = await briefing_agent.run(prompt)
briefing = result.output
briefing.generated_at = datetime.now(timezone.utc).isoformat()[:19] + "Z"
briefing.hours = hours
briefing.stats = stats
```

The LLM output (which has `output_type=Briefing`) may include stats in its structured output, but these are overwritten with the deterministic stats computed at lines 125-136. This is intentional and correct — the LLM should not be trusted to produce accurate numbers. Same pattern in digest.py:237-239. Good design.

### R-5.5: Research agent search_samples queries hardcoded collection name [low]

**File:** `agents/research.py:118-119`

```python
results = ctx.deps.qdrant.query_points(
    "samples",
```

The `search_knowledge_base` tool uses `ctx.deps.collection` (configurable via Deps), but `search_samples` hardcodes `"samples"`. This is likely intentional since the samples collection is a different entity, but it does mean that if the samples collection were renamed, only this one hardcoded reference would need updating.

---

## Robustness Findings

### B-5.1: No LLM failure handling in any agent [high]

**Files:** All 6 source files.

No agent catches exceptions from `agent.run()` / LLM calls. If LiteLLM's fallback chain is exhausted (all models down), the uncaught exception will:

- **briefing.py:235** `result = await briefing_agent.run(prompt)` — crash the daily briefing timer
- **digest.py:236** `result = await digest_agent.run(prompt)` — crash the daily digest timer
- **scout.py:272** `result = await eval_agent.run(prompt)` — caught per-component at line 375, but the full scout pipeline can still emit partial results
- **management_prep.py:295,317,331** — crash the prep generation
- **research.py:143,163** — crash user-facing REPL
- **code_review.py:72** — crash user-facing review

Scout is the only agent with per-component error handling (lines 375-377):

```python
except Exception as e:
    log.error(f"Failed to scan {spec.key}: {e}")
    report.errors.append(f"{spec.key}: {e}")
```

This is good — a failure evaluating one component doesn't prevent others from being scanned. But no other agent has any equivalent.

For timer-driven agents (briefing, digest), an unhandled LLM failure means the systemd service exits non-zero, triggering the `notify-failure@.service` template. This is acceptable as a failure notification path but means no briefing/digest is produced that day. A `try/except` around the LLM call with a degraded-mode output (stats-only briefing, raw data digest) would be more resilient.

### B-5.2: Scout Tavily API failure returns empty results silently [medium]

**File:** `agents/scout.py:152-165`

```python
try:
    with urlopen(req, timeout=15) as resp:
        data = json.loads(resp.read().decode())
        return [...]
except (URLError, TimeoutError, json.JSONDecodeError) as e:
    log.warning(f"Tavily search failed for '{query}': {e}")
    return []
```

This is graceful degradation — good. However:

1. **HTTP errors (403, 429, 500)** are caught via URLError, which is correct since HTTPError is a subclass of URLError.
2. **Rate limiting (429)** is not specifically handled — the `time.sleep(0.5)` at line 181 provides basic rate limiting between searches, but if Tavily returns 429, the scout will just log a warning and continue with empty results. There's no retry or backoff.
3. **API key invalid** produces an HTTP 401/403 which is caught and logged. The scout proceeds with "No search results found" for every component, producing a report full of low-confidence "current-best" recommendations. The report does NOT clearly indicate that web search was non-functional.

If all Tavily searches fail, `search_component()` returns `"No search results found."` which is passed to the LLM evaluator. The LLM would likely produce "current-best" with low confidence. The report itself has an `errors` field but search failures don't populate it — only per-component exceptions do.

### B-5.3: Briefing does not validate scout/digest JSON structure [medium]

**File:** `agents/briefing.py:139-178`

Scout report loading:
```python
scout_data = json.loads(SCOUT_REPORT.read_text())
scout_ts = scout_data.get("generated_at", "")
actionable = [
    r for r in scout_data.get("recommendations", [])
    if r.get("tier") in ("adopt", "evaluate")
]
```

Digest report loading:
```python
digest_data = json.loads(DIGEST_REPORT.read_text())
headline = digest_data.get("headline", "")
```

Both are wrapped in `try/except (json.JSONDecodeError, KeyError)` which provides basic protection. However, `TypeError` is NOT caught. If `scout_data.get("recommendations")` returned a non-iterable (e.g., a string), the list comprehension would raise `TypeError`. This is unlikely given the Pydantic serialization, but the catch block should include `TypeError` for defense in depth.

### B-5.4: Digest Qdrant scroll uses timestamp-based filtering without fallback [medium]

**File:** `agents/digest.py:71-111`

```python
def collect_recent_documents(hours: int = 24) -> list[dict]:
    since_ts = time.time() - (hours * 3600)
    try:
        client = get_qdrant()
        from qdrant_client.models import Filter, FieldCondition, Range
        results = client.scroll(
            collection_name="documents",
            scroll_filter=Filter(
                must=[FieldCondition(
                    key="ingested_at",
                    range=Range(gte=since_ts),
                )]
            ),
            limit=200,
            with_payload=True,
            with_vectors=False,
        )
```

The function relies on an `ingested_at` field existing in Qdrant payloads. If documents were ingested before this field was added to the RAG pipeline (or if the field is named differently), the filter would return zero results. The bare `except Exception` at line 110 handles Qdrant failures gracefully (returns `[]`), but there's no way to distinguish "nothing was ingested recently" from "the ingested_at field doesn't exist on any documents."

Additionally, `limit=200` is a hard cap. If more than 200 documents were ingested in the lookback window, only the first 200 are returned. The scroll API returns a next_page_offset for pagination, but it's not used.

### B-5.5: Management prep vault path dependency not configurable [low]

**File:** `agents/management_prep.py:37-43`

```python
from cockpit.data.management import (
    collect_management_state,
    PersonState,
    ManagementSnapshot,
    _parse_frontmatter,
    VAULT_PATH,
)
```

The vault path comes from `cockpit.data.management.VAULT_PATH`. The management_prep agent imports `VAULT_PATH` and uses it directly in `_read_recent_meetings()` at line 157. Tests correctly patch this (test_management_prep.py:98), but the actual value is determined by the management module, not by an environment variable in this agent. This is fine architecturally but means the agent cannot be used with a different vault path unless `VAULT_PATH` is patched at the management module level.

### B-5.6: Management prep handles missing person gracefully [correct]

**File:** `agents/management_prep.py:278-284`

```python
if person is None:
    return PrepDocument(
        summary=f"No active person note found for '{person_name}' in the vault.",
        suggested_topics=["Check that a person note exists in 10-work/people/"],
    )
```

Good — returns a helpful degraded PrepDocument instead of crashing. Similarly, `generate_team_snapshot()` at lines 302-306 handles the empty-team case.

### B-5.7: Management prep does not handle malformed frontmatter [correct]

**File:** `agents/management_prep.py:162`

```python
fm = _parse_frontmatter(md_file)
```

The underlying `_parse_frontmatter()` in `cockpit/data/management.py` handles all malformed cases (no frontmatter, invalid YAML, missing end marker, non-dict YAML, non-existent file) by returning `{}`. This was verified in `tests/test_management.py` (lines 48-94, 9 test cases). Management_prep then uses `.get()` on the result, so missing keys are handled safely. Good defensive design throughout the vault parsing chain.

### B-5.8: Research agent has no score threshold filtering [low]

**File:** `agents/research.py:89-106`

```python
results = ctx.deps.qdrant.query_points(
    ctx.deps.collection,
    query=query_vec,
    limit=5,
)

if not results.points:
    return "No relevant documents found in the knowledge base."

# Format results with source attribution
chunks = []
for p in results.points:
    filename = p.payload.get("filename", "unknown")
    text = p.payload.get("text", "")
    score = p.score
    chunks.append(f"[{filename}, relevance={score:.3f}]\n{text}")
```

All 5 results are returned regardless of relevance score. A query that has no semantically relevant documents will still return 5 results — just with low scores. The score is included in the output so the LLM can judge relevance, but there's no minimum threshold to prevent injecting truly irrelevant context. A threshold like `if p.score < 0.3: continue` would prevent noise.

The empty-result case (`if not results.points`) is handled, which is good.

---

## Focus Area Answers

### 1. Context Tool Registration

| Agent | Imports `get_context_tools`? | Registers tools? | Line Reference |
|-------|------------------------------|-------------------|----------------|
| research.py | Yes (line 73) | Yes (lines 74-75) | Consistent pattern |
| code_review.py | Yes (line 58) | Yes (lines 59-60) | Consistent pattern |
| briefing.py | Yes (line 110) | Yes (lines 111-112) | Consistent pattern |
| scout.py | Yes (line 236) | Yes (lines 237-238) | Consistent pattern |
| digest.py | **No** | **No** | Missing — C-5.1 |
| management_prep.py | Yes (line 136) | Yes (lines 137-140), all 3 agents | Consistent pattern |

**Verdict:** 5 of 6 agents register context tools. Digest is the outlier.

### 2. LLM Fallback Behavior

| Agent | Model Alias | LiteLLM Route | Catches LLM Failure? |
|-------|-------------|---------------|----------------------|
| research | `balanced` | claude-sonnet | No |
| code_review | `balanced` | claude-sonnet | No |
| briefing | `fast` | claude-haiku | No |
| scout | `balanced` | claude-sonnet | Yes (per-component, line 375) |
| digest | `fast` | claude-haiku | No |
| management_prep | `balanced` x2, `fast` x1 | claude-sonnet, claude-haiku | No |

LiteLLM handles transparent failover (claude-sonnet -> gemini-pro etc.), but if the entire chain fails, the exception propagates uncaught in 5 of 6 agents. See B-5.1.

### 3. Output Validation

Agents with `output_type` (Pydantic structured output):
- **briefing.py** (`Briefing`): Pydantic AI handles parsing/validation. Stats are overwritten post-LLM (line 239). Good.
- **scout.py** (`Recommendation`): Pydantic AI handles parsing. Component key is overwritten (line 275). Good.
- **digest.py** (`Digest`): Pydantic AI handles parsing. Stats overwritten (line 240). Good.
- **management_prep.py** (`PrepDocument`, `TeamSnapshot`, `ManagementOverview`): Pydantic AI handles parsing. No post-processing.

Agents with unstructured output:
- **research.py**: Returns raw string. No validation.
- **code_review.py**: Returns raw string. No validation.

**Verdict:** Structured agents rely on Pydantic AI's built-in validation (which includes retries on parse failure). The critical stats fields are overwritten with deterministic values — correct. Unstructured agents have no output validation, which is acceptable for their use case.

### 4. Briefing Data Freshness

**Input data sources** (`agents/briefing.py:115-233`):

| Data Source | Collection Method | Missing/Stale Behavior |
|-------------|-------------------|----------------------|
| Activity report | `generate_activity_report(hours)` (line 118) | Has `data_sources` field tracking availability |
| Health snapshot | `run_checks()` (line 121) | Live check, always current |
| Scout report | `SCOUT_REPORT.read_text()` (line 141) | **No freshness check** — R-5.2 |
| Digest report | `DIGEST_REPORT.read_text()` (line 162) | No freshness check (expected to be 15 min old) |
| Goals | `get_goals()` (line 196) | Reads operator.json, always current |

The activity report includes a `data_sources` object tracking which sources were available:
- Lines 182-191: Warnings are generated for unavailable Langfuse, missing health history, missing drift report
- These warnings are injected into the LLM prompt as "## Data Source Warnings"

**Verdict:** Health and activity data have explicit availability tracking. Scout report has no freshness check (R-5.2). Digest report has no freshness check but is expected to run 15 minutes before briefing. Missing files (scout, digest) are handled by `if SCOUT_REPORT.exists()` guards.

### 5. Scout Web Search

**Tavily call:** `agents/scout.py:129-165` — uses stdlib `urllib.request` (not the Tavily MCP or SDK).

| Failure Mode | Handling | Line |
|--------------|----------|------|
| TAVILY_API_KEY missing | Skip search, return `[]` | 131-133 |
| URLError (network, HTTP 4xx/5xx) | Log warning, return `[]` | 163-165 |
| TimeoutError (15s) | Log warning, return `[]` | 163-165 |
| JSON decode error | Log warning, return `[]` | 163-165 |
| Rate limit (429) | No specific handling, treated as URLError | — |
| All searches empty | "No search results found" → LLM evaluates | 183-184 |

**Rate limiting:** `time.sleep(0.5)` between search hints (line 181). No exponential backoff on 429.

**API key loading:** Environment variable first, then `pass show api/tavily` via subprocess (lines 507-517). Good fallback chain.

**Verdict:** Error handling is present and graceful (no crashes), but there's no retry logic, no backoff on rate limits, and no report-level indication that web search was degraded. See B-5.2.

### 6. Management Prep Vault Dependency

| Scenario | Handling | Line |
|----------|----------|------|
| No people notes in vault | Returns `PrepDocument` with helpful message | 280-284 |
| No meetings directory | Returns `[]` from `_read_recent_meetings` | 157-158 |
| Malformed frontmatter | `_parse_frontmatter()` returns `{}`, `.get()` defaults safe | management.py |
| Person not found by name | Returns degraded `PrepDocument` | 280-284 |
| Empty team | Returns `TeamSnapshot` with "No active people" | 302-306 |
| Meeting file unreadable | Try/except returns `""` for content | 173-175 |
| Non-existent vault path | `collect_management_state()` returns empty snapshot | management.py tests confirm |

**Verdict:** Excellent defensive design. Every vault failure mode degrades gracefully rather than crashing.

### 7. Digest Content Collection

**Mechanism:** `agents/digest.py:71-111` — Qdrant scroll with timestamp filter.

```python
since_ts = time.time() - (hours * 3600)
results = client.scroll(
    collection_name="documents",
    scroll_filter=Filter(must=[FieldCondition(key="ingested_at", range=Range(gte=since_ts))]),
    limit=200,
)
```

| Scenario | Behavior |
|----------|----------|
| Nothing ingested recently | Returns `[]`, digest reports "no new documents" |
| Qdrant down | `except Exception: return []` — silent failure |
| >200 documents in window | First 200 returned, rest silently dropped (B-5.4) |
| `ingested_at` field missing | Filter returns `[]` (no points match) |
| Vault inbox missing | `VAULT_INBOX.is_dir()` check returns `[]` |

Results are grouped by source file (deduplication) and include chunk counts, which is good for the digest summary.

**Verdict:** Reasonable for the use case. The 200-doc cap is the main concern for heavy ingestion periods (B-5.4).

### 8. Research Agent Quality

**Qdrant query:** `agents/research.py:86-106`

```python
query_vec = embed(query, model=ctx.deps.embedding_model)
results = ctx.deps.qdrant.query_points(
    ctx.deps.collection,
    query=query_vec,
    limit=5,
)
```

| Aspect | Assessment |
|--------|-----------|
| Embedding | Uses Ollama nomic-embed with correct `search_query` prefix (via `embed()`) |
| Collection | Configurable via `Deps.collection` (default "documents") |
| Result limit | Hardcoded 5 |
| Score filtering | **None** — all 5 returned regardless of score (B-5.8) |
| Empty handling | Returns "No relevant documents found" message |
| Source attribution | Includes filename and relevance score in output |
| Ranking | Qdrant handles ranking by vector similarity |

**Verdict:** Functional but basic. No score threshold, no metadata filtering, no reranking. The score is visible to the LLM which provides a soft quality signal.

---

## Test Coverage Assessment

### Tested Functions

| Function | Test File | Tests | Coverage Quality |
|----------|-----------|-------|-----------------|
| `BriefingStats`, `ActionItem`, `Briefing` schemas | test_briefing.py | 4 | Good — defaults, round-trip |
| `format_briefing_human()` | test_briefing.py | 5 | Good — headline, stats, actions, commands, no-actions |
| `format_briefing_md()` | test_briefing.py | 4 | Good — headers, stats, priority order, edge cases |
| `briefing.send_notification()` | test_briefing.py | 4 | Good — calls, content, edge cases, failure |
| `DigestStats`, `NotableItem`, `Digest` schemas | test_digest.py | 5 | Good — defaults, round-trip |
| `format_digest_human()` | test_digest.py | 6 | Good — headline, stats, notable, actions, empties |
| `format_digest_md()` | test_digest.py | 5 | Good — headers, stats, items, empties, edge cases |
| `collect_recent_documents()` | test_digest.py | 3 | Good — grouped, empty, error |
| `collect_vault_inbox()` | test_digest.py | 2 | Good — nonexistent, recent files |
| `collect_collection_stats()` | test_digest.py | 2 | Good — success, partial failure |
| `digest.send_notification()` | test_digest.py | 3 | Good — calls, content, edge case |
| `_find_person()` | test_management_prep.py | 3 | Good — exact, case-insensitive, not found |
| `_read_recent_meetings()` | test_management_prep.py | 4 | Good — filename, attendees, missing dir, limit |
| `_collect_person_context()` | test_management_prep.py | 3 | Good — basic, coaching, feedback |
| `_collect_team_context()` | test_management_prep.py | 1 | Minimal |
| `generate_1on1_prep()` | test_management_prep.py | 2 | Good — not found, found (mocked LLM) |
| `generate_team_snapshot()` | test_management_prep.py | 2 | Good — empty, with people (mocked LLM) |
| `format_prep_md()` | test_management_prep.py | 1 | Adequate |
| `format_snapshot_md()` | test_management_prep.py | 1 | Adequate |
| `format_overview_md()` | test_management_prep.py | 1 | Adequate |

### Untested Functions (in tested agents)

| Function | File | Reason |
|----------|------|--------|
| `generate_briefing()` | briefing.py | Core pipeline — complex, multiple data sources |
| `generate_digest()` | digest.py | Core pipeline — LLM synthesis |
| `generate_overview()` | management_prep.py | Not tested (only team_snapshot and 1on1_prep tested) |
| `briefing.main()` | briefing.py | CLI entry point |
| `digest.main()` | digest.py | CLI entry point |
| `management_prep.main()` | management_prep.py | CLI entry point |

### Untested Agents (no test file)

| Agent | LOC | Key Untested Logic |
|-------|-----|-------------------|
| research.py | 181 | `search_knowledge_base`, `search_samples`, `query()`, `interactive()`, system prompt builder |
| code_review.py | 120 | `review()`, model override, stdin/file/diff input routing |
| scout.py | 544 | `load_registry()`, `_tavily_search()`, `search_component()`, `evaluate_component()`, `_build_usage_map()`, `run_scout()`, all formatters, notification |

### Test Quality Notes

1. **Mocking pattern is consistent:** All tests use `unittest.mock.patch` appropriately for external dependencies.
2. **Pydantic AI agent calls are mocked correctly:** test_management_prep.py uses `AsyncMock` for `agent.run()`.
3. **No integration tests:** All tests are unit tests with mocked externals. No tests verify actual Qdrant queries, LLM responses, or file I/O against real services.
4. **test_management.py** (387 LOC, related) provides excellent coverage of the vault parsing layer that management_prep depends on — 9 frontmatter tests, 10 staleness tests, 4 typed notes tests, 7 people collection tests, 4 coaching tests, 3 feedback tests, 3 state assembly tests.

---

## Summary

### Finding Counts by Severity

| Severity | Completeness | Correctness | Robustness | Total |
|----------|-------------|-------------|------------|-------|
| Critical | 0 | 0 | 0 | **0** |
| High | 1 (C-5.2) | 0 | 1 (B-5.1) | **2** |
| Medium | 4 (C-5.1, C-5.3, C-5.4, C-5.6) | 2 (R-5.1, R-5.2) | 3 (B-5.2, B-5.3, B-5.4) | **9** |
| Low | 1 (C-5.5) | 2 (R-5.3, R-5.5) | 2 (B-5.5, B-5.8) | **5** |
| **Total** | **6** | **4** | **6** | **16** |

### Key Concerns

1. **Test coverage gaps are significant.** Three agents (research, code_review, scout) have zero tests — totaling 845 LOC untested. Scout is the highest risk due to its external API dependency and complexity.

2. **No LLM failure handling.** If the LiteLLM fallback chain is exhausted, 5 of 6 agents crash. The two timer-driven agents (briefing, digest) would fail silently on their systemd schedule. The `notify-failure@.service` template provides notification, but no degraded output is produced.

3. **Scout report freshness is not enforced.** The briefing agent documents a 7-day freshness window for scout data but does not implement it. A stale scout report could produce misleading action items in the daily briefing.

4. **Digest agent is the only agent missing context tools.** This breaks the post-refactoring convention. Whether this is intentional or an oversight should be clarified.

5. **Code review model override drops context tools.** The `--model` flag creates a new agent without re-registering context tools, silently degrading the agent's capability.

### Positive Observations

- **Vault parsing is extremely robust.** The management data collection chain (`_parse_frontmatter` -> `_collect_people` -> `collect_management_state`) handles every conceivable failure mode. Well-tested.
- **Deterministic stats override LLM output.** Briefing and digest correctly overwrite LLM-generated stats with computed values — not trusting the LLM for numbers.
- **Scout per-component error isolation.** A failure in one component evaluation doesn't prevent others from being scanned.
- **Consistent agent architecture.** All agents follow the same pattern: data collection (zero LLM) -> prompt assembly -> LLM synthesis -> format -> save/display. Clean separation of concerns.
- **Notification unification** (for briefing and digest via shared.notify) provides multi-channel delivery.
