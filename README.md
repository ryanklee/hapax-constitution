# hapax-constitution

A governance architecture for LLM agent systems, specified as a pattern with two reference implementations.

## Problem

LLM agents that manage personal infrastructure — preparing meeting context, maintaining knowledge bases, monitoring system health, reacting to filesystem changes — accumulate autonomy without accumulating constraints. Prompt-level instructions are advisory. They degrade under context pressure, cannot be audited, and provide no mechanism for handling cases the author didn't anticipate.

This project specifies a constitutional pattern for agent governance: formal axioms with weighted enforcement, derived implications at graduated tiers, an interpretive canon for ambiguous cases, and sufficiency probes that check obligations (what the system must do) rather than only prohibitions (what it must not).

## Design context

The two systems that implement this pattern externalize executive function into infrastructure. Their operator has ADHD and autism, which makes the cognitive cost of tracking open loops, maintaining relational context, and noticing staleness patterns acute. But the underlying problem is general: knowledge workers perform substantial executive function work that produces no deliverables, scales poorly with attention, and compounds silently when neglected. The constitution exists because agents doing this work require constraints that are structural, not advisory.

## The pattern

Four interlocking mechanisms compose the architecture.

### Filesystem-as-bus

All state lives as markdown files with YAML frontmatter on disk. Directories are collections. Agents read files to gather context and write files to produce output. This yields human-readable state (open any file in a text editor), git-native history (every state transition is a diff), tool-agnostic interoperability (any language reads markdown), and graceful degradation (if the engine is down, the data is still there).

The pattern predates and is now validated by two independent formalizations: "From Everything-is-a-File to Files-Are-All-You-Need" (arXiv:2601.11672, January 2026) and "Everything is Context: Agentic File System Abstraction for Context Engineering" (arXiv:2512.05470, December 2025). Unlike the virtual filesystem abstractions those papers propose, this pattern uses literal files on disk — debuggable with `cat`, `grep`, `diff`, and `git log`.

### Agent architecture

Three tiers, differentiated by invocation model and autonomy:

- **Tier 1 (Interactive):** Claude Code with MCP tools and system cockpit. Full operator supervision.
- **Tier 2 (On-demand):** Pydantic AI agents invoked by CLI, API, or Tier 1. Stateless per-invocation; all persistent state lives on the filesystem or in vector storage.
- **Tier 3 (Autonomous):** systemd timers running Tier 2 agents on schedules. High-frequency agents (health monitoring, knowledge maintenance) are deterministic with zero LLM calls.

Agents never invoke other agents. Orchestration is flat. They communicate through filesystem artifacts, which decouples them temporally and makes the coordination graph auditable through standard Unix tools.

### Axiom governance

Constitutional axioms are weighted constraints defined in YAML. Each axiom produces derived implications at graduated enforcement tiers:

| Tier | Enforcement | Meaning |
|------|-------------|---------|
| T0 | Block | Existential violation. Code matching this pattern must not exist. |
| T1 | Review | Significant constraint. Requires operator awareness. |
| T2 | Warn | Quality preference. Should be followed absent reason not to. |

Each implication carries an **interpretive canon** — a classification borrowed from legal theory that governs how the implication is applied to unforeseen cases:

- **Textualist:** Apply the literal text. If the implication says "no auth," there is no auth.
- **Purposivist:** Apply the axiom's intent. If the purpose is reducing cognitive load, evaluate whether the code increases it.
- **Absurdity:** Reject literal application if the result contradicts the axiom's purpose.
- **Omitted-case:** The axiom is silent. Apply the nearest precedent or escalate to the operator.

The interpretive canon addresses a problem that formal constraint systems share with legal codes: specification cannot anticipate every case. Rather than expanding the specification indefinitely, the canon provides a principled method for applying existing axioms to new situations.

Each implication also carries a **mode** — either `prohibition` (the system must NOT do this) or `sufficiency` (the system MUST do this). Most governance systems check only for violations. Sufficiency probes check obligations: does the system provide actionable error messages? Does it persist state across restarts? Does it automate routine work? This inversion — checking what SHOULD exist, not just what SHOULDN'T — is absent from the agent governance literature (ArbiterOS, ABC, PCAS, GaaS) as of early 2026.

A **precedent store** records edge-case rulings, building a common-law layer that makes axiom interpretation consistent and auditable across time.

### Reactive engine

