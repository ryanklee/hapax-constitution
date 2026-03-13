# hapax-constitution

A governance architecture for LLM agent systems, specified as a pattern with two reference implementations.

## The problem with governing agents

LLM agents that manage personal infrastructure — preparing meeting context, maintaining knowledge bases, monitoring system health, reacting to filesystem changes — accumulate autonomy without accumulating constraints. The standard approach is to put governance instructions in the prompt: "never generate feedback about individuals," "always include a next action in error messages," "do not store data about people without consent." This works until it doesn't.

Prompt-level instructions degrade under context pressure. Long conversations, complex tool chains, and multi-step reasoning erode instruction-following. The instructions cannot be audited independently of the model's behavior. They provide no mechanism for handling situations the author didn't anticipate. And they fail silently — you discover the violation after it has already happened, in production, when someone reads the output.

The natural response is to add more rules. But rules share the same fundamental limitation as prompts: they cover anticipated cases. A rule that says "no authentication code" handles the cases you thought of when you wrote it. It doesn't handle the case where an agent introduces session management through a dependency, or implements user-switching through a configuration mechanism that doesn't use the word "auth." Rules fail silently on the unanticipated case, which is exactly the case that matters.

This project specifies a governance architecture that borrows its structure from constitutional law. The key insight is that legal systems solved this problem centuries ago — not perfectly, but durably. Constitutions, statutes, case law, judicial reasoning canons, and affirmative obligations compose into a system that handles cases its authors never imagined. The question is whether that structure transfers to software agents.

## Why constitutional law

A constitution is not a rulebook. A rulebook says "do this, don't do that." A constitution says "these principles are supreme, and everything that follows must be consistent with them." The difference matters because a rulebook must anticipate every situation (and cannot), while a constitution provides principled reasoning methods for situations nobody anticipated.

The architecture has four interlocking mechanisms.

### Axioms and implications

Axioms are weighted constraints defined in YAML. They look like rules, but they function differently — each axiom produces *derived implications* at graduated enforcement tiers, and those implications carry metadata about how to interpret them in edge cases.

Four axioms govern the current implementations:

| Axiom | Weight | Core constraint |
|-------|--------|----------------|
| `single_user` | 100 | One operator develops and uses the system. No auth, no roles, no multi-user features anywhere. |
| `executive_function` | 95 | The system compensates for executive function challenges (ADHD, autism). Zero-config agents, actionable errors, automated routines, visible state. |
| `corporate_boundary` | 90 | Work data stays in employer systems. Graceful degradation when crossing network boundaries. |
| `management_governance` | 85 | LLMs prepare context; humans deliver words to other humans. No generated feedback language or coaching recommendations about individuals. |

Each axiom produces derived implications — currently 68+ across all tiers. An implication is a specific, enforceable constraint derived from the general axiom. The axiom `executive_function` says the system should reduce cognitive load; the implication `ex-err-001` says error messages must include a concrete next action — a command to run, a file to check, a service to restart. Implications are enforced at three tiers:

- **T0 (Block):** Code matching this pattern must not exist in the repository. Commit hooks prevent it from landing.
- **T1 (Review):** Significant constraint. Requires operator awareness before proceeding.
- **T2 (Warn):** Quality preference. Should be followed absent specific reason not to.

The graduation matters. Not every implication deserves to block a commit. Legal systems solved this with *levels of scrutiny* — strict scrutiny for fundamental rights, rational basis for ordinary legislation. The tier system applies the same logic: existential violations get existential enforcement, preferences get advisory enforcement, and the gap between them is principled rather than arbitrary.

### Interpretive canons

Here is the problem that every formal constraint system shares with every legal code: you cannot anticipate every case. A city ordinance says "no vehicles in the park." Does that prohibit an ambulance responding to a medical emergency? A child's bicycle? A monument of a decommissioned tank? The words are clear, but the application is not.

Legal systems handle this with *canons of construction* — principled reasoning methods that judges use to apply statutes to cases the legislature never considered. This architecture borrows four:

