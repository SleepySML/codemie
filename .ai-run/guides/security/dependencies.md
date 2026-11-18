# Dependencies

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Dependency surfaces, version moves, licences and the enterprise package; remediation order is in `security/README.md`.

## Overview

Python dependencies are pinned in `pyproject.toml` (Poetry, Python 3.12-3.13) with the enterprise package as an optional extra; a security pin carries a `# Security (EPMCDME-...)` comment naming the CVE.

| Part | What it holds | Start here |
|---|---|---|
| `pyproject.toml` | dependencies, extras, licence allowlist | `pyproject.toml::tool.poetry.dependencies` |
| `poetry.lock` | resolved versions | `pyproject.toml::tool.poetry.extras` |

## Dependencies

### Rules
- R1. Locate the manifest a package lives on first (`git ls-files '*pyproject.toml' '*requirements*.txt' '*.lock'`); Python resolves through Poetry, plus `hosted-applications/requirements-validator.txt`, a hash-pinned copy of `poetry.lock` versions for the validator image: when starlette, uvicorn or their dependencies move in `poetry.lock`, regenerate it in the same change. Evidence: `pyproject.toml::tool.poetry.dependencies`, `hosted-applications/requirements-validator.txt::starlette`
- R2. Bump a direct dependency with `poetry update`, force a transitive one by adding a constraint, and keep the lock in step: after a `pyproject.toml` edit run `make check-deps` (exit 1 = stale or missing lock, fix with `poetry lock`; exit 2 = Poetry not found). Evidence: `scripts/check_poetry_lock.py::check`, `Makefile::check-deps`
- R3. Admit a dependency only when its licence is in `allow-only` under `tool.pip-licenses` or a named exception; a new package needs review, not just a bump. Evidence: `pyproject.toml::tool.pip-licenses`
- R4. Resolve `codemie-enterprise` from the private source in `tool.poetry.source`; it is the optional `enterprise` extra. Evidence: `pyproject.toml::tool.poetry.extras`
- R5. Validate a fix that reaches `codemie_enterprise` against an editable install of its GitLab clone, then undo the install. Evidence: `Makefile::install-enterprise`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Bump a vulnerable package | edit its constraint in `pyproject.toml`, add the Security comment, re-lock | `pyproject.toml::datasets` |
| Allow a licence | edit `allow-only` under `tool.pip-licenses` | `pyproject.toml::tool.pip-licenses` |
| Install the enterprise extra | `make install-enterprise` | `Makefile::install-enterprise` |

## Notes

Index: [`README.md`](README.md). Owns which surface a package lives on, the command that moves it,
and the local workflow for the enterprise package.
Lock-file discipline — minimal manifest edit, targeted relock, never a wholesale regeneration — is
owned by `secops:container-cve-fix`. A full re-resolution widens the diff past the package that was
meant to change, so the security fix cannot be separated from the churn around it.
