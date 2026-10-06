# agent-devkit

Reusable multi-agent development workflow: **planner → implementer →
reviewer** pipelines, driven by detailed specs, with cheap models doing the
coding. This repo is the canonical home of the methodology; each project
holds only the project-specific parts (`specs/`, `docs/`, `board/`).

**New here?** Read in this order: *The model* → *Bootstrap a new project*
→ *Wiring per harness* → then the mechanics sections (chaining, issues,
locks, enforcement). Agent role files are in `roles/`, ready-made
templates in `templates/` (including `handoff.md`, the chaining contract).

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

## Bootstrap a new project (agent checklist)

An agent setting this up in a fresh codebase, in order:

1. `python scripts/init.py --target <project>` — installs scripts, CI
   workflow, `.agent-devkit/` (roles, templates, config), scaffolds
   `board/`, `docs/`, `specs/tasks/`.
2. Adopt the project instruction file from
   `templates/copilot-instructions.md` — copy into whichever file the
   harness reads (`.github/copilot-instructions.md`, `CLAUDE.md`, or
   `AGENTS.md`), adapting the project name.
3. Seed `docs/overview.md` + `docs/changelog.md` from the templates (init
   does this if they are missing) and fill in intent.
4. Create the spec skeleton: numbered flat files in `specs/` (system map,
   decisions log, milestones log, one file per component), using the
   status markers and one-home-per-register rules from the instruction
   file template. If specs already exist, keep their layout and add only
   `specs/tasks/`.
5. Record the devkit pin: `devkit.version` = the devkit commit hash.
6. Commit. Then run pipelines per "The model" above.

## Wiring per harness

- **Copilot SDK (VS Code):** spawn each stage as a session whose prompt =
  the handoff (from `templates/handoff.md`) + the role file text; pass the
  role's model at session creation (`create_session`'s model parameter —
  the chain sets it explicitly). Message the planner session URI
  (`send_message`) for issues/approvals — that is the "resume" mechanism.
- **Claude Code:** role files double as `.claude/agents/*.md` definitions;
  set the model in each definition; `CLAUDE.md` = the instruction template;
  a `PreToolUse` hook wrapping `scripts/path_guard.py` adds enforcement.
- **Any harness with git (minimum viable):** paste the role file text into
  each session prompt, run `claim.py`/`release.py`/`verify.py` manually at
  the documented points, and rely on the `agent-gates` CI workflow for
  permission enforcement. Branch prefixes (`plan/`, `impl/`, `review/`)
  are the only contract the tooling needs.

## Design decisions (rationale lives with the decision)

- Session-chaining over a central orchestrator: stateless, breaks safely.
- Worktrees for implementers, locks for shared text: git isolates code;
  locks are what `specs/`/`docs/` edits need.
- Task files kept slim: detailed specs make recipes redundant; the claim
  list and per-change acceptance criteria are what machinery needs.
- Human-initiated requirement fan-out: decomposition at the requirement
  level is the human's job (intent); agents decompose only into steps
  inside a work order.
