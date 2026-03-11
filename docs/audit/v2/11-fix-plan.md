# Fix Plan — Audit v2

**Date:** 2026-03-03
**Source:** 10 audit documents (D1–D9 + Holistic), ~113 findings
**Method:** Prioritized by operator impact, grouped by blast radius, sized for safe batch execution

---

## Aggregate Findings

| Domain | Total | High | Medium | Low |
|--------|-------|------|--------|-----|
| D1 Shared Foundation | 12 | 0 | 1 | 11 |
| D2 Data Ingestion | 14 | 0 | 2 | 12 |
| D3 Operator Profile | 16 | 0 | ~4 | ~12 |
| D4 Health & Observability | 17 | 0 | 3 | 14 |
| D5 Intelligence Agents | 10 | 0 | 2 | 8 |
| D6 Cockpit TUI | 9 | 1 | 2 | 6 |
| D7 Web Layer | 6 | 0 | 1 | 5 |
| D8 Infrastructure | 6 | 0 | 1 | 5 |
| D9 Documents | 13 | 0 | 4 | 9 |
| H Holistic | 10 | 2 | 5 | 3 |
| **Total** | **~113** | **3** | **~25** | **~85** |

The 3 HIGH findings (R2-6.1, H2-3.1, H2-4.1) all trace to the same root cause: two bugs in `cockpit/data/decisions.py` introduced during the v1 fix session.

---

## Fix Tiers

### Tier 0: Critical Path — decisions.py (2 bugs, 1 file)

**Priority:** Immediate. These are the only HIGH findings in the entire audit.
**Batch size:** 1 file, 2 one-line fixes + tests
**Risk:** Minimal — both are isolated to `decisions.py`
**Domains:** D6, H

| # | Finding | Fix | LOC |
|---|---------|-----|-----|
| F-0.1 | R2-6.1: Missing `import os` in decisions.py | Add `import os` to imports | 1 |
| F-0.2 | R2-6.2: `a.key` should be `a.id` | Change `a.key` → `a.id` at line 62 | 1 |

**Why this is Tier 0:** These two bugs break the accommodation effectiveness tracking flow — the system's core executive function feedback loop. The accommodation system works (propose → confirm → activate), but the decision recorder never captures which accommodations were active, so the profiler can never correlate accommodation states with decision patterns. The operator's most personal infrastructure is silently non-functional.

**Test additions:**
- Test rotation triggers after 500+ lines (exercises the `os` import)
- Test `active_accommodations` populated from active `Accommodation` objects (exercises `a.id`)
- Mock active accommodations in decision recording test

**Holistic findings resolved:** H2-3.1, H2-4.1

---

### Tier 1: Data Integrity — Profile Corruption Chain (3 fixes, 2 files)

**Priority:** High. Self-reinforcing failure path: interrupted write → silent corruption → crash → no recovery.
**Batch size:** 2 files, focused changes + tests
**Risk:** Low — all changes are additive (backup, validation, atomic write)
**Domains:** D3

| # | Finding | Fix | LOC |
|---|---------|-----|-----|
| F-1.1 | Fix 27 (not implemented): `regenerate_operator()` no validation | Add pre-write size check + backup + atomic write | ~15 |
| F-1.2 | R2-3.1: `load_existing_profile()` swallows all exceptions | Log at `warning` level, distinguish corruption from absence | ~5 |
| F-1.3 | B2-3.2: `regenerate_operator()` crashes on corrupt operator.json | Add try/except around `json.loads`, fall back to backup if available | ~10 |

**Connected chain:** F-1.1 prevents corruption. F-1.2 makes corruption visible. F-1.3 recovers from corruption. All three are needed to close the self-reinforcing loop.

**Test additions:**
- Test `regenerate_operator()` with valid JSON → atomic write succeeds
- Test `regenerate_operator()` with corrupt JSON → graceful recovery from backup
- Test `load_existing_profile()` with corrupt JSON → warning logged, returns None
- Test pre-write size check rejects empty/tiny LLM output

---

### Tier 2: Silent Data Bugs (3 fixes, 3 files)

**Priority:** Medium. Each bug silently degrades data quality without visible symptoms.
**Batch size:** 3 files, independent fixes
**Risk:** Low — each is a localized correction
**Domains:** D4, D6