- **Textualist:** Apply the literal text. If the implication says "no authentication code," there is no authentication code. This is the default and handles most cases.
- **Purposivist:** Apply the axiom's intent. The `single_user` axiom exists because multi-user infrastructure adds complexity that a single-operator system doesn't need. If a library introduces user-switching as a side effect, the purposivist canon catches it even though the textualist canon might not — there's no "auth code" per se, but the purpose is violated.
- **Absurdity:** Reject a literal application that produces an absurd result. The `single_user` axiom prohibits multi-user features, but a textualist reading that prohibits the system from *connecting to multi-user services* (like GitHub or Google Calendar) would prevent the system from functioning. The absurdity canon prevents this interpretation.
- **Omitted-case:** The axioms are silent on this situation. Apply the nearest precedent, or escalate to the operator for a ruling.

Each implication carries a canon classification, so the enforcement layer knows *how* to interpret it, not just *what* it says. This is what separates a governance architecture from a linter: a linter can check patterns, but it cannot reason about intent, purpose, or absurdity.

### Precedent store (stare decisis)

*Stare decisis* — "to stand by things decided" — is the legal principle that courts should follow their own prior rulings. It exists because consistent interpretation matters more than perfect interpretation. A legal system where each judge starts from first principles on every case produces unpredictable, contradictory results even when the underlying statute is clear.

