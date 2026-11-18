# AGENTS.md

**Purpose**: AI-optimized execution guide for agents working with the CodeMie backend repository.

Detailed project guidance lives under `.ai-run/guides/`, which is the source of truth for AI-assisted development in this repo.

## Guide Imports

<!-- ai-run-init:guide-imports start -->
| Category | Guide Path | Purpose |
|---|---|---|
| Project | `.ai-run/guides/project.md` | Project identity, tracker, source control, and MR adapter context |
| Quality Gates | `.ai-run/guides/quality-gates.md` | Exact validation commands and gate policy |
| A2A | `.ai-run/guides/integration/a2a-integration.md` | A2A protocol version decision and wire-shape markers |
| API | `.ai-run/guides/api/endpoint-conventions.md` | Request and response models, PUT and POST semantics |
| API | `.ai-run/guides/api/rest-api-patterns.md` | FastAPI router patterns |
| Agents | `.ai-run/guides/agents/agent-tools.md` | Agent-facing tool ownership and schema compatibility |
| Agents | `.ai-run/guides/agents/langchain-agent-patterns.md` | LangChain agent runtime patterns |
| Architecture | `.ai-run/guides/architecture/layered-architecture.md` | Layered architecture |
| Architecture | `.ai-run/guides/architecture/project-structure.md` | Package boundaries of codemie and codemie_tools |
| CLI Analytics Events | `.ai-run/guides/integration/cli-analytics-events.md` | Contract of the six plugin events: JSON, stored columns, identity and re-sends |
| CLI Analytics Storage | `.ai-run/guides/integration/cli-analytics-storage.md` | PostgreSQL storage for OTel CLI analytics: settings, sizing, operations |
| Cloud | `.ai-run/guides/integration/cloud-integrations.md` | Cloud provider integrations |
| Code Quality | `.ai-run/guides/standards/code-quality.md` | Python and Ruff standards |
| Configuration | `.ai-run/guides/development/configuration-patterns.md` | Config and environment patterns |
| Container Image | `.ai-run/guides/security/images.md` | Image stages, base image source and scanning |
| Database | `.ai-run/guides/data/database-patterns.md` | SQLModel and session patterns |
| Database | `.ai-run/guides/data/repository-patterns.md` | Repository boundary and storage factories |
| Error Handling | `.ai-run/guides/development/error-handling.md` | Typed exceptions and handlers |
| Error Handling | `.ai-run/guides/development/logging-patterns.md` | Contextual logging, severity and metric failure labels |
| FAQ Git | `.ai-run/guides/integration/faq-git-integration.md` | Git FAQ (llm_routing_faq) datasource: parser, routing, contracts |
| Git Workflow | `.ai-run/guides/standards/git-workflow.md` | Branch, commit, and review conventions |
| Google Docs | `.ai-run/guides/integration/google-docs-integration.md` | Google Docs integration |
| Integrations | `.ai-run/guides/integration/external-services.md` | Adapter boundary, credentials, provider errors and inbound webhooks |
| Integrations | `.ai-run/guides/integration/llm-providers.md` | Model provider config, LiteLLM, request tagging, speech-to-text |
| Local Setup | `.ai-run/guides/development/local-testing.md` | Test scope and environment blocks when running locally |
| Local Setup | `.ai-run/guides/development/setup-guide.md` | Environment files, local dependencies and the Stop hook |
| Local Verification | `.ai-run/guides/testing/local-verification.md` | Harness sanity, area tests and live API checks before an MR |
| MCP | `.ai-run/guides/integration/mcp-integration.md` | MCP configuration and tools |
| Performance | `.ai-run/guides/development/performance-patterns.md` | Async and batching patterns |
| Request Hedging | `.ai-run/guides/integration/request-hedging.md` | Hedged tool and provider calls for assistants |
| Search | `.ai-run/guides/data/database-optimization.md` | Pagination, batching and search size limits |
| Search | `.ai-run/guides/data/elasticsearch-integration.md` | Elasticsearch repositories and index setup |
| Security | `.ai-run/guides/development/security-patterns.md` | Auth, validation, and secrets |
| Service Layer | `.ai-run/guides/architecture/service-layer-patterns.md` | Service orchestration |
| Service Testing | `.ai-run/guides/testing/testing-service-patterns.md` | Service test patterns |
| Test Management | `.ai-run/guides/integration/xray-integration.md` | X-ray datasource and QA tools |
| Testing | `.ai-run/guides/testing/testing-api-patterns.md` | Router tests and router contract guards |
| Testing | `.ai-run/guides/testing/testing-patterns.md` | pytest policy and patterns |
| Tool Creation | `.ai-run/guides/agents/custom-tool-creation.md` | Creating a toolkit and tool, native connector requirements |
| Tool Creation | `.ai-run/guides/agents/tool-overview.md` | Toolkit layers and the static tool registry |
| Vulnerability Remediation | `.ai-run/guides/security/README.md` | CVE remediation: order, `/sanity` handoff, traps; routes on to dependencies and images |
| Vulnerability Remediation | `.ai-run/guides/security/dependencies.md` | Dependency surfaces, licences and the enterprise package |
| Work Item Sources | `.ai-run/guides/integration/confluence-integration.md` | Confluence datasource processing |
| Work Item Sources | `.ai-run/guides/integration/jira-integration.md` | Jira datasource and project-management tools |
| Workflows | `.ai-run/guides/workflows/langgraph-workflows.md` | LangGraph workflow patterns |
<!-- ai-run-init:guide-imports end -->

