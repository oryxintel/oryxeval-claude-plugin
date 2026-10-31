# Design notes - why the oryxeval plugin is shaped this way

Rationale behind non-obvious decisions, so future edits do not undo them by
accident. Update this when the *reasoning* changes, not just the text.

## Evals: four commands, and the method a library cannot ship

An AI engineer changes a prompt and cannot tell whether it is better. Two agent
responses to that were observed, both wrong. One wrote a throwaway probe - a
script that fires a handful of turns and prints them for a human to read in
bulk: no baseline, no metric, no control, nothing to re-run next month. The
other bypassed oryxflow entirely and hand-rolled the runner (see the quick-eval
note below). The `oryxeval` library supplies the machinery; what the
plugin adds is the PATH from "I changed a prompt" to a defensible number, and
the judgement that makes the number mean something. These notes record why that
half is shaped the way it is.

### Four commands, and why deciding is split from generating

`eval-plan` writes a plan and NO code; `eval-init` copies files. The obvious
alternative - one command that asks the questions and scaffolds in the same
breath - was rejected twice over. It would give `init-` a second meaning in this
plugin, where everywhere else (`init-project`, `init-gitlfs`) it means "copies
files" and nothing more. And an eval designed after its scaffold exists gets
designed AROUND the scaffold: the metric drifts toward whatever the template's
placeholder metric is easiest to edit into, which is the one decision in an eval
that must not be made by convenience. The split also mirrors the library's own
`preview` / `run` pairing, so the shape - decide, look at it, then commit - is
already familiar to anyone using oryxflow.

The cost of an extra command is a step someone has to know about, and that is
paid off rather than argued away: `eval-init` run with no plan present does the
plan step INLINE and waits for a yes before writing anything. The split is
therefore free to whoever ignores it and load-bearing for whoever does not.

`eval-cases` is separate on the standing test for a command: a case set GROWS
over months, so the command is run repeatedly, while init runs once per eval.
An `eval-compare` failed that same test and was rejected - comparison is what
`eval-run` already prints, and across time it is a cache read rather than a
new run, so a fifth command would earn nothing.

### Why the `eval-*` names invert the plugin's own convention

The rule lives in `CLAUDE.md` under command naming; the reasoning is here, so
a future reader files the inversion as a decision rather than a slip. It was
made in the oryxflow plugin, where the eval suite was the only multi-command
group; here every command is in the suite and the names stay as they were.

`<verb>-<noun>` is right when a command stands alone: with nothing to group, the
verb is the only useful sort key. Evals are the plugin's only MULTI-command
suite, and for a suite the grouping is worth more than the consistency - typing
`/oryxeval:eval` surfaces all four, in the order they are meant to be run, to a
user who does not yet know any of their names. That discovery is the whole
argument, and it is why the inversion does not generalize: invert only when
adding to a suite or starting one.

### Why the quick-eval rule is named and concrete in SKILL.md

Asked for a QUICK eval, a capable agent inspected an existing oryxflow-based
eval in the same repository and chose to bypass it - "no oryxflow caching -
quick run, plain asyncio." Its instincts about method were good: it built a
byte-exact baseline arm out of git history, a deterministic metric plus a
guardrail, and an LLM judge for the fuzzy part. What it rejected was the
CEREMONY, and it was not wrong that ceremony is a bad price for a quick job. It
then paid full price for every case on every iteration of the workload that
ITERATES MOST - a quick eval is by definition the one you re-run after every
tweak, so the run it declined to cache is the run that gets repeated.

Two things follow, and the second is the transferable one. The rule is named and
concrete - a "quick eval" IS `ev.sweep(fn, ...)` in a scratch file, six lines,
no class - because the failure was observed rather than imagined; a general
"prefer the cached path" would not have survived contact with an agent that had
already reasoned its way past it. And the fix was to make the cached path
SHORTER than the asyncio it lost to, not to argue for it: an argument loses to a
cheaper-looking alternative no matter how well the docs put it, while six lines
against fifteen leaves nothing to trade away. Same shape as promotion beating
rewriting: when an agent reasons past a documented
rule, change what compliance COSTS, not the volume of the argument.

### Why eval-run checks the harness before it interprets any number

A real harness capped an output field at 2000 characters for CSV readability,
and its metric read the LAST LINE of that field. Long replies were sliced
mid-sentence, so the metric scored the model down for the harness's own
scissors. Every arm scored 0%.

The generalization is the useful part and it is cheap to apply: a metric that
comes back identically 0% or 100% in EVERY arm is almost never a model result -
it is measuring the harness. That heuristic is what caught the truncation, and
it is why `eval-run` orders its steps instrument-first: look at the dead-metric
flag, read one full RAW output for a row the metric called a failure, and only
then narrate anything. Narrating first is the trap, because a plausible
mechanism fits a 0% just as comfortably as it fits a real regression, and once
written down it is what gets acted on.

### Why the baseline is a git ref, not a string replacement

An earlier design said anchored replacement: patch the live template back to its
old wording. A real harness found the limit - two of four template changes were
DELETIONS, and a replacement cannot put a deleted section back where it was. A
ref is byte-exact by construction and does not decay as the live template moves,
so `ev.git_tree(ref, paths)` is what `eval-plan` proposes and what the method
teaches.

