# Project Context

## Project Identity

| Field | Value | Source |
|---|---|---|
| Project name | codemie | pyproject.toml::name |
| Repository/package | codemie | README.md::codemie |
| Project code/key | EPMCDME | scripts/git-hooks/commit_msg.sh::EPMCDME |

## Work Item Tracker

| Field | Value |
|---|---|
| Provider | Jira |
| Key/prefix | EPMCDME |
| Adapter status | configured |
| Adapter instructions | Use the codemie-jira-assistant/Jira workflow when a task requires ticket lookup; commit and MR skills require an `EPMCDME-####` ticket. |

## Ticket Adapter

**Status**: configured
**Adapter**: Invoke the `codemie-jira-assistant` skill via the Skill tool.
**Lookup**: Invoke the `codemie-jira-assistant` skill with the ticket key and a request for summary, description, acceptance criteria, and links.
**Create**: Invoke the `codemie-jira-assistant` skill with the complete ticket payload or approved story file as the argument.
**Output**: Jira ticket key (EPMCDME-<n>) and URL returned by the skill
**Comment**: Invoke the `codemie-jira-assistant` skill with a request to add a comment to the ticket, passing the comment body as the argument.
**Attach**: Invoke the `codemie-jira-assistant` skill with a request to attach the file at {{FILE_PATH}} to ticket {{ID}} under the name `sdlc-<kind>-<filename>`.

## Source Control And Review

| Field | Value |
|---|---|
| Provider | GitLab |
| Repository remote | git@gitbud.epam.com:epm-cdme/codemie.git |
| Default target branch | main |
| Review artifact type | MR |

## MR Adapter

**Status**: configured
**Adapter**: glab (GitLab CLI)
**Instructions**: Branch, commit, push and MR rules from .ai-run/guides/standards/git-workflow.md; MR description from .gitlab/merge_request_templates/Default.md

## Complexity Scoring

**Status**: configured
**Field**: Labels
**Format**: t-shirt

## Lifecycle Intent Handling

### record_complexity_score
Via the `codemie-jira-assistant` skill, add a complexity label to the ticket with the size from `data.complexity_size` (XS–XXL, as-is):
`initial` → `sdlc-factory-est-<SIZE>`, `actual` → `sdlc-factory-act-<SIZE>`. Add only — never remove or change existing labels, so an `sdlc-standard` ticket ends with both labels.
Ticket ID: from the branch name (`EPMCDME-\d+`) or the run work item.

### get_field
For `Labels`: always return `field_value: null` — complexity labels are additive, there is nothing to overwrite.

### artifact_published
Invoke the `codemie-jira-assistant` skill and attach the artifact file using the `--file` flag:

```bash
codemie assistants chat "289d2751-afd9-4c77-a272-90df7cd71702" \
  "Attach this file to Jira ticket EPMCDME-<ID> as the approved <kind> artifact." \
  --file "<path-to-artifact>"
```

Ticket ID: extract from the current branch name (pattern `EPMCDME-\d+`) or from the run work item.
`<path-to-artifact>`: use `data.artifact_path`, or the path to `spec.md` / `plan.md` in the run directory.
For `complexity-assessment.json` (product-owner, `artifact_kind: complexity`), also add `sdlc-factory-est-<size>` from its `size` key via `codemie-jira-assistant`, as in `record_complexity_score`.