## Task Classifier

<!-- ai-run-init:task-classifier start -->
| Category | User Intent | Example Requests | P0 Guide | P1 Guide |
|---|---|---|---|---|
| A2A | A2A protocol | "A2A agent card", "A2A message shape" | `.ai-run/guides/integration/a2a-integration.md` | - |
| API | FastAPI routes, routers, validation | "add endpoint", "split a router into a package" | `.ai-run/guides/api/rest-api-patterns.md` | `.ai-run/guides/api/endpoint-conventions.md` |
| Agents | agents, callbacks, agent runtime | "modify agent", "agent output handling" | `.ai-run/guides/agents/langchain-agent-patterns.md` | `.ai-run/guides/agents/agent-tools.md` |
| Architecture | system structure, layer boundaries, where code belongs | "where should this go?", "refactor service boundaries" | `.ai-run/guides/architecture/layered-architecture.md` | `.ai-run/guides/architecture/project-structure.md` |
| CLI Analytics | CLI analytics events | "new event field", "event classification" | `.ai-run/guides/integration/cli-analytics-events.md` | `.ai-run/guides/integration/cli-analytics-storage.md` |
| Cloud | AWS, Azure, GCP, Kubernetes | "cloud tool change", "S3 or blob storage" | `.ai-run/guides/integration/cloud-integrations.md` | - |
| Code Quality | code style, lint, typing | "fix lint", "where to put new code in a module" | `.ai-run/guides/standards/code-quality.md` | - |
| Configuration | config variables, feature flags | "config var", "feature flag" | `.ai-run/guides/development/configuration-patterns.md` | - |
| Container Image | container image findings, rebuild and rescan | "patch base image", "rescan container" | `.ai-run/guides/security/images.md` | - |
| Database | PostgreSQL models, sessions, migrations | "add a table", "write a migration" | `.ai-run/guides/data/database-patterns.md` | `.ai-run/guides/data/repository-patterns.md` |
| Error Handling | exceptions, error responses | "handle error", "raise a 404" | `.ai-run/guides/development/error-handling.md` | `.ai-run/guides/development/logging-patterns.md` |
| FAQ Git | FAQ git datasource | "FAQ article format", "FAQ routing" | `.ai-run/guides/integration/faq-git-integration.md` | - |
| Git | only when user explicitly asks git ops | "commit", "create MR" | `.ai-run/guides/standards/git-workflow.md` | - |
| Google Docs | Google Docs datasource | "Google Doc indexing", "Google auth for docs" | `.ai-run/guides/integration/google-docs-integration.md` | - |
| Integrations | external service calls, webhooks | "add integration", "fix provider call" | `.ai-run/guides/integration/external-services.md` | `.ai-run/guides/integration/llm-providers.md` |
| Local Setup | local environment, poetry, docker compose | "set up the project", "Stop hook fails" | `.ai-run/guides/development/setup-guide.md` | `.ai-run/guides/development/local-testing.md` |
| Local Verification | verify a change before an MR | "verify before MR", "harness run fails" | `.ai-run/guides/testing/local-verification.md` | - |
| MCP | MCP servers and tools | "MCP server times out", "MCP config" | `.ai-run/guides/integration/mcp-integration.md` | - |
| Performance | async, background tasks, streaming | "slow endpoint", "run work in background" | `.ai-run/guides/development/performance-patterns.md` | - |
| Request Hedging | request hedging | "add a hedge tool", "tune hedging timeout" | `.ai-run/guides/integration/request-hedging.md` | - |
| Search | Elasticsearch, index, search queries | "ES query", "index stats" | `.ai-run/guides/data/elasticsearch-integration.md` | `.ai-run/guides/data/database-optimization.md` |
| Security | auth, permissions, input validation, secrets | "auth bug", "sanitize input" | `.ai-run/guides/development/security-patterns.md` | - |
| Service Layer | business logic in services, orchestration | "add a service method", "move logic out of a router" | `.ai-run/guides/architecture/service-layer-patterns.md` | - |
| Service Testing | service-level tests | "test a service", "async test" | `.ai-run/guides/testing/testing-service-patterns.md` | - |
| Test Management | X-ray, Zephyr, test management | "X-ray import", "QA tool change" | `.ai-run/guides/integration/xray-integration.md` | - |
| Testing | only when user explicitly asks tests | "write tests", "fix failing test" | `.ai-run/guides/testing/testing-patterns.md` | `.ai-run/guides/testing/testing-api-patterns.md` |
| Tool Creation | new toolkit, new tool, native connector | "add tool", "add a toolkit for a new service" | `.ai-run/guides/agents/custom-tool-creation.md` | `.ai-run/guides/agents/tool-overview.md` |
| Vulnerability Remediation | a scanner or CVE ticket names this repo | "fix a reported CVE", "bump vulnerable package" | `.ai-run/guides/security/README.md` | `.ai-run/guides/security/dependencies.md` |
| Work Item Sources | Jira datasource and tools | "Jira datasource bug", "Jira tool change" | `.ai-run/guides/integration/jira-integration.md` | `.ai-run/guides/integration/confluence-integration.md` |
| Workflows | workflow node, transition, interrupt | "workflow node", "transition" | `.ai-run/guides/workflows/langgraph-workflows.md` | - |
| Development | errors, logging, config, performance | handle error; add logging; config var | `.ai-run/guides/development/error-handling.md` | `.ai-run/guides/development/logging-patterns.md` |
<!-- ai-run-init:task-classifier end -->

