# Code Quality Standards

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Ruff-enforced tooling, naming, typing, complexity, imports and module cohesion; gate commands are in `quality-gates.md`.

## Overview

Ruff is the only linter and formatter: `make ruff` formats, applies safe fixes and checks; the pre-commit hook checks staged Python files and license headers. Rules are in `pyproject.toml` under `tool.ruff`.

| Part | What it holds | Start here |
|---|---|---|
| `pyproject.toml` | Ruff config | `pyproject.toml::tool.ruff` |
| `scripts/git-hooks/` | pre-commit, commit-msg, pre-push hooks | `scripts/git-hooks/_ruff_staged.sh::changed_files` |
| `scripts/license_headers/` | Apache header checker | `scripts/license_headers/check_license_headers.py::EXCLUDE_DIR_PREFIXES` |

## Enforced tooling

### Rules
- R1. Run the lint command `.ai-run/guides/quality-gates.md` names before committing, and `git add` the files it rewrites (the `codemie-pre-commit` hook fails when Ruff would change a staged file). Evidence: `.pre-commit-config.yaml::codemie-pre-commit`
- R2. Give a new `.py` or `.sh` file the Apache 2.0 header; `make license-fix` adds it (the pre-commit hook runs the checker in `--check` mode). Evidence: `scripts/git-hooks/pre_commit.sh::LICENSE_CMD`
- R3. Leave generated and external code (`src/external/*`, generated plugin code) to its configured ignores instead of editing it for style. Evidence: `pyproject.toml::tool.ruff.lint.per-file-ignores`

## Naming

### Rules
- R1. Follow the pep8-naming rules Ruff selects (`N`); mixedCase class attributes (N815), an exception name without an `Error` suffix (N818) and a first argument other than `self` (N805) are permitted, not required. Evidence: `pyproject.toml::N818`

## Type safety

### Rules
- R1. Write new annotations as `X | None` and parameterized generics (`list[str]`), and spell out the union a duck-typed parameter accepts. Evidence: `src/codemie/clients/postgres.py::PostgresClient.get_libpq_url`
- R2. Annotate the parameters and return type of a new public function; no type checker runs in the gates, so annotations are checked in review. Evidence: `src/codemie/clients/postgres.py::PostgresClient.get_libpq_url`

## Complexity limits

### Rules
- R1. Keep a function's McCabe complexity at 16 or below (Ruff `C901` fails above `max-complexity = 16`); `kata_import_service.py` is the single file-level exception. Evidence: `pyproject.toml::max-complexity`
- R2. Put a new concern in a new module next to its user; do not grow a file of about 500 lines with code outside its responsibility. Evidence: `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport`

## Imports

### Rules
- R1. Leave the order of import lines you do not touch: Ruff's isort rules (`I`) are not selected, so `known-first-party` sorts nothing and a reorder adds diff noise. Evidence: `pyproject.toml::known-first-party`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| The commit is rejected with "Ruff would change the following staged files" | the hook formats the staged blob and fails when the result differs | run `make ruff`, `git add` the files and commit again | `scripts/git-hooks/_ruff_staged.sh::changed_files` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Format and fix | `make ruff` | `Makefile::ruff` |
| Add missing license headers | `make license-fix` | `Makefile::license-fix` |
| Exclude a path from license checks | add it to `EXCLUDE_DIR_PREFIXES` | `scripts/license_headers/check_license_headers.py::EXCLUDE_DIR_PREFIXES` |
