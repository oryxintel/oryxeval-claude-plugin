# CLAUDE.md - working ON the oryxeval plugin

This repo is the **source of a Claude Code plugin**. A session here is for
*developing the plugin*, NOT for using it. You are editing skill source; do not
run the oryxeval skill against this repo (there is no eval here).

**Read `docs/design/architecture.md` first.** It is the map of this repo and a
"to change X, edit Y" playbook.

Work as a senior engineer. Prioritize by real user pain in the agents' feedback
notes: guidance that led to wrong or stale results first, then recurring
friction, then polish. Work mostly autonomously, but list every major decision
the user might reverse (guidance changed or removed, conventions enforced,
deferring a reported issue) under "Decisions" in your report, and stop to ask
before releases, pushes, marketplace publishing or anything irreversible.

Use repo-relative paths; do not hardcode absolute machine paths. When a tool
needs an absolute path, copy the primary working directory verbatim.

## What this plugin is

One skill, `oryxeval` (activates in ANY Python repo when an LLM prompt, schema
or model change is planned or made), four manual commands and one scaffold.
The library is `oryxeval` on PyPI (`pip install oryxeval`); this repo only
teaches an agent to use it. The plugin is listed in the `oryxintel` marketplace
(`oryxeval@oryxintel`); this repo has no `marketplace.json`.

## Docs

User-facing docs live in the sibling repo `../oryxlib-docs` (site docs.oryxflow.dev), not here.
Read `../oryxlib-docs/MAINTAINING.md` before editing pages. Any user-facing change here
(commands, install, skill behaviour users see) needs a docs update there, committed in that
repo. Pages for this plugin (the ones that name its `/oryxeval:` slash commands):
`docs/docs/claude-plugin/evals.md` and `docs/docs/claude-plugin/commands.md`. The oryxeval library
pages (`llm-evals*.md`) do not mention the commands. The changelog stays
HERE (`docs/CHANGELOG.md`).

## Layout

```
.claude-plugin/plugin.json   # manifest
commands/eval-{plan,init,cases,run}.md   # /oryxeval:eval-*
skills/oryxeval/SKILL.md     # entry point, always in context on activation
skills/oryxeval/evals.md     # the eval METHOD, loaded ON DEMAND
resources/template-eval/     # the scaffold eval-init copies into evals/
docs/{CHANGELOG.md,plans/,design/}
```

## Authoring conventions (match the existing files)

- **ASCII only.** No emojis/unicode/smart quotes (Windows safety).
- **Wrap prose at ~78 columns.**
- **The commands are `eval-<verb>`, not `<verb>-<noun>`, DELIBERATELY:** typing
  `/oryxeval:eval` surfaces all four in run order. A new standalone command
  would take `<verb>-<noun>`; invert only when adding to the suite.
- **A command that writes files, runs git or spends money is
  `disable-model-invocation: true`** - the USER types it. All four carry it
  (two write files; `eval-run` bills API calls). The skill names a command in
  words; it never invokes one.
- **Two-tier content split is load-bearing:** `SKILL.md` holds the essentials;
  depth goes to `evals.md`. A new on-demand file needs a trigger in `SKILL.md`.
- **Be token-efficient in ALL plugin instructions** (skill, commands, template
  comments, hook messages, changelog bullets): each is read by an agent and paid
  in context. Say it once, in the fewest words that land; the "why" goes in
  `design-notes.md`. A rule that gets *longer* under edit is a smell.
- **Address the agent directly (second person).** Never "the agent" or "we".
- **Never add install advice for the code under test** (pyproject, editable
  install, a venv); imports resolve via the `sys.path` line in
  `evals/_env.py`. Installing `oryxeval` itself is fine.
- Behavior change: update `design-notes.md` if the *rationale* changed, and
  `docs/CHANGELOG.md` always.

## Never write private identifiers into repo files

**This repo is public and git history is forever.** Applies to every committed
file and commit message, *especially* `docs/plans/`, where real-world context
creeps in. Sanitize as you WRITE: scrubbing later needs a history rewrite.

| Don't | Do |
| --- | --- |
| Absolute local paths - `D:\Users\<name>\dev\thing`, `/home/<name>/...` | `/path/to/<repo>`, a repo-relative path, or the repo name |
| An out-of-repo file's location | "tracked in an external note, `<filename>.md`" |
| Client / customer / employer names, incl. as a directory or project slug | "a downstream consumer project" |
| A client's data vendors, systems, or task/class names that embed them | A generic stand-in |
| Other private project codenames | "a private sibling project", or omit |
| Anything credential-shaped - tokens, keys, bucket names, internal hostnames | A placeholder (`<PYPI_TOKEN>`), or say where it is stored |

A plan or example must stay executable after sanitizing: keep its shape, drop
only the identity. `.githooks/pre-commit` runs `check_private_terms.py`, which
blocks added lines holding a local path or a listed client name. List the names,
one per line, in `~/.oryx-private-terms` or a gitignored `.private-terms`.
`--all` audits every tracked file.

## Develop / test loop

```
claude --plugin-dir /path/to/oryxeval-claude-plugin   # load without installing
/reload-plugins                                       # after each edit
claude plugin validate .                              # check the manifest
git config core.hooksPath .githooks                   # ONE-TIME: enable hooks
```

The pre-commit hook enforces: every leading `BREAKING:` changelog bullet has a
`Migration:` clause, and no private identifiers in added lines.

## Release

Publishing is pull-based and VERSION-GATED: `plugin.json` sets an explicit
`version`, so a consumer only receives a change when that string CHANGES. The
bump IS the publish.

- **While iterating:** add bullets under `## [Unreleased]` in
  `docs/CHANGELOG.md` (`### Added` / `### Changed` / `### Removed`). Do NOT
  touch `plugin.json`. Commit freely.
- **Cutting a release:** rename `## [Unreleased]` to `## [VERSION] - YYYY-MM-DD`
  and set `plugin.json` `version` to the SAME string (the top dated heading and
  `version` must be equal); add a fresh empty `## [Unreleased]`. The version
  tracks the `oryxeval` library release it pairs with. Run `claude plugin
  validate .`, commit, push.

## Git

- Imperative summary line; end with the Co-Authored-By trailer.
- Default branch is `main`. Do not use `master`.
- Bash tool: PowerShell here-strings (`@'...'@`) are not here-strings; use
  repeated `-m` flags with plain quoted strings.
