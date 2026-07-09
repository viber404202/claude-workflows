# claude-workflows

Central, org-wide [Claude Code](https://code.claude.com) GitHub Actions workflow
for the **viber404202** organization. Maintain the automation here once; every
repo consumes it through a thin caller.

## How it works

- [`.github/workflows/claude.yml`](.github/workflows/claude.yml) is a
  [reusable workflow](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
  (`on: workflow_call`) containing all the logic — the `@claude` responder and
  the automatic PR reviewer.
- Each repo adds a ~20-line **caller** workflow that declares the event triggers
  and delegates to this one. Update this repo → all repos get the change (they
  track `@main`).

## Add Claude to a new repo

Create `.github/workflows/claude.yml` in the target repo:

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]
  pull_request:
    types: [opened, reopened, ready_for_review]

jobs:
  claude:
    uses: viber404202/claude-workflows/.github/workflows/claude.yml@main
    secrets: inherit
    permissions:
      contents: read
      pull-requests: write
      issues: read
      id-token: write
      actions: read
```

## Secrets

Each developer needs a long-lived Claude token stored as a secret named
`CLAUDE_TOKEN_<USERNAME>` (username uppercased, `-` → `_`). Define these once as
**organization secrets** (Settings → Secrets and variables → Actions) and grant
them to all repos; `secrets: inherit` passes them through to this workflow.
Generate a token: <https://code.claude.com/docs/en/authentication#generate-a-long-lived-token>

## Versioning

Callers reference `@main` for automatic updates. For stability, cut a tag
(e.g. `v1`) and have callers reference `@v1` instead, then move the tag when you
want to roll out a change.
