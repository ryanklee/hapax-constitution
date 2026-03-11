# hapaxromana

Architectural specification and coordination space for a three-tier autonomous agent system. Contains design documents, axiom governance, domain lattice definitions, and sufficiency models. No runnable code — agent implementations live in separate repos.

## Contents

```
axioms/               Constitutional and domain axiom definitions
  registry.yaml       4 axioms with scope, weight, and type
  implications/       Per-axiom derived implications (72 total, 16 T0 blocks)
  precedents/seed/    Seed precedent definitions for Qdrant
domains/
  registry.yaml       4 life domains with relationships and person extensions
knowledge/
  management-sufficiency.yaml   27 requirements across 3 tiers
  music-sufficiency.yaml        6 requirements (stub)
  personal-sufficiency.yaml     4 requirements (stub)
  technical-sufficiency.yaml    5 requirements (stub)
research/             Management theory deep dives (Larson, Team Topologies, Scaling People)
docs/
  plans/              Design docs and implementation plans (dated)
  audit/              System audit reports (v1 and v2)
agent-architecture.md 13 implemented agents, 3 planned
operations-manual.md  First day / week / month operational guide
```

## Related Repos

| Repo | Purpose |
|------|---------|
| [ai-agents](~/projects/ai-agents/) | Pydantic AI agents (Tier 2) + cockpit API backend |
| [cockpit-web](~/projects/cockpit-web/) | React SPA web dashboard (cockpit frontend) |
| [hapax-vscode](~/projects/hapax-vscode/) | VS Code extension — chat sidebar, RAG search, management commands |
| [rag-pipeline](~/projects/rag-pipeline/) | Docling RAG ingestion (Tier 3) |
| [llm-stack](~/llm-stack/) | Docker Compose infrastructure |
| [hapax-system](~/projects/hapax-system/) | Claude Code skills, agents, rules, hooks |

## Axiom Governance

| Axiom | Weight | Scope | Core Principle |
|-------|--------|-------|---------------|
| single_user | 100 | constitutional | No auth, no multi-user, no collaboration |
| executive_function | 95 | constitutional | Zero-config, automated routines, actionable errors |
| corporate_boundary | 90 | domain | Plugin works across Zscaler boundary, graceful degradation |
| management_governance | 85 | domain | LLMs prepare, humans deliver — never generate feedback language |

## Document Authority

1. Source code (ground truth)
2. CLAUDE.md (canonical reference)
3. operations-manual.md (operational how-to)
4. agent-architecture.md (design reference)
5. README.md (this file — summary)