Anchored replacement is NOT deleted from the method, because it is still the
right tool for a PROBE - a rewrite not yet shipped, which has no ref to check
out. The obligation that comes with it is that a probe must RAISE when its
anchor is gone. A probe that silently no-ops has run the baseline twice under
two arm names and reports no difference, which reads as "the change did
nothing": a wrong answer wearing the shape of a right one, the same class as the
silent data errors.

### The eval template's PLACEHOLDER markers are consumed in two stages

`resources/template-eval/` ships the oryxflow plugin's uniform `PLACEHOLDER
SCAFFOLD` marker ("not real yet"), but with TWO fillers. `eval-init` fills the
declaration markers - the production import, the metric, the guardrail, the
arm parameters - from the plan and deletes those lines. The marker over the
case set survives, because at that point the set really is still three
placeholder rows. `eval-cases` deletes that last one once a real set is
written.

Recorded because the alternatives both break the one uniform rule. Deleting
every marker at init would make a 3-row eval look finished - exactly the
"present file mistaken for real work" the marker exists to prevent. Deleting
none would leave a filled `eval.py` marked as scaffolding forever, so
`/oryxflow:check-standards` reports a finished eval as unfinished and the user
learns to ignore a marker, after which it means nothing anywhere in the
project. The general rule for any template: every marker shipped needs a named
command that removes it, and where two commands share the job, the handoff is
worth writing down.

### All four eval commands are `disable-model-invocation: true`

The same rule the other write-commands carry, with one escalation: two of these
write files, and `eval-run` spends the USER'S MONEY on API calls, so an unwanted
invocation is not a diff to revert. The skill's job at that moment is to
SUGGEST - after a prompt edit in a repo that has an `evals/` directory it names
which arm went stale and what the re-run costs, then stops. Naming the bill is
what makes suggesting sufficient rather than useless: a user who can see "24
calls, about $0.40" has what they need to type the command themselves. The
command re-derives and re-prints the bill before spending anything, so the
skill's estimate stays advisory and the gate that stops a run lives in one
place.

## Evals in any repo: one `evals/` project, a second skill, no gates

Recorded 2026-10-06, from reading a downstream consumer project (an LLM web app)
that had five hand-built evals and ~25 throwaway probe scripts.

**Why `evals/` is ONE project, with a folder per eval.** The consumer's real
cost was setup - where production code is importable from, how credentials
load (its loader resolved relative to the working directory, so a run from the
repo root "succeeded" with no credentials and empty results) - re-derived in
every eval, in five different layouts. One launch directory, one cache and one
`_env.py` solve it once. A folder per eval (rather than flat `eval_<name>.py`
files) because half an eval is hand-maintained non-code - plan, cases,
fixtures, committed results - and you work on evals one at a time. One
`run_eval_<name>.py` per eval rather than a dispatcher, because `ev.cli(Cls)`
derives its flags from one class. The shared cache keys on class name, so
class names must be unique - `eval-init` enforces it. Rejected: one project
per eval (the current shape before this change) - isolation nobody needed, at
the price of the setup duplication that actually hurt.

**Why probes are committed.** Plans in the consumer carried well-designed
probes (arms, controls, pass bars) as gitignored `tmp/probe_*.py`, deleted
after the decision. Two failures: placed outside the plan's task list, a task-
by-task executor never ran them; and nothing re-checked a surface when it
changed again two weeks later. A probe as `evals/run_eval_<name>.py` is the
same effort, cached, and still there.

**Why a skill apart from oryxflow.** The `oryxflow` skill is described as a
data-science skill, so an agent planning a router change in a web app never
loads it - and that is where evals were needed. The `oryxeval` skill owns the
eval trigger in any repo; the `oryxflow` skill keeps only a pointer. The split
into its own plugin and library (`oryxeval`) followed. It tells a planner to
put the eval INSIDE the plan's tasks, but defers to project instructions that
make evals optional: it then asks the plan to name the skipped eval rather
than drop it silently.

**Why no hard gates.** The probes set absolute pass bars, missed them, and
rewrote them as "B >= A" after the fact; results were mixed (one label up,
another down) far more often than pass/fail. So bars are written down before
the run as REFERENCES, preferably relative to the baseline arm, and the run
ends in a written judgement. The library mirrors it: with a baseline arm the
verdict reports every arm against it and names no winner.

**Why docs before source.** The probes and evals kept rebuilding what
pydantic-evals ships (a hand-rolled `asyncio.gather` runner, a hand-written
pydantic judge, tool stubs). Reading the docs first is the cheapest fix; the
installed version still wins over them.

## No project install - DO NOT REVISIT

26.10.5 of the oryxflow plugin shipped a template `pyproject.toml` + `pip
install -e .` so `import tasks` resolved outside the root. Removed; never bring
back any form of it for an eval (pyproject, setup.py, install step or advice).
Eval projects share module names (`tasks`, `flow`, `cfg`), so installs into one
interpreter shadow each other - it forces a venv per project, and agents made
one each time (bulky, and churns a synced folder). For evals, fix imports ONLY
with a `sys.path` line in `evals/_env.py` and by launching from `evals/`. A new
`ModuleNotFoundError` is a gap there - fix it there. (Installing the harness,
`oryxeval`, is fine; the rule is about the code under test.)
