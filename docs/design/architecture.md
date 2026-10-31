# Architecture - the oryxeval plugin

The map to read FIRST when changing this repo. It points to the operative
files rather than restating them:

- `skills/oryxeval/SKILL.md` - the skill's runtime behavior.
- root `CLAUDE.md` - dev loop, conventions, release.
- `README.md` - install + usage.
- `docs/design/design-notes.md` - WHY each non-obvious decision was made (read
  before reversing one).

## What this is

A Claude Code plugin with one skill, `oryxeval`, four manual commands and one
scaffold. They take "I changed a prompt" to a defensible number, using the
`oryxeval` Python library (`pip install oryxeval`), which is NOT shipped here:
the skill documents how to use it. It works in ANY Python repo; `evals/`
becomes one small oryxflow project. The plugin is listed in the `oryxflow`
marketplace (`/plugin install oryxeval@oryxflow`); this repo has no
`marketplace.json`.

## Components

| Path | Responsibility | Load tier |
|------|----------------|-----------|
| `.claude-plugin/plugin.json` | manifest; `version` tracks the library calver | - |
| `skills/oryxeval/SKILL.md` | activates in ANY repo on planning/editing an LLM prompt, schema, model or router input; routes to `evals.md` and the commands | on activation |
| `skills/oryxeval/evals.md` | the METHOD (layout, probe tier, four questions, mandatory guardrail, coverage before quality, real vs synthetic cases + holdout, git-ref baseline, reuse-before-you-write, judge rules, dead-metric heuristic, reading a verdict) | on demand |
| `commands/eval-plan.md` | DECIDE what an eval measures (function under test, metric, guardrail, baseline ref); writes only `evals/<name>/README.md`, no code | on invoke |
| `commands/eval-init.md` | copy `template-eval/` into the repo's `evals/` (`_env.py` once, then `run_eval_<name>.py` + `<name>/`), fill it from the plan, end in a 3-call smoke run from `evals/` | on invoke |
| `commands/eval-cases.md` | grow the case set from 3 placeholder rows to a real 15-25 (harvest real, propose axes, tag real vs synthetic) | on invoke |
| `commands/eval-run.md` | print the bill and wait, run the sweep, check the harness before believing a number, diagnose; SPENDS the user's money | on invoke |
| `resources/template-eval/` | the scaffold: `_env.py` (once per repo), `run_eval_NAME.py`, and `NAME/` (`agent.py`, `eval.py`, a 3-row `cases.csv`, `fixtures/`, `results/`) | copied by `eval-init` |
| `docs/plans/` | the two plans the commands and layout came from | dev-time |
| `docs/design/*`, `docs/CHANGELOG.md`, root `CLAUDE.md` | map, rationale, history, dev rules | dev-time |

All four commands are manual (`disable-model-invocation: true`).

## Load tiers

`SKILL.md` loads on activation; `evals.md` only when a trigger in `SKILL.md`
points at it. Every line in an activation file is paid on each load, so
`SKILL.md` stays essentials-only and depth lives in `evals.md`.

## Control & data flow - the `eval-*` chain

```
/oryxeval:eval-plan  -> reads the diff + call path; settles four questions
                     -> writes evals/<name>/README.md ONLY (no code)
/oryxeval:eval-init  -> SHELL copy of resources/template-eval/ (skip-existing):
                        _env.py once, run_eval_<name>.py, <name>/
                     -> fills the declarations (and _env.py) from that README
                     -> 3-call smoke run from evals/ (credentials + wiring)
/oryxeval:eval-cases -> harvest real, propose axes, generate -> cases.csv
/oryxeval:eval-run   -> bill -> confirm -> sweep -> results/<date>-<arms>.md
```

Deciding is a separate command from generating on purpose (design-notes
"Evals: four commands"), and `eval-init` with no plan present does the plan
step inline. Two commands write files and `eval-run` bills the user, so the
skill names them in words and never invokes one.

## Invariants (do NOT break)

| Invariant | Why / where |
|-----------|-------------|
| ASCII only in all files | Windows safety; `CLAUDE.md` |
| `SKILL.md` stays lean; depth lives in `evals.md` | load-tier cost |
| The METHOD has one home, `evals.md`; commands point at it | avoid drift |
| Every template marker has a named command that removes it | design-notes "PLACEHOLDER markers" |
| Never add install advice for the code under test; imports resolve via `evals/_env.py` | design-notes "No project install" |
| All four commands stay `disable-model-invocation: true` | design-notes |

## Where to change X

| To change... | Edit... | Then... |
|--------------|---------|---------|
| What activates the skill | `SKILL.md` frontmatter | - |
| The eval METHOD | `skills/oryxeval/evals.md` | keep `SKILL.md` pointing at it |
| What a command DOES | the matching `commands/eval-*.md` | point at `evals.md`, do not restate it |
| A scaffold file | `resources/template-eval/` directly | a marker you ADD needs an owner: `eval-init` fills declaration ones, `eval-cases` deletes the case-set one; changelog bullet |
| Cut a release | `plugin.json` `version` + `docs/CHANGELOG.md` | see root `CLAUDE.md` |
| Record a design decision | `docs/design/design-notes.md` | update this map if structure changed |
