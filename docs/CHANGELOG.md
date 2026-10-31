# Changelog

All notable changes to the oryxeval plugin are recorded here. Format follows
[Keep a Changelog](https://keepachangelog.com/); the version tracks the
`oryxeval` library release it pairs with. `## [Unreleased]` ships
nothing until it is stamped with a version (see the repo `CLAUDE.md`).

Entry conventions: a bullet-leading `BREAKING:` token when a change breaks an
existing user, with a same-bullet `Migration:` clause; backticked command and
file names.

## [Unreleased]

### Added
- First release: the `oryxeval` skill, the four manual commands
  `/oryxeval:eval-plan`, `/oryxeval:eval-init`, `/oryxeval:eval-cases` and
  `/oryxeval:eval-run`, and the `resources/template-eval/` scaffold, moved from
  the oryxflow plugin. Needs `pip install "oryxeval>=26.11.0"`. Install:
  `/plugin install oryxeval@oryxflow`. Migrating an existing `evals/`: change
  only its import lines (the skill's `oryxflow.evals` rule).