## Critical Rules

<!-- ai-run-init:critical-rules start -->
| Rule | Trigger | Action |
|---|---|---|
| Check Guides First | ANY task | Match request to category, load the P0 `.ai-run/guides/` guide before broad code search. |
| Testing | "write tests" / "run tests" | ONLY then |
| Git Operations | User asks to commit, push, create MR, or similar | Only then perform git side effects; load `.ai-run/guides/standards/git-workflow.md` first. |
| Shell | ANY shell command | Use bash/Linux syntax and report commands actually run. |
| Ask a reviewer to post /sanity in the MR description | Opening an MR that touches dependencies, the Dockerfile, or security-relevant code | load `.ai-run/guides/security/README.md` |
| Expect the lint and full-verification gates to rewrite files and re-stage them | Running the lint or full verification gate | load `.ai-run/guides/quality-gates.md` |
| Load the setup guide's Stop hook section | The Claude Code Stop hook fails because Poetry is not on PATH | load `.ai-run/guides/development/setup-guide.md` |
| Testing | User asks to write, run, or fix tests | Only then work on tests; load `.ai-run/guides/testing/testing-patterns.md` first. |
| Python Environment | Python, Poetry, or tool commands | Load `.ai-run/guides/development/setup-guide.md` or `.ai-run/guides/quality-gates.md` before running. |
| Stop Hook | `make ruff` fails with `poetry: No such file or directory` | Load `.ai-run/guides/development/setup-guide.md` — see **Claude Code Stop Hook** section. |
| Project Conventions | Any project-specific convention | Load the relevant guide; do not infer exact values from this entrypoint. |
<!-- ai-run-init:critical-rules end -->

