# Images

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Container image stages, base image source, rebuild and scanning; Python dependency moves are in `security/dependencies.md`.

## Overview

The image has a `builder` stage that installs Python dependencies with Poetry (with the enterprise extra by default) and a `production` stage that copies the virtualenv and `/app`; both stages come from the `codemie-base-python` images, so this Dockerfile runs no `apt-get`.

| Part | What it holds | Start here |
|---|---|---|
| `Dockerfile` | build stages | `Dockerfile::INSTALL_ENTERPRISE` |
| `standalone/Dockerfile` | standalone image | `standalone/Dockerfile::FROM` |

## Container image

### Rules
- R1. Fix an OS package in the `codemie-base-python` base image repository: this Dockerfile runs no `apt-get`, so its OS packages come from the base images. Evidence: `Dockerfile::FROM`
- R2. Expect a `builder` stage that installs Python dependencies with Poetry and a `production` stage that copies the virtualenv and `/app`, both on `codemie/codemie-base-python` images. Evidence: `Dockerfile::FROM`
- R3. Build the enterprise image with the `google_credentials` secret; without it `INSTALL_ENTERPRISE=true` cannot install the private extra and a scan proves nothing about the shipped image. Evidence: `Dockerfile::INSTALL_ENTERPRISE`
- R4. Rebuild, tag the verification build distinctly, and scan that tag with the scanner that reported the finding; confirm the named CVE is gone. Evidence: `Dockerfile::PYTHON_VERSION`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A scan of a local build reports the enterprise package missing or clean | without the `google_credentials` secret the builder cannot install the private extra | build with the secret, or hand the rebuild to someone who holds it | `Dockerfile::INSTALL_ENTERPRISE` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Build without enterprise | pass `--build-arg INSTALL_ENTERPRISE=false` | `Dockerfile::INSTALL_ENTERPRISE` |
| Change the Python version | edit `ARG PYTHON_VERSION` | `Dockerfile::PYTHON_VERSION` |
| Fix an OS package | change the base image repository, not this Dockerfile | `Dockerfile::FROM` |

## Notes

Index: [`README.md`](README.md). Owns the image, its stages, the enterprise build secret, and how OS
packages are pinned in it.
Rebuild-and-rescan as the verification predicate for a container CVE is owned by
`secops:container-cve-fix` and `secops:trivy-scan`. No gate in this repository looks inside the
Scope is remediation. Running the project locally follows the onboarding repository —
<https://gitbud.epam.com/epm-cdme/codemie-onboarding#docker-compose-structure--usage>.
