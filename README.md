# hapax-constitution

A governance architecture for LLM agent systems, specified as a pattern with two reference implementations.

## Background

LLM agents that manage personal infrastructure — preparing meeting context, maintaining knowledge bases, monitoring system health, reacting to filesystem changes — accumulate autonomy as they are granted access to more tools and data. Prompt-level governance instructions degrade under context pressure: long conversations, complex tool chains, and multi-step reasoning reduce instruction-following reliability. Violations are discovered after the fact, in produced output.

Adding more rules does not address the structural issue. Rules cover anticipated cases. An agent may introduce session management through a dependency or implement user-switching through a configuration mechanism that does not use the word "auth." Rules do not cover cases the author did not anticipate.

This project specifies a governance architecture that borrows its structure from constitutional law. Constitutions, statutes, case law, interpretive canons, and affirmative obligations compose into a system that handles unanticipated cases through principled reasoning rather than exhaustive enumeration.

## Constitutional structure

A constitution differs from a rulebook in that it establishes supreme principles against which all downstream decisions must be consistent, rather than enumerating permitted and prohibited actions. The architecture has four interlocking mechanisms.

### Axioms and implications

Axioms are weighted constraints defined in YAML. Each axiom produces derived implications at graduated enforcement tiers. Implications carry metadata about how to interpret them in edge cases.

Four axioms govern the current implementations:

| Axiom | Weight | Core constraint |
|-------|--------|----------------|
| `single_user` | 100 | One operator develops and uses the system. No auth, no roles, no multi-user features. |
| `executive_function` | 95 | The system compensates for executive function challenges (ADHD, autism). Zero-config agents, actionable errors, automated routines, visible state. |
| `corporate_boundary` | 90 | Work data stays in employer systems. Graceful degradation when crossing network boundaries. |
| `management_governance` | 85 | LLMs prepare context; humans deliver words to other humans. No generated feedback language or coaching recommendations about individuals. |

Each axiom produces derived implications — currently 90 across all tiers. An implication is a specific, enforceable constraint derived from a general axiom. For example, the axiom `executive_function` states that the system should reduce cognitive load; the implication `ex-err-001` states that error messages must include a concrete next action — a command to run, a file to check, a service to restart. Implications are enforced at four tiers:

- **T0 (Block):** Code matching this pattern must not exist in the repository. Commit hooks prevent it from landing.
- **T1 (Review):** Significant constraint. Requires operator awareness before proceeding.
- **T2 (Warn):** Quality preference. Should be followed absent specific reason not to.
- **T3 (Lint):** Style and convention preferences. Advisory only.

The tier graduation follows the same logic as levels of scrutiny in constitutional law — strict scrutiny for fundamental rights, rational basis for ordinary legislation. Existential violations receive existential enforcement; preferences receive advisory enforcement.

### Interpretive canons

Formal constraint systems share a common limitation with legal codes: they cannot anticipate every case. Legal systems address this with canons of construction — principled reasoning methods for applying statutes to unanticipated situations. This architecture borrows four:

- **Textualist:** Apply the literal text. If the implication says "no authentication code," there is no authentication code.
- **Purposivist:** Apply the axiom's intent. If a library introduces user-switching as a side effect, the purposivist canon identifies the violation even when the textualist canon does not — there is no "auth code" per se, but the axiom's purpose is violated.
- **Absurdity:** Reject a literal application that produces an absurd result. The `single_user` axiom prohibits multi-user features, but a textualist reading that prohibits connecting to multi-user services (GitHub, Google Calendar) would prevent the system from functioning.
- **Omitted-case:** The axioms are silent on this situation. Apply the nearest precedent, or escalate to the operator for a ruling.

Each implication carries a canon classification, so the enforcement layer knows how to interpret it, not just what it says.

### Precedent store (stare decisis)

*Stare decisis* — "to stand by things decided" — is the legal principle that courts should follow prior rulings. Consistent interpretation has value independent of whether any individual ruling is optimal. Without precedent, each evaluation of an edge case starts from first principles, producing inconsistent results.

