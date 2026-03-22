# hapax-constitution

Governance specification for a research project implementing Clark & Brennan's (1991) conversational grounding theory in a production voice AI system, evaluated via Single Case Experimental Design (SCED) with Bayesian analysis. See [hapax-council](https://github.com/ryanklee/hapax-council) for the primary research artifact and experiment design.

## Role in the research project

The research apparatus comprises 45+ LLM agents operating across perception, voice, knowledge, and system domains. These agents accumulate autonomy as they access tools, data, and external services. Prompt-level governance degrades under context pressure: long conversations, complex tool chains, and multi-step reasoning reduce instruction-following reliability. Violations surface after the fact, in produced output.

This repository specifies the axiom governance framework that constrains both runtime implementations ([hapax-council](https://github.com/ryanklee/hapax-council), [hapax-officium](https://github.com/ryanklee/hapax-officium)). The published `hapax-sdlc` package provides enforcement tooling consumed by both systems.

## Governance architecture

The architecture borrows its structure from constitutional law. Four mechanisms compose into a system that handles unanticipated cases through principled reasoning rather than exhaustive enumeration.

### Axioms and implications

Axioms are weighted constraints defined in YAML. Each axiom produces derived implications at graduated enforcement tiers.

| Axiom | Weight | Core constraint |
|-------|--------|----------------|
| `single_user` | 100 | One operator. No auth, no roles, no multi-user features. |
| `executive_function` | 95 | Compensates for executive function challenges (ADHD, autism). Zero-config agents, actionable errors, automated routines. |
| `corporate_boundary` | 90 | Work data stays in employer systems. Graceful degradation across network boundaries. |
| `management_governance` | 85 | LLMs prepare context; humans deliver words. No generated feedback language or coaching recommendations about individuals. |

90 implications across four enforcement tiers: T0 (block at commit), T1 (require review), T2 (warn), T3 (lint). Tier graduation follows the logic of levels of scrutiny in constitutional law.

### Interpretive canons

Four canons handle unanticipated cases: textualist (literal text), purposivist (axiom intent), absurdity (reject literal applications producing absurd results), omitted-case (apply nearest precedent or escalate).

### Precedent store

Operator rulings on edge cases are recorded with situation, reasoning, resolution, and scope. Future evaluations query semantically via Qdrant before escalating. Precedents carry authority weights and are append-only with supersession tracking.

### Sufficiency probes

Implications carry a mode: prohibition (must not do) or sufficiency (must do). The enforcement engine runs both. The `executive_function` axiom requires that every error message contains a concrete next action; a system that never generates feedback language but also never includes next actions passes prohibition checks while failing sufficiency.

## Structural patterns

Two patterns specified alongside axiom governance:

**Filesystem-as-bus.** Coordination state as markdown files with YAML frontmatter on disk. No message broker, database, or RPC. Trades transactional consistency for human-readable state, git-native history, and tool-agnostic interoperability. At single-operator scale, the consistency trade-off has no practical cost. Independently described by "From Everything-is-a-File to Files-Are-All-You-Need" (arXiv:2601.11672) and "Everything is Context" (arXiv:2512.05470).

**Three-tier agent architecture.** Tier 1 (interactive: Claude Code, dashboards), Tier 2 (on-demand: pydantic-ai agents), Tier 3 (autonomous: systemd timers). Agents communicate through filesystem artifacts; no orchestrator or workflow engine.

## Related work

Several independent formalizations of agent governance appeared in late 2025 and early 2026: ArbiterOS (arXiv:2510.13857), Agent Behavioral Contracts (arXiv:2602.22302), PCAS (arXiv:2602.16708), Governance-as-a-Service (arXiv:2508.18765). This architecture converges on similar structural conclusions and adds interpretive canons, sufficiency probes, and personal axioms (governance tailored to a specific operator's cognitive constraints).

## Contents

| Path | Contents |
|------|----------|
| [`axioms/registry.yaml`](axioms/registry.yaml) | Axiom definitions (weights, scopes, types) |
| [`axioms/implications/`](axioms/implications/) | Derived implications per axiom (90 total) |
| [`axioms/precedents/`](axioms/precedents/) | Precedent seeds |
| [`knowledge/`](knowledge/) | Sufficiency models (management, music, personal, technical) |
| [`research/`](research/) | Governance research (axiom evaluation, gap analysis, landscape analysis, management theory) |
| [`pattern-guide.md`](pattern-guide.md) | Full architectural pattern with code examples |
| [`agent-architecture.md`](agent-architecture.md) | Three-tier agent system design |
| [`sdlc/`](sdlc/) | SDLC enforcement tools (published as `hapax-sdlc` package) |

## Ecosystem

| Repository | Role |
|-----------|------|
| [hapax-council](https://github.com/ryanklee/hapax-council) | Primary research artifact — voice daemon, grounding system, experiment infrastructure |
| **hapax-constitution** (this repo) | Governance specification — axioms, implications, canons, precedents |
| [hapax-officium](https://github.com/ryanklee/hapax-officium) | Supporting software — management decision support |
| [hapax-watch](https://github.com/ryanklee/hapax-watch) | Research instrument — Wear OS biometric companion |
| [hapax-mcp](https://github.com/ryanklee/hapax-mcp) | Infrastructure — MCP server for Claude Code |

## Citation

If you use this specification in your research, please cite it using the [CITATION.cff](CITATION.cff) file.

## License

Apache 2.0 — see [LICENSE](LICENSE).
