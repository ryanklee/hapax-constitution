# CLAUDE.md

Governance architecture for LLM agent systems — constitutional axioms, derived implications, interpretive canon, sufficiency probes, and precedent store. **Specification repo**: YAML definitions and markdown documentation, not application code.

Shared conventions (uv, ruff, testing, git workflow) are in the workspace `CLAUDE.md` — this file covers constitution-specific details only.

## Key Files

- **`axioms/registry.yaml`** — Canonical axiom definitions (IDs, weights, scope, text)
- **`axioms/implications/`** — Per-axiom derived implications with tier (T0/T1/T2), enforcement mode, sufficiency levels
- **`domains/`** — Domain-scoped axiom extensions (infrastructure, management)
- **`knowledge/`** — Sufficiency probes per domain (management, music, personal, technical)
- **`pattern-guide.md`** — Guide for implementing the governance pattern

## Conventions

- **NEVER switch branches in the primary worktree.** The primary checkout (`~/projects/hapax-constitution`) stays on `main`. For any feature branch, use `git worktree add ../hapax-constitution--<branch-slug> <branch>`. This prevents concurrent Claude sessions from clobbering each other's state. When done, `git worktree remove`.
- **Always PR completed work before moving on.** When you finish a coherent batch of work (feature, fix, refactor), create a PR immediately — do not wait to be asked. Only skip the PR if the work is genuinely incomplete or broken. Push and PR freely; this is expected behavior. **Do NOT start new work until the current work is resolved** — resolved means either a PR has been submitted or there is no branch/changes remaining to PR. This is a blocking requirement.
- **You own every PR you create through to merge.** Do not abandon PRs. Monitor CI checks, fix failures, update the branch if behind, and merge when ready. A PR is not done until it is merged into main. If checks fail, diagnose and fix them before moving on. If the branch falls behind, update it. This is your responsibility — no one else will do it.
- **This is a spec repo.** Changes to `axioms/registry.yaml` are always high-complexity and require human review.
- **YAML is the source of truth** for axiom definitions and implications. Markdown documents describe; YAML defines.
- **Weight ordering matters.** Higher weight = higher precedence. Constitutional axioms always outweigh domain axioms (supremacy clause).
- **Tier semantics are strict.** T0 = block (existential violation), T1 = review (requires awareness), T2 = warn (quality preference), T3 = lint (advisory style/documentation guidance, enforcement: linter).

## Local Validation

```bash
uv sync
uv run ruff check .              # Lint Python (SDLC scripts)
uv run ruff format --check .     # Format check
yamllint axioms/ domains/ knowledge/  # YAML lint (if yamllint installed)
```

CI runs YAML linting automatically via `yaml-lint.yml` workflow.

## SDLC Pipeline

Spec-focused: no implement workflow. Triage → Review → Axiom Gate (structural only) → Auto-merge. Changes touching `axioms/registry.yaml` always classified as L complexity. Scripts in `scripts/`, workflows in `.github/workflows/`.