When an axiom implication encounters an edge case — for example, whether Tailscale multi-device access violates `single_user` (it does not: one operator, multiple devices, no user-switching) — the operator makes a ruling. That ruling is recorded as a precedent: the situation, the reasoning, the resolution, and the scope. Future evaluations query the precedent store via semantic search in Qdrant before escalating to the operator.

The axioms form the constitutional layer. The implications form the statutory layer. The precedents form the case law layer. Each layer handles what the layer above cannot specify in advance. Precedents carry authority weights (operator rulings outrank agent-generated ones), can be promoted or superseded with recorded reasoning, and are append-only — superseded precedents are retained because the reasoning behind old decisions may be relevant to future ones.

### Sufficiency probes

Most governance systems are prohibition-only: they verify that the system does not do prohibited things. A system that never generates feedback language but also never includes next actions in error messages passes a prohibition-only audit while failing to meet its positive obligations.

Sufficiency probes check for the absence of required things. The `executive_function` axiom does not just prohibit complex configuration — it requires that agents work with zero configuration. It does not just prohibit vague errors — it requires that every error message contains something actionable.

Each implication carries a mode: `prohibition` (the system must not do this) or `sufficiency` (the system must do this). The enforcement engine runs both. As of early 2026, ArbiterOS, Agent Behavioral Contracts, PCAS, and Governance-as-a-Service all implement prohibition checking; none implement sufficiency checking.

## Structural patterns

Beyond axiom governance, the architecture specifies two structural patterns.

### Filesystem-as-bus

All coordination state lives as markdown files with YAML frontmatter on disk. Directories are collections. Agents read files to gather context and write files to produce output. There is no message broker, no shared database, no RPC.

This trades ordered delivery and transactional consistency for: human-readable state (open any file in a text editor), git-native history (every state transition is a diff), tool-agnostic interoperability (any language reads markdown), zero-infrastructure coordination (no broker to run or monitor), and graceful degradation (if the engine is down, the data is still accessible).

At single-operator scale with sequential or low-concurrency agent execution, the consistency trade-off has no practical cost. The pattern is independently described by two recent formalizations: "From Everything-is-a-File to Files-Are-All-You-Need" (arXiv:2601.11672, January 2026) and "Everything is Context" (arXiv:2512.05470, December 2025). Unlike the virtual filesystem abstractions those papers propose, this pattern uses literal files on disk — debuggable with `cat`, `grep`, `diff`, and `git log`.

### Three-tier agent architecture

- **Tier 1 (Interactive):** Claude Code with MCP tools, logos dashboard, web dashboard. Full operator supervision.
- **Tier 2 (On-demand):** Pydantic AI agents invoked by CLI, API, or Tier 1. Stateless per-invocation; all persistent state lives on the filesystem or in vector storage.
- **Tier 3 (Autonomous):** systemd timers running Tier 2 agents on schedules. High-frequency agents (health monitoring, knowledge maintenance) are deterministic — zero LLM calls.

Agents do not invoke other agents. There is no orchestrator, DAG, or workflow engine. Agents communicate through filesystem artifacts, which decouples them temporally and makes the coordination graph auditable with standard Unix tools.

### Reactive engine

An inotify watcher monitors the filesystem bus. File changes produce enriched events (including document type from YAML frontmatter). Rules — pure functions mapping events to actions — evaluate against each change. Multiple rules can fire; duplicate actions collapse. Actions execute in phases: deterministic work first (unlimited concurrency, zero cost), then LLM work (semaphore-bounded to prevent GPU saturation or API cost runaway). The engine tracks its own writes and skips events from them, preventing infinite self-trigger loops.

## Implementations

Two systems implement this pattern. They share architecture and infrastructure (Qdrant, LiteLLM, Ollama, PostgreSQL) but not code — each owns its full stack. The constitution constrains both; the implementations evolve independently.