| # | Finding | Fix | LOC |
|---|---------|-----|-----|
| F-2.1 | R2-4.1: `_manifest_age()` reads `generated_at` instead of `timestamp` | Change key name | 1 |
| F-2.2 | R2-6.3: `end_interview()` clears state before confirming flush | Move `self.interview_state = None` inside success branch | ~5 |
| F-2.3 | R2-2.1: `services_processed` includes partially-failed services | Track success/failure separately in `ProcessResult` | ~10 |

**F-2.1 detail:** The briefing agent never knows how old the infrastructure manifest is. The operator's daily briefing could reference weeks-old data with no staleness warning. One-line fix: `data.get("timestamp", "")`.

**F-2.2 detail:** If `flush_interview_facts()` fails, the operator loses accumulated interview data with no recovery. Move the state clearing to after confirmed flush success.

**Test additions:**
- Test `_manifest_age()` returns actual age string from manifest with `timestamp` key
- Test `end_interview()` preserves state on flush failure
- Test `ProcessResult.services_processed` excludes services with mid-processing errors

---

### Tier 3: Silent Failure Patterns (4 fixes, 4 files)

**Priority:** Medium. Cross-cutting anti-pattern: `except Exception: pass` hiding real failures.
**Batch size:** 4 files, independent fixes — can be parallelized
**Risk:** Low — adding logging, not changing control flow
**Domains:** D1, D3, D4, D8

| # | Finding | Fix | LOC |
|---|---------|-----|-----|
| F-3.1 | B2-1.1: `langfuse_get()` returns `{}` for all failures | Return `LangfuseResult` named tuple with `data`, `error`, `available` fields | ~25 |
| F-3.2 | B2-3.1: Git reader bypasses `_read_capped()` error isolation | Wrap `read_git_info()` call in try/except in `read_all_sources()` | ~5 |
| F-3.3 | B2-4.2: `knowledge_maint` swallows Qdrant errors silently | Add `errors_encountered` counter to report; surface in CLI output | ~15 |
| F-3.4 | R2-8.2: `.envrc` silently falls back to empty on pass failure | Add `pass ls` guard (matching `generate-env.sh` pattern) and warning | ~5 |

**F-3.1 scope:** This is the largest single fix. Callers (`activity_analyzer.py`, `profiler_sources.py`) must be updated to handle the new return type. The change is backwards-compatible if callers use `.data` (dict) instead of the raw return.

**Test additions:**
- Test `langfuse_get()` returns distinct results for missing credentials vs HTTP error vs success
- Test `read_all_sources()` continues after git subprocess failure
- Test `knowledge_maint` report includes error count when Qdrant fails
- Verify `.envrc` guard by inspection (no automated test — infrastructure config)

---

### Tier 4: Atomic Write Consistency (3 fixes, 3 files)

**Priority:** Medium-low. Cross-cutting pattern from H2-2.1.
**Batch size:** 3 files, same pattern applied
**Risk:** Very low — well-established tempfile+rename pattern
**Domains:** D6, D1, D8 (holistic H2-2.1)

| # | Finding | Fix | LOC |
|---|---------|-----|-----|
| F-4.1 | H2-2.1 / B2-6.1: `micro_probes.py` `save_state()` not atomic | Apply tempfile + os.replace pattern | ~8 |
| F-4.2 | H2-2.1 / R2-1.3: `vault_writer.py` non-atomic writes | Apply tempfile + os.replace to `write_to_vault()` | ~8 |
| F-4.3 | H2-2.1 / B2-8.1: health-watchdog fix-attempts.json not atomic | Apply atomic write pattern | ~8 |

**Pattern:** All three should use the same pattern already applied in `accommodations.py:124-131` and `chat.py:250-261`.

---

### Tier 5: Robustness Improvements (4 fixes)

**Priority:** Low-medium. Real improvements but low blast radius.
**Batch size:** Independent, can be split across sessions
**Risk:** Low
**Domains:** D2, D4, D7

| # | Finding | Fix | LOC |
|---|---------|-----|-----|
| F-5.1 | B2-4.1: Watchdog retries with no backoff | Add attempt counter (3-strike pause) to health-watchdog fix state | ~15 |
| F-5.2 | B2-2.1: Location Records.json loaded entirely into memory | Add streaming JSON parser or sample-and-warn for files >200MB | ~20 |
| F-5.3 | R2-2.3: `_yaml_list()` doesn't escape embedded double-quotes | Escape `"` → `\"` inside quoted strings | ~3 |
| F-5.4 | C2-7.1: No rate limiting or auth on cockpit API | Add optional API key middleware (consistent with Obsidian plugin pattern) | ~20 |

---