An inotify watcher monitors the filesystem bus. File changes produce enriched events (including document type from YAML frontmatter). Rules — pure functions mapping events to actions — evaluate against each change. Multiple rules can fire; duplicate actions collapse.

Actions execute in phases:
- **Phase 0 (deterministic):** Cache refreshes, metric recalculation, file indexing. Unlimited concurrency. Zero cost.
- **Phase 1+ (LLM):** Synthesis, summarization, evaluation. Semaphore-bounded to prevent GPU saturation or API cost runaway.

Self-trigger prevention ensures that files written by the engine do not re-trigger evaluation. Notification delivery batches on a configurable interval to prevent storms.

## Axioms

Four axioms are defined in the registry. Two are constitutional (scope: all implementations). Two are domain-scoped.

| Axiom | Weight | Scope | Summary |
|-------|--------|-------|---------|
| `single_user` | 100 | Constitutional | One operator develops and uses the system. No auth, roles, or multi-user features. |
| `executive_function` | 95 | Constitutional | The system compensates for executive function challenges. Zero-config agents, actionable errors, automated routines, visible state. |
| `management_governance` | 85 | Domain: management | LLMs prepare context; humans deliver words to other humans. Never generate feedback language or coaching recommendations about individuals. |
| `corporate_boundary` | 90 | Domain: infrastructure | Work data stays in employer-controlled systems. Graceful degradation when home-only services are unreachable. |

The axioms produce 68+ derived implications across all tiers. See `axioms/implications/` for the full set.

## Implementations

Two systems implement this pattern. They share architecture, not code — each owns its full stack.

**[hapax-council](https://github.com/ryanklee/hapax-council)** — A personal operating environment. 26+ agents across management, knowledge, sync, voice, and system domains. Always-on voice daemon with ambient perception. RAG pipeline ingesting 7 external sources. Reactive cockpit with FastAPI API and React dashboard. Instantiates all four axioms.

**[hapax-officium](https://github.com/ryanklee/hapax-officium)** — A management decision support system, designed to be forked. 16 agents for 1:1 preparation, team health tracking, management profiling, and briefings. Includes a self-demonstrating capability: bootstrap from synthetic seed data, and the system demos itself — to an audience it profiles — against live operational state. Instantiates three axioms (`single_operator`, `decision_support`, `management_safety`).

## Relationship to prior work

The constitutional pattern was designed in early 2026. Several independent formalizations of agent governance appeared in late 2025 and early 2026:

- **ArbiterOS** (arXiv:2510.13857, October 2025) — governance-first paradigm with non-bypassable "Arbiter Loop"
- **Agent Behavioral Contracts** (arXiv:2602.22302, February 2026) — Design by Contract for agent behavior with drift bound proofs
- **PCAS** (arXiv:2602.16708, February 2026) — policy compiler with reference monitor pattern, improving compliance from 48% to 93%
- **Governance-as-a-Service** (arXiv:2508.18765, August 2025) — modular enforcement with Trust Factor scoring

This pattern arrives at similar structural conclusions (governance separated from the governed, multi-layered enforcement, formal specification) but adds the interpretive canon, sufficiency probes, and personal axioms — mechanisms not present in the research literature.

The 1st Workshop on Operating Systems Design for AI Agents (AgenticOS, co-located with ASPLOS 2026) signals that the systems research community now formally recognizes agent scheduling, state management, and constraint enforcement as OS-level concerns.

## Contents

| Path | Contents |
|------|----------|
| [`pattern-guide.md`](pattern-guide.md) | Full architectural pattern with annotated code examples |
| [`agent-architecture.md`](agent-architecture.md) | Three-tier agent system design |
| [`operations-manual.md`](operations-manual.md) | Operational reference for running implementations |
| [`axioms/registry.yaml`](axioms/registry.yaml) | Axiom definitions (weights, scopes, types) |
| [`axioms/implications/`](axioms/implications/) | Derived implications per axiom (68+ total) |
| [`axioms/precedents/`](axioms/precedents/) | Precedent seeds for common-law interpretation |
| [`knowledge/`](knowledge/) | Sufficiency models (management, music, personal, technical) |
| [`domains/`](domains/) | Domain-specific extensions and life-domain registry |
| [`research/`](research/) | Management theory synthesis (Larson, Team Topologies, Scaling People) |

## License

Apache 2.0 — see [LICENSE](LICENSE).