## Repository Rules

<!-- Outside the ai-run-init managed regions on purpose: the generator preserves content
     outside its markers, and this rule is not derivable from the guides or manifests. -->

| Rule | Trigger | Action |
|---|---|---|
| Sanity Regression | Opening an MR that touches dependencies, the Dockerfile, or security-relevant code | This repository has no pipeline config, so nothing re-checks an MR automatically. The MR description must ask a reviewer to post `/sanity`. |

## Commands

<!-- ai-run-init:commands start -->
| Need | Source Guide | Source Evidence | Notes |
|---|---|---|---|
| Quality gates (lint, license-check, secret-scan, build, test, static-analysis) | `.ai-run/guides/quality-gates.md` | each gate's Run, Pass and Fail in the guide | Load guide before running |
| Git / review workflow | .ai-run/guides/standards/git-workflow.md | git history and project context | Load guide before git operations |
| Setup and local environment | `.ai-run/guides/development/setup-guide.md` | README, Makefile, pyproject | Load before dependency or environment commands. |
| Run the application | `.ai-run/guides/development/setup-guide.md` | README, Makefile | Load before starting local services. |
| Tests and coverage | `.ai-run/guides/testing/testing-patterns.md` | Makefile, pytest config | Only run or write tests when explicitly requested. |
| Secure coding, auth, secret handling | `.ai-run/guides/development/security-patterns.md` | Quality gates guide, Makefile | Application-level security, not vulnerability remediation. |
| Fixing a reported CVE or scanner finding | `.ai-run/guides/security/README.md` | Makefile, pyproject, Dockerfile | Covers dependencies, image rebuild and scan, and the `/sanity` handoff. |
<!-- ai-run-init:commands end -->

## Pre-Delivery Checklist

- Requirements handled and scoped to the user request.
- Relevant `.ai-run/guides/` P0 guide was checked first.
- Project-specific exact values came from a guide, not from this entrypoint.
- Validation commands actually run are reported exactly in the final response.
- Tests are only run or written when explicitly requested.
- Git operations are only performed when explicitly requested.

## Conflict Handling

| Conflict | Action |
|---|---|
| A guide conflicts with source code | Trust current source, then update guidance only if the task owns that documentation change. |
| README and Makefile disagree on a command | Prefer the Makefile target and note the README mismatch in the response. |
| User asks for tests but scope is unclear | Ask for the intended scope or run the narrowest relevant test. |
| User asks for git work without a required work item | Ask for the missing work item before committing. |
| Existing host-specific skills disagree with `.ai-run/guides/` | Follow `.ai-run/guides/` unless the user explicitly selects the host-specific workflow. |

## Source Evidence Priority

| Evidence type | Priority |
|---|---|
| Current source files | Highest for implementation behavior. |
| `Makefile` and manifests | Highest for commands and dependencies. |
| README and contribution docs | Useful for workflow context. |
| `.ai-run/guides/` | Primary AI guidance source. |
| Host-specific skill files | Use only when that skill is explicitly invoked. |

## Behavioral Defaults

<!-- ai-run-init:behavioral-defaults start -->
Behavioral guidelines to reduce common LLM coding mistakes. The Critical Rules above take precedence where they differ.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask; in an unattended run, record the assumption where the run records its decisions.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing before you go on.

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals; write or run tests only as the Critical Rules allow:
- "Add validation" → "Name the invalid inputs it must reject and how each is checked"
- "Fix the bug" → "Reproduce it, then show it no longer happens"
- "Refactor X" → "Show the behavior is the same before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

Source: [andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills).
<!-- ai-run-init:behavioral-defaults end -->
