# oryxeval - the Claude Code plugin for measuring LLM prompt changes

Changed a prompt, a response schema, a model or a router input and cannot tell
whether it is better? This plugin takes that question to a defensible number,
in ANY Python repo. It drives the `oryxeval` library: cached eval arms,
baselines from git, and intervals.

## Install

The plugin is listed in the `oryxflow` marketplace:

```
/plugin marketplace add https://github.com/oryxintel/oryxflow-claude-plugin.git
/plugin install oryxeval@oryxflow
```

Then install the library into the environment that runs your production code:
`pip install "oryxeval>=26.11.0"`.

An `evals/` written against `oryxflow.evals` keeps its cache only if you change
its import lines and nothing else; the steps are the `oryxflow.evals` rule in
[skills/oryxeval/SKILL.md](skills/oryxeval/SKILL.md).

To load a local clone without installing:
`claude --plugin-dir /path/to/oryxeval-claude-plugin`.

## Measure a prompt change

Four commands, in order:

```
/oryxeval:eval-plan    # decide what is measured - writes the plan, no code
/oryxeval:eval-init    # scaffold the eval from the plan, ending in a smoke run
/oryxeval:eval-cases   # grow the case set from 3 placeholders to a real 15-25
/oryxeval:eval-run     # run it, print the verdict, interpret the result
```

`evals/` becomes one small oryxflow project of its own: you launch every eval
from it, it holds one cache, and `evals/_env.py` records once how this repo
loads credentials and imports production code.

`eval-plan` settles the questions that decide whether an eval means
anything - the function under test, the metric, the guardrail that must not get
worse, and the git ref to beat - with proposals drawn from your actual diff, and
records the answers in `evals/<name>/README.md`. `eval-init` copies the eval
template into `evals/`, wires it to those answers without overwriting anything,
and finishes with a 3-call smoke run, so a missing API key surfaces as an error
instead of an empty result that caches as a measurement. `eval-cases` harvests
real cases out of the repository, says plainly what it could not find, and
proposes the axes the set is missing (controls, needs-user-input, mood pairs)
rather than a flat list. `eval-run` prints the projected bill and waits before
spending anything, then checks the harness for a dead metric before it believes
any number.

Every cell is cached, so a re-run costs nothing for what did not change - edit
one prompt and only that arm re-runs; edit a scorer or a judge rubric and only
the scoring re-runs, with no model calls. All four commands are manual (two
write files, and `eval-run` bills you for API calls); the skill will volunteer
that an arm has gone stale, with the cost, but never runs one for you.

### Getting an agent in another repo to use it

1. Install the plugin for that project (or user-wide), as above.
2. Install the library into that project's environment:
   `pip install "oryxeval>=26.11.0"`.
3. Optionally add this block to the project's `CLAUDE.md`, filled in, so every
   session knows the project specifics without rediscovering them:

```markdown
## LLM evals
- Evals live in `evals/` (oryxeval); run every one FROM `evals/`.
- `evals/_env.py` loads credentials from <where> and makes <package> importable.
- A plan that changes a prompt, response schema, model or router input names the
  eval that measures it, as a task in the plan. Extend an existing eval in
  `evals/` before writing a new one; a quick probe is `evals/run_eval_<name>.py`.
- Commands: /oryxeval:eval-plan -> eval-init -> eval-cases -> eval-run.
```

The `oryxeval` skill triggers on its own when an agent plans or edits an LLM
prompt; the block adds what only your project knows.

## Developing the plugin

See [CLAUDE.md](CLAUDE.md) and [docs/design/architecture.md](docs/design/architecture.md).
Updates are pull-based: a consumer receives a change only when `plugin.json`
`version` changes ([CHANGELOG](docs/CHANGELOG.md)).

## Contents

```
.claude-plugin/plugin.json
commands/          # eval-plan, eval-init, eval-cases, eval-run
skills/oryxeval/   # SKILL.md (any repo) + evals.md (the method, on demand)
resources/template-eval/   # the files eval-init copies into a repo's evals/
docs/              # CHANGELOG, design notes + architecture, plans
```

## License

MIT
