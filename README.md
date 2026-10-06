# agent-devkit

Reusable multi-agent development workflow: **planner → implementer →
reviewer** pipelines, driven by detailed specs, with cheap models doing the
coding. This repo is the canonical home of the methodology; each project
holds only the project-specific parts (`specs/`, `docs/`, `board/`).

## The model

- **Human fans out** one pipeline per functional requirement. Agents never
  decide what to build; the planner only scopes and claims.
- **Within a pipeline everything is sequential**: planner (design + task
  work order) → implementer (code + tests) → reviewer (spec compliance).
- **Across pipelines everything is parallel**: one git worktree per
  pipeline (`plan/T-*`, `impl/T-*` branches), file locks guard shared
  files (`specs/`, `docs/`, configs), so pipelines never step on each other.
- **Specs are canonical** (`specs/`, agent-tier, detailed: signatures,
  data shapes, edge cases, acceptance criteria). Human docs (`docs/`) are
  derived, short, and marked non-authoritative.
- **Task files are work orders, not recipes**: spec-section links, files to
  touch (what gets locked), per-change acceptance criteria, tests, status.
  The spec owns the HOW; the task owns the WHERE/WHAT-NOW.
- **The test suite is the real gate.** Reviewers check spec compliance;
  `scripts/verify.py` checks mechanical correctness (lints + tests).

## Session chaining (pipeline triggering)

Each stage ends by spawning the next session, passing:

1. the task file path (`specs/tasks/T-<id>.md`),
2. the branch/worktree for the next stage,
3. the model + effort from `config/model-chains.yaml` (models are set at
   session creation — the chain sets them explicitly),
4. the role file to load (`.agent-devkit/roles/<role>.md`).

There is no central orchestrator; the chain is stateless and breaks safely.
If a session dies, re-spawn the next link manually from the task file.

## Issue channel

Implementers and reviewers cannot edit specs. Ambiguity goes to
`board/issues/` (one file per issue, from `templates/issue.md`) **plus** a
message to the originating planner session (its URI is recorded in the
task file). The planner session is resumed, not replaced — it holds the
design context, and resuming prevents the spec from forking. The planner
resolves the issue *in the spec*, then messages the implementer to resume.
The issue file is the durable record; the message is just the doorbell.

## Lock board

- `python scripts/claim.py --role <role> --task T-<id> <paths...>`
  claims before editing (sorted order, atomic `mkdir`, idempotent
  re-claim by same task). Blocks while a file is claimed elsewhere.
- `python scripts/release.py --task T-<id>` releases (always, including
  failure paths).
- Locks older than **30 minutes** are auto-reclaimed on the next claim
  attempt and logged to `board/lock-events.log`.
- `board/BOARD.md` is a generated view of active claims.

## Permission enforcement (deterministic, not prompt-based)

Branch naming is the role signal, enforced by `scripts/path_guard.py` and
the `agent-gates` CI workflow on every push/PR:

| Branch | May change |
|---|---|
| `plan/T-*` | `specs/`, `docs/`, `board/`, `.github/copilot-instructions.md` |
| `impl/T-*` | everything except `specs/` and `docs/` |
| `review/T-*` | `board/` only |

## Roles

See `.agent-devkit/roles/` after install, or `roles/` here. Model/effort
config: `config/model-chains.yaml`. Current default stack:

| Role | Model | Effort |
|---|---|---|
| planner | `xiaomi/mimo-v2.6-pro` | high |
| implementer | `z-ai/glm-5.3-flash` | medium |
| reviewer | `deepseek/deepseek-v4.1-flash` | low |

## Install into a project

```powershell
python scripts/init.py --target /path/to/project
```

Copies scripts + CI workflow + roles/templates/config, scaffolds
`board/` and `docs/`. Re-run safe. Pin the devkit commit in
`devkit.version` per project.

## Design decisions (rationale lives with the decision)

- Session-chaining over a central orchestrator: stateless, breaks safely.
- Worktrees for implementers, locks for shared text: git isolates code;
  locks are what `specs/`/`docs/` edits need.
- Task files kept slim: detailed specs make recipes redundant; the claim
  list and per-change acceptance criteria are what machinery needs.
- Human-initiated requirement fan-out: decomposition at the requirement
  level is the human's job (intent); agents decompose only into steps
  inside a work order.