### Tier 6: Test Coverage Gaps (6 items)

**Priority:** Low-medium. Important for regression prevention but no immediate impact.
**Batch size:** Can be spread across sessions
**Risk:** None — additive only
**Domains:** D3, D5

| # | Finding | Tests to Add | Est. LOC |
|---|---------|-------------|----------|
| F-6.1 | C2-3.1: Profiler pipeline orchestration | Integration tests for `run_auto()`, `run_extraction()` with mocked LLM | ~80 |
| F-6.2 | C2-3.2: `regenerate_operator()` untested | Test manifest update pipeline + `_regenerate_operator_md()` | ~50 |
| F-6.3 | C2-5.1: research.py lacks dedicated tests | Test `search_knowledge_base()`, `query()`, `_build_system_prompt()` | ~60 |
| F-6.4 | C2-5.3: Scout pipeline untested | Test `evaluate_component()`, `run_scout()` with mocked LLM + Tavily | ~80 |
| F-6.5 | C2-5.2: `generate_overview()` untested | Test management overview generation | ~20 |
| F-6.6 | C2-5.4: No CLI main() tests for D5 agents | Test argparse handling, file output, notification dispatch | ~60 |

---

### Tier 7: Documentation Updates (13 items)

**Priority:** Low. No code impact, but important for operator reference accuracy.
**Batch size:** All in one session — text-only changes
**Risk:** None
**Domains:** D9

| # | Finding | Document | Fix |
|---|---------|----------|-----|
| F-7.1 | A-9.1 | Multiple | Standardize health check description to "11 groups, variable check count" |
| F-7.2 | A-9.2 | `ai-agents/README.md` | Update test count to current number |
| F-7.3 | A-9.5 | `operations-manual.md` | Add `models`, `auth`, `connectivity` to check group list |
| F-7.4 | P-9.1 | `operations-manual.md` | Add management-prep, digest, knowledge-maint agent sections |
| F-7.5 | P-9.1 | `operations-manual.md` | Add digest.timer and knowledge-maint.timer to timer table |
| F-7.6 | K-9.1 | `operations-manual.md` | Update agent count to 12 |
| F-7.7 | U-9.1 | `agent-architecture.md` | Mark as historical or update to reflect current state |
| F-7.8 | P-9.2 | `agent-architecture.md` | Move digest + knowledge-maint from "Planned" to "Implemented" |
| F-7.9 | A-9.3 | `agent-architecture.md` | Remove sample-watch from Tier 3 diagram |
| F-7.10 | A-9.4 | `agent-architecture.md` | Correct knowledge-maint description (systemd, not n8n) |
| F-7.11 | P-9.3 | `agent-architecture.md` | Resolve or remove Open Questions section |
| F-7.12 | K-9.2 | `ai-agents/README.md` | Fix timer table (obsidian-sync is desktop app, not timer) |
| F-7.13 | H2-1.1 | New or in CLAUDE.md | Add document authority hierarchy note |

---

### Tier 8: Low-Priority Carry-Forwards

Items acknowledged but deferred. Each is individually low-impact and defensible as-is.

**Code quality (no functional impact):**
- C2-1.1: `validate_embed_dimensions()` dead code — remove or wire into startup
- C2-1.2: `get_qdrant()` no reset API — add `reset_qdrant()` if needed
- C2-1.3: `langfuse_config.py` untested (P3 carry-forward)
- C2-1.4: `vault_utils.py`/`email_utils.py` indirect-only tests (P3)
- R2-1.1: `OperatorSchema` minimal validation — acceptable for single-operator
- R2-1.2: Module-level constants frozen at import — standard Python
- R2-1.4: `VAULT_PATH` import-time computation — standard Python
- B2-1.2: `validate_embed_dimensions()` not in startup — covered by per-call checks
- B2-1.3: `_qdrant_client` singleton not thread-safe — GIL protection sufficient
- B2-1.4: `notify.py` outer catch at debug level — correct behavior
- C2-6.1: Orphaned InfraPanel/ScoutPanel widgets (~30 LOC dead code)
- C2-6.2: `/accommodate` proposal path missing — feature gap, not bug
- R2-4.3: `--dry-run` dead code in knowledge_maint — confusing but safe
- C2-3.4: `store_to_qdrant()` legacy code — remove or document
- C2-2.4: Profiler bridge zero-embedding — not a finding (correct design)

