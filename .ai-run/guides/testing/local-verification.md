# Local Verification

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Harness sanity run and live API checks before an MR; unit and integration tests are in `testing/testing-patterns.md`.

## Overview

`make test-harness` runs the external `codemie-test-harness` sanity suite through `uvx` against a running backend; the MR template asks for its summary in a `## Test harness` section.

| Part | What it holds | Start here |
|---|---|---|
| `Makefile` | harness target | `Makefile::test-harness` |
| `.gitlab/merge_request_templates/Default.md` | MR sections the bot checks | `.gitlab/merge_request_templates/Default.md` § Test harness |

## Local verification

### Rules
- R1. Before an MR, run `make test-harness` against a backend started from the branch, then live API calls for what the change does. Evidence: `Makefile::test-harness`
- R2. Against a local backend pass explicit worker and rerun limits (`uvx codemie-test-harness run sanity-api -n 2 --reruns 2`); the shortcut's parallelism can overload it. Evidence: `Makefile::test-harness`
- R3. Drive the change through its public API with a positive control and its failure paths, and record request, response and log lines. Evidence: `Makefile::run`
- R4. Reproduce a failure on `main` with the same complete configuration before calling it pre-existing, and compare pass rates over several runs for LLM-judged tests. Evidence: `Makefile::test-harness`
- R5. Put the harness summary in the MR's `## Test harness` section and each live scenario under Verified locally. Evidence: `.gitlab/merge_request_templates/Default.md::harness`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Run the sanity suite | `make test-harness` | `Makefile::test-harness` |
| Start the backend from the branch | `make run` | `Makefile::run` |
| Report results | fill the Test harness section of the MR template | `.gitlab/merge_request_templates/Default.md` § Test harness |

## Notes

How to verify a change beyond unit tests before opening an MR: the harness sanity run and live API calls. Gate commands and the MR compliance rules stay in
`.ai-run/guides/quality-gates.md`; this guide is the procedure for choosing and running the extra checks.