**[hapax-council](https://github.com/ryanklee/hapax-council)** — Personal operating environment. 45+ agents across management, knowledge, sync, voice, perception, studio, governance, and system domains. Always-on voice daemon with a multi-cadence perception engine fusing audio, visual, biometric, and environmental signals through a typed FRP pipeline. Temporal intelligence stack (Husserlian retention/impression/protention, SystemStimmung self-regulation, predictive content scheduling). GPU-accelerated studio compositor with visual effects and visual layer overlays. RAG pipeline ingesting 10 external sources. Logos API with React dashboard. Instantiates all five axioms.

**[hapax-officium](https://github.com/ryanklee/hapax-officium)** — Management-domain extraction, designed to be forked by other engineering managers. 17 agents for 1:1 preparation, team health tracking, management profiling, temporal simulation, and briefings. Includes a self-demonstrating capability: bootstrap from synthetic seed data, and the system generates a demonstration against live operational state. Originally part of council, extracted when the management agents proved independently usable. Instantiates three axioms (`single_operator`, `decision_support`, `management_safety` — renamed from the canonical IDs to fit the management domain vocabulary).

## Relationship to prior work

The constitutional pattern was designed in early 2026. Several independent formalizations of agent governance appeared in late 2025 and early 2026:

- **ArbiterOS** (arXiv:2510.13857) — governance-first paradigm with a non-bypassable "Arbiter Loop"
- **Agent Behavioral Contracts** (arXiv:2602.22302) — Design by Contract for agent behavior with drift bound proofs
- **PCAS** (arXiv:2602.16708) — policy compiler with reference monitor pattern, improving compliance from 48% to 93%
- **Governance-as-a-Service** (arXiv:2508.18765) — modular enforcement with Trust Factor scoring

This pattern arrives at similar structural conclusions — governance separated from the governed, multi-layered enforcement, formal specification — and adds three mechanisms not present in the literature: interpretive canons (principled reasoning about unanticipated cases), sufficiency probes (affirmative obligations, not only prohibitions), and personal axioms (governance tailored to a specific operator's cognitive constraints rather than generic safety rails). The seven-layer enforcement architecture converges across constitutional law, AI safety, policy-as-code (OPA, Cedar), and architecture fitness functions.

The 1st Workshop on Operating Systems Design for AI Agents (AgenticOS, co-located with ASPLOS 2026) indicates that the systems research community now formally recognizes agent scheduling, state management, and constraint enforcement as OS-level concerns.

## Contents

| Path | Contents |
|------|----------|
| [`pattern-guide.md`](pattern-guide.md) | Full architectural pattern with annotated code examples |
| [`agent-architecture.md`](agent-architecture.md) | Three-tier agent system design |
| [`operations-manual.md`](operations-manual.md) | Operational reference for running implementations |
| [`axioms/registry.yaml`](axioms/registry.yaml) | Axiom definitions (weights, scopes, types) |
| [`axioms/implications/`](axioms/implications/) | Derived implications per axiom (90 total) |
| [`axioms/precedents/`](axioms/precedents/) | Precedent seeds for common-law interpretation |
| [`knowledge/`](knowledge/) | Sufficiency models (management, music, personal, technical) |
| [`domains/`](domains/) | Domain-specific extensions and life-domain registry |
| [`research/`](research/) | Research foundations (management theory, axiom enforcement, prior art) |

## Part of the Hapax Research Project

This governance specification defines the axiom framework used by a research project implementing Clark & Brennan's (1991) conversational grounding theory in a voice AI system. See [hapax-council](https://github.com/ryanklee/hapax-council) for the research context and SCED experiment design.

| Repository | Role |
|-----------|------|
| [hapax-council](https://github.com/ryanklee/hapax-council) | Primary research artifact — voice daemon, grounding system, experiment infrastructure |
| **hapax-constitution** (this repo) | Governance specification — axioms, implications, canons |
| [hapax-officium](https://github.com/ryanklee/hapax-officium) | Supporting software — management decision support |
| [hapax-watch](https://github.com/ryanklee/hapax-watch) | Research instrument — Wear OS biometric companion |
| [hapax-mcp](https://github.com/ryanklee/hapax-mcp) | Infrastructure — MCP server for Claude Code |

## Citation

If you use this software in your research, please cite it using the [CITATION.cff](CITATION.cff) file.

## License

Apache 2.0 — see [LICENSE](LICENSE).