**Edge cases (low probability, graceful degradation):**
- R2-2.2: Proton `records_skipped` residual calculation
- R2-2.3: `_yaml_list()` embedded quotes (moved to Tier 5)
- R2-2.4: Location parser strips timezone
- R2-2.5: Gmail parser epoch fallback
- R2-4.2: `gdrive_sync_freshness` doesn't check freshness (service not yet created)
- R2-4.4: Near-duplicate samples first 500 points
- R2-4.5: Stale source detection on temp unmount (mitigated by dry-run)
- R2-3.2: `load_structured_facts()` discards malformed items
- R2-3.3: `save_facts()` can write empty array
- R2-3.4: `_feedback_facts` defaults direction to "given"
- R2-3.5: `run_extraction()` skips curation
- R2-5.1: Briefing scout timestamp fragile parsing
- R2-5.2: Digest scroll limit=200
- B2-2.2: No RAG ingest rate limiting (Ollama is bottleneck)
- B2-2.3: Gemini LLM export parser speculative
- B2-2.4: Progress tracker no concurrent run handling
- B2-2.5: Gmail MBOX temp file on crash
- B2-3.3: Operator cache never expires
- B2-3.4: coaching/feedback rglob entire vault
- B2-3.5: Langfuse reader up to 4000 items
- B2-3.6: Stale point cleanup scope limited
- B2-4.3: Langfuse pagination no page limit
- B2-4.4: `run_fixes()` bash -c interpolation
- B2-4.5: Drift detector truncates at 8000 chars
- B2-4.6: `find_near_duplicates()` blocks event loop
- B2-4.7: `http_get()` returns 0 for all errors
- B2-5.1: Scout pass retrieval silent swallow
- B2-5.2: Scout search failure indistinguishable from empty
- B2-5.3: Management prep truncation no indicator
- B2-5.4: Research no score threshold
- B2-6.2: Silent exception swallowing (multiple locations)
- B2-6.3: Chat streaming no explicit timeout
- B2-6.4: decisions.py orphaned temp files (resolved by F-0.1)
- R2-7.1: Dockerfile binds to 0.0.0.0
- R2-7.2: health/history endpoint no cache-age header
- C2-7.2: No structured error envelope
- B2-7.1: SPA catch-all route ordering fragile
- B2-7.2: Obsidian chat history unbounded
- R2-8.1: generate-env.sh unquoted values
- C2-8.1: Inconsistent notification dispatch (holistic H2-2.2)
- C2-8.2: No systemd hardening (v1 carry-forward)
- B2-8.2: n8n credential backup unencrypted
- C2-4.1: Check count documentation cosmetic
- C2-4.2: Introspect missing Docker volumes
- C2-4.3: No tests for run_fixes()
- C2-4.4: No tests for generate_manifest()
- C2-4.5: collect_service_events() untested
- C2-2.1: No dedicated Gemini parser
- C2-2.2: Photos parser minimal metadata
- C2-2.3: Proton no --resume
- C2-3.3: Six reader functions untested
- C2-3.5: generate_digest() drops non-standard dimensions
- H2-2.2: Notification dispatch fragmentation
- H2-3.2: Micro-probe state dead end
- H2-4.2: System complexity approaches attention budget
- H2-5.1: No cross-domain contract tests
- H2-5.3: Drift detector scope gap

---

## Execution Schedule

### Session A: Tier 0 + Tier 2 (decisions.py + silent data bugs)

**Scope:** 4 fixes in 4 files
**Est. LOC changed:** ~20
**Est. test LOC:** ~60

1. Fix `decisions.py`: add `import os`, change `a.key` → `a.id` (F-0.1, F-0.2)
2. Fix `activity_analyzer.py`: `generated_at` → `timestamp` (F-2.1)
3. Fix `chat_agent.py`: reorder `end_interview()` state clearing (F-2.2)
4. Add tests for all three fixes
5. Run full test suite
6. Commit

**Why combined:** Tier 0 is 2 lines. Tier 2.1 is 1 line. All are isolated single-line or small corrections. Total blast radius is tiny. Combining keeps the session productive without exceeding safe batch size.

### Session B: Tier 1 (profile corruption chain)

**Scope:** 3 fixes in 2 files
**Est. LOC changed:** ~30
**Est. test LOC:** ~80

1. Add atomic write + pre-write validation to `regenerate_operator()` (F-1.1)
2. Add warning logging to `load_existing_profile()` (F-1.2)
3. Add try/except + backup recovery to `regenerate_operator()` JSON loading (F-1.3)
4. Add tests for corruption → recovery → backup paths
5. Run full test suite
6. Commit

