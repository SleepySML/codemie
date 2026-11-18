# Vulnerability Remediation

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: CVE remediation order, discovery and the regression handoff; dependency surfaces are in `security/dependencies.md` and the container image in `security/images.md`.

## Overview

Security fixes in this repository land in `pyproject.toml`/`poetry.lock` for Python packages and in the base image repository for OS packages; `make verify` runs ruff, license, gitleaks and the tests.

| Part | What it holds | Start here |
|---|---|---|
| `pyproject.toml` | Python pins with `Security (EPMCDME-...)` comments | `pyproject.toml::datasets` |
| `Dockerfile` | builder and production stages on base images | `Dockerfile::INSTALL_ENTERPRISE` |
| `.gitleaks.toml` | secret-scan allowlist | `Makefile::gitleaks` |

## Vulnerability remediation

### Rules
- R1. Expect `git commit` to run the staged Ruff check, `ruff check`, the license check and the container gitleaks scan; tests and `make sonar-local` run at `git push` when `CODEMIE_PREPUSH_ENABLED` is on, and `CODEMIE_PRECOMMIT_ENABLED=false` skips the gitleaks hook alone. Evidence: `scripts/git-hooks/pre_commit.sh::LICENSE_CMD`, `scripts/git-hooks/pre_push.sh::CODEMIE_PREPUSH_ENABLED`, `scripts/git-hooks/validate_secrets.sh::CODEMIE_PRECOMMIT_ENABLED`
- R2. Ask a reviewer in the MR description to post `/sanity`: `.gitlab-ci.yml` runs only a `pre-ci-guard` stage on MR pipelines and none of its jobs runs a Makefile gate. Evidence: `.gitlab-ci.yml::wait-ci-init`
- R3. Fix a Python package in `pyproject.toml` and the lock, not as an image-level override, and an OS package in the base image; mark the pin with `# Security (EPMCDME-...)` naming the CVEs. Evidence: `pyproject.toml::datasets`
- R4. Derive gate lists, pins and manifests by command (`grep -n '^verify:' Makefile`, `git log --grep=CVE`) instead of a copied list. Evidence: `Makefile::verify`
- R5. After a manifest change run `poetry check --lock`, then `make verify`; a change reaching enterprise code is tested against the GitLab clone of the enterprise package. Evidence: `Makefile::verify`
- R6. Hand a committed secret to a human for rotation; scrubbing the tree leaves the secret in history. `make verify` includes `make gitleaks`. Evidence: `Makefile::gitleaks`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Run the full local check | `make verify` | `Makefile::verify` |
| Scan for secrets | `make gitleaks` | `Makefile::gitleaks` |
| Find earlier CVE fixes | `git log --grep=CVE` | `pyproject.toml::datasets` |

## Notes

Entry point for remediating a reported vulnerability in this repository. Owns the remediation
order, the discovery commands, the exit-code traps, and the `/sanity` handoff.
Process policy — classify, route, fix, verify, deliver — is owned by the `secops` bundle
(`secops:security-remediate` and the per-class fix skills). This file records only what is
specific to this repository.

| When | Read |
|---|---|
| You need the gate commands and skip policy | [`../quality-gates.md`](../quality-gates.md) |
| You are changing a package | [`dependencies.md`](dependencies.md) |
| The finding is in the image, a stage, or an OS package | [`images.md`](images.md) |
| You need branch, commit, or MR mechanics | [`../standards/git-workflow.md`](../standards/git-workflow.md) |
| You are writing code — auth, input validation, secret handling | [`../development/security-patterns.md`](../development/security-patterns.md) |
