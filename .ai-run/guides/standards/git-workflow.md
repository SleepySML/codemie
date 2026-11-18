# Git Workflow

## Branch Naming Convention
- base branch: main

Use ticket-first feature branches for project work.

| Avoid | Prefer |
|---|---|
| `feature/random-description` when a Jira task exists | `EPMCDME-12345_short-description` |
| Long or vague branch descriptions | Short kebab-case or snake-compatible summary after the ticket |

Evidence: README.md § Development Workflow and CONTRIBUTING.md § How to Contribute name `<TICKET-ID>_short-description`; remote branches show both that form and `feature/epmcdme-<n>`, and no hook in `.pre-commit-config.yaml` checks the branch name.

## Commit Message Format

Use `EPMCDME-####: Description` for repository commits.

```
EPMCDME-<number>: <Description>
```

Examples: `EPMCDME-16609: Show request cost, cache writes and tool success in CLI Session Detail (with EPMCDME-16741)`, `EPMCDME-14855: Fix agent streaming truncation, tool-error cap, redaction and A2A timeout`, `EPMCDME-13429: Record the default budget assignment when an admin unassigns a custom budget`

| Avoid | Prefer |
|---|---|
| `fix: update thing` | `EPMCDME-12345: Update thing` |
| `EPMCDME-12345 update thing` | `EPMCDME-12345: Update thing` |

Evidence: `scripts/git-hooks/commit_msg.sh` (the `codemie-commit-msg` hook) rejects a subject that does not match `^EPMCDME-[0-9]+:` and lets `Merge `, `Revert `, `fixup!`, `squash!` and `amend!` subjects through; `git log --oneline -200` shows that prefix on each subject.

## Merge Strategy

Default to squash merge for review artifacts unless the reviewer or maintainer asks for a different strategy. Keep the final squash commit in the ticket-prefixed format.

| Avoid | Prefer |
|---|---|
| Merging a noisy WIP commit stack | One reviewed squash commit with the Jira ticket prefix |
| Changing merge strategy without review context | Follow the MR reviewer or maintainer instruction |

Evidence: README.md § Development Workflow asks for at least one approval and a green CI pipeline; `git log --merges` on `main` is empty, so the history is linear. No repository setting in this checkout states the merge method.

## Git Side Effects

Only commit, push, branch, or create MRs when the user explicitly asks for git operations.

| Avoid | Prefer |
|---|---|
| Proactive commits after editing files | Report changed files and wait unless git was requested |
| Committing without a work item | Ask for the `EPMCDME-####` ticket first |

```bash
glab mr create \
  --title "EPMCDME-XXXX: Capital sentence" \
  --target-branch main \
  --remove-source-branch \
  --draft \
  --yes
```

Evidence: user policy carried by the entrypoint's Git Operations rule; the MR command shape follows `.ai-run/guides/project.md` MR Adapter.

## MR Description

Every MR description follows the default GitLab template,
[`.gitlab/merge_request_templates/Default.md`](../../../.gitlab/merge_request_templates/Default.md). It is the single
source for the sections and the rules for filling them; the instructions live in its comments, so do not restate them
here. The web UI loads it automatically. With `glab`, pass the filled template via `--description`.

## Inline MR Comments via glab

`glab mr note` = general comment only. For diff-anchored inline comments use the discussions API.

**Step 1 — get SHAs** (latest version = index 0):
```bash
glab api "projects/epm-cdme%2Fcodemie/merge_requests/<MR_ID>/versions" \
  | python3 -c "import sys,json; v=json.load(sys.stdin)[0]; print(v['base_commit_sha'], v['head_commit_sha'], v['start_commit_sha'])"
```

**Step 2 — find line number**: from diff hunk `@@ -OLD +NEW_START @@`, count non-`-` lines to target.

**Step 3 — post** (must use `--input` + `Content-Type`; `--field "position[...]"` silently drops nested objects):
```bash
cat > /tmp/comment.json << 'EOF'
{
  "body": "comment text",
  "position": {
    "base_sha": "<base>", "head_sha": "<head>", "start_sha": "<start>",
    "new_path": "src/path/to/file.py", "old_path": "src/path/to/file.py",
    "position_type": "text", "new_line": 123
  }
}
EOF
glab api "projects/epm-cdme%2Fcodemie/merge_requests/<MR_ID>/discussions" \
  --method POST -H "Content-Type: application/json" --input /tmp/comment.json
```

For added lines (`+`): set `new_line` only. Removed lines (`-`): set `old_line` only. Context: set both.

If response `notes[0].position` is `null` — position not resolved, fell back to general comment. Delete with:
```bash
glab api "projects/epm-cdme%2Fcodemie/merge_requests/<MR_ID>/notes/<NOTE_ID>" --method DELETE
```

## Anti-Patterns

| Do not | Do instead | Why |
|---|---|---|
| `git commit --no-verify` or `git push --no-verify` to get past a failing hook | Fix what the hook reports (reword the subject to `EPMCDME-<n>:`, stage the files `ruff` rewrote) and re-run the command | README.md § Git Hooks marks skipping as not recommended; `scripts/git-hooks/commit_msg.sh` exits 1 on a subject without the `EPMCDME-[0-9]+:` prefix, and the Alembic heads check the pre-push hook runs is repeated in the MR pipeline for MRs targeting `main` |

## Troubleshooting

| Issue | Fix |
|---|---|
| Missing Jira ticket for commit | Ask for the `EPMCDME-####` work item before committing |
| Branch created from the wrong base | Stop and ask before rebasing or recreating branches |
| Existing unrelated changes | Preserve them; do not revert user work |
| Commit hook changes files | Review and stage only intended files before retrying |
| Inline comment posts as general note | Use `--input` + `-H "Content-Type: application/json"`; check `notes[0].position` in response |