The same problem arises in agent governance. When an axiom implication encounters an edge case — say, whether Tailscale multi-device access violates `single_user` (it doesn't: one operator, multiple devices, no user-switching) — the operator makes a ruling. That ruling is recorded as a precedent: the situation, the reasoning, the resolution, and the scope. Future evaluations query the precedent store (via semantic search over embeddings in Qdrant) before escalating to the operator.

This builds a common-law layer over the constitutional layer. The axioms are the constitution. The implications are statutes. The precedents are case law. Each layer handles what the layer above cannot specify in advance. Over time, the precedent corpus makes axiom interpretation consistent and reduces operator interrupts — the system develops institutional memory about how to apply its own principles.

Precedents carry authority weights (operator rulings outrank agent-generated ones), can be promoted (widened in scope) or superseded (overruled with recorded reasoning), and accumulate into a searchable body of institutional knowledge. The mechanism is append-only: superseded precedents are never deleted, because the reasoning behind old decisions may matter for future ones.

### Sufficiency probes

Most governance systems are prohibition-only: they check that the system does NOT do bad things. This is necessary but incomplete. A system that never generates feedback language (good) but also never includes next actions in error messages (bad) passes a prohibition-only audit while failing the operator.

Sufficiency probes invert the check. Instead of scanning for the presence of bad things, they scan for the *absence of required things*. The `executive_function` axiom doesn't just prohibit complex configuration — it *requires* that agents work with zero configuration. It doesn't just prohibit vague errors — it *requires* that every error message contains something actionable.

Each implication carries a mode: `prohibition` (the system must NOT do this) or `sufficiency` (the system MUST do this). The enforcement engine runs both. This inversion — checking for what must exist, not just what must not exist — is absent from the agent governance literature as of early 2026. ArbiterOS, Agent Behavioral Contracts, PCAS, and Governance-as-a-Service all implement prohibition checking. None implement sufficiency checking.

The distinction matters most for the `executive_function` axiom, where the operator's cognitive constraints mean that passive compliance (not making things worse) is insufficient — the system must actively reduce cognitive load. A prohibition-only system cannot verify this.

## The pattern

Beyond axiom governance, the architecture specifies two structural patterns.

### Filesystem-as-bus

All coordination state lives as markdown files with YAML frontmatter on disk. Directories are collections. Agents read files to gather context and write files to produce output. There is no message broker, no shared database, no RPC.

This is a deliberate trade-off. Message queues provide ordered delivery and transactional consistency. Filesystem-as-bus provides: human-readable state (open any file in a text editor), git-native history (every state transition is a diff), tool-agnostic interoperability (any language reads markdown), zero-infrastructure coordination (no broker to run or monitor), and graceful degradation (if the engine is down, the data is still there and can be edited manually).

At single-operator scale with sequential or low-concurrency agent execution, the consistency trade-off has no practical cost. The pattern is independently validated by two recent formalizations: "From Everything-is-a-File to Files-Are-All-You-Need" (arXiv:2601.11672, January 2026) and "Everything is Context" (arXiv:2512.05470, December 2025). Unlike the virtual filesystem abstractions those papers propose, this pattern uses literal files on disk — debuggable with `cat`, `grep`, `diff`, and `git log`.

### Three-tier agent architecture

- **Tier 1 (Interactive):** Claude Code with MCP tools, system cockpit, web dashboard. Full operator supervision.
- **Tier 2 (On-demand):** Pydantic AI agents invoked by CLI, API, or Tier 1. Stateless per-invocation; all persistent state lives on the filesystem or in vector storage.
- **Tier 3 (Autonomous):** systemd timers running Tier 2 agents on schedules. High-frequency agents (health monitoring, knowledge maintenance) are deterministic — zero LLM calls.

Agents never invoke other agents. There is no orchestrator, no DAG, no workflow engine. Orchestration is flat: agents communicate through filesystem artifacts, which decouples them temporally and makes the coordination graph auditable with standard Unix tools. `ls -lt` shows what ran. `git log` shows what changed. `diff` shows what was produced.

### Reactive engine

An inotify watcher monitors the filesystem bus. File changes produce enriched events (including document type from YAML frontmatter). Rules — pure functions mapping events to actions — evaluate against each change. Multiple rules can fire; duplicate actions collapse. Actions execute in phases: deterministic work first (unlimited concurrency, zero cost), then LLM work (semaphore-bounded to prevent GPU saturation or API cost runaway). The engine tracks its own writes and skips events from them, preventing infinite self-trigger loops.

## Implementations

Two systems implement this pattern. They share architecture and infrastructure (Qdrant, LiteLLM, Ollama, PostgreSQL) but not code — each owns its full stack. The constitution constrains both; the implementations evolve independently.

**[hapax-council](https://github.com/ryanklee/hapax-council)** — Personal operating environment. 26+ agents across management, knowledge, sync, voice, and system domains. Always-on voice daemon with a multi-cadence perception engine fusing audio, visual, and environmental signals through a typed FRP pipeline. RAG pipeline ingesting 7 external sources. Reactive cockpit with FastAPI API and React dashboard. Instantiates all four axioms.

**[hapax-officium](https://github.com/ryanklee/hapax-officium)** — Management-domain extraction, designed to be forked by other engineering managers. 16 agents for 1:1 preparation, team health tracking, management profiling, and briefings. Includes a self-demonstrating capability: bootstrap from synthetic seed data, and the system generates a demonstration — tailored to a profiled audience — against live operational state. Officium was originally part of council and was extracted when the management agents proved independently usable. Instantiates three axioms (`single_operator`, `decision_support`, `management_safety` — renamed from the canonical IDs to fit the management domain vocabulary).

## Relationship to prior work

The constitutional pattern was designed in early 2026. Several independent formalizations of agent governance appeared in late 2025 and early 2026:

- **ArbiterOS** (arXiv:2510.13857) — governance-first paradigm with a non-bypassable "Arbiter Loop"
- **Agent Behavioral Contracts** (arXiv:2602.22302) — Design by Contract for agent behavior with drift bound proofs
- **PCAS** (arXiv:2602.16708) — policy compiler with reference monitor pattern, improving compliance from 48% to 93%
- **Governance-as-a-Service** (arXiv:2508.18765) — modular enforcement with Trust Factor scoring

This pattern arrives at similar structural conclusions — governance separated from the governed, multi-layered enforcement, formal specification — but adds three mechanisms absent from the literature: the interpretive canon (principled reasoning about unanticipated cases), sufficiency probes (affirmative obligations, not only prohibitions), and personal axioms (governance tailored to a specific operator's cognitive constraints, not generic safety rails). The seven-layer enforcement architecture converges across constitutional law, AI safety, policy-as-code (OPA, Cedar), and architecture fitness functions, suggesting these layers are structurally necessary rather than design choices.

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
| [`research/`](research/) | Research foundations (management theory, axiom enforcement, prior art) |

## License

Apache 2.0 — see [LICENSE](LICENSE).