**Why separate:** Profile modification has the widest blast radius of any fix. The operator's self-model is at stake. Dedicated session with thorough testing.

### Session C: Tier 3 (silent failure patterns)

**Scope:** 4 fixes in 4 files
**Est. LOC changed:** ~50
**Est. test LOC:** ~40

1. Refactor `langfuse_get()` return type (F-3.1) + update callers
2. Wrap git reader in error isolation (F-3.2)
3. Add error counting to knowledge_maint report (F-3.3)
4. Add pass guard to .envrc (F-3.4)
5. Run full test suite
6. Commit

**Note:** F-3.1 is the largest single fix and touches multiple files. If context budget is tight, split it into its own session.

### Session D: Tier 4 + Tier 5 (atomic writes + robustness)

**Scope:** 7 fixes across 6 files
**Est. LOC changed:** ~80
**Est. test LOC:** ~30

1. Apply atomic write pattern to micro_probes, vault_writer, health-watchdog (F-4.1–F-4.3)
2. Add watchdog backoff (F-5.1)
3. Fix _yaml_list() quote escaping (F-5.3)
4. Run full test suite
5. Commit

**Deferred from this session:** F-5.2 (location streaming) and F-5.4 (API auth) are larger scope changes that should get their own sessions if prioritized.

### Session E: Tier 6 (test coverage)

**Scope:** Test-only additions
**Est. test LOC:** ~350

1. Profiler pipeline integration tests (F-6.1, F-6.2)
2. Research/code_review dedicated tests (F-6.3)
3. Scout pipeline tests (F-6.4)
4. Run full test suite
5. Commit

**Note:** This can be split into multiple sessions. Prioritize F-6.1 (profiler pipeline) and F-6.4 (scout pipeline) as they cover the most complex untested code.

### Session F: Tier 7 (documentation)

**Scope:** Text-only changes across 4-6 documents
**Est. LOC changed:** ~200

1. Update operations-manual.md (F-7.3 through F-7.6)
2. Update ai-agents/README.md (F-7.2, F-7.12)
3. Decide: update or mark agent-architecture.md as historical (F-7.7 through F-7.11)
4. Standardize health check descriptions (F-7.1)
5. Add document authority note (F-7.13)
6. Commit

---

## Execution Principles (lessons from v1)

1. **Small batches.** The v1 session applied 89 fixes in one night. Fix quality degraded in later batches (D3: 25% success, D6: introduced bugs). No session should exceed ~10 fixes.

2. **Test before commit.** The decisions.py bugs (R2-6.1, R2-6.2) were introduced because the rotation and accommodation paths had no tests. Every fix in this plan includes test additions.

3. **Operator-critical first.** Tier 0 fixes the operator's executive function feedback loop. Tier 1 protects the operator's self-model. Everything else is infrastructure.

4. **One commit per session.** No multi-commit sessions. If a fix fails testing, remove it from the batch rather than committing partial work.

5. **No bare except.** The v2 audit identified `except Exception: pass` as a cross-cutting anti-pattern (Pattern 1 in holistic findings). New fixes must not introduce new bare exception handlers without logging.

6. **Verify atomicity.** Six modules need atomic writes. Four different patterns exist. All new writes must use the tempfile + os.replace pattern from `accommodations.py:124-131`.

---

## Fix Count Summary

| Tier | Fixes | Est. LOC | Priority |
|------|-------|----------|----------|
| T0: Critical path | 2 | ~2 + ~60 test | Immediate |
| T1: Data integrity | 3 | ~30 + ~80 test | High |
| T2: Silent data bugs | 3 | ~16 + ~30 test | Medium |
| T3: Silent failures | 4 | ~50 + ~40 test | Medium |
| T4: Atomic writes | 3 | ~24 + ~20 test | Medium-low |
| T5: Robustness | 4 | ~58 + ~20 test | Low-medium |
| T6: Test coverage | 6 | ~350 test | Low-medium |
| T7: Documentation | 13 | ~200 docs | Low |
| T8: Carry-forwards | ~60 | deferred | Low |
| **Active total** | **38** | **~180 + ~600 test + ~200 docs** | — |

38 fixes across 6 sessions, plus 1 documentation session. ~60 items acknowledged and deferred to Tier 8.

The two most impactful fixes in the entire plan are F-0.1 and F-0.2: adding `import os` and changing `a.key` to `a.id` in `decisions.py`. Combined: 2 lines of code to restore the operator's executive function feedback loop.
