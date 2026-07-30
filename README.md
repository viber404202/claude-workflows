# claude-workflows

A central, reusable [Claude Code](https://code.claude.com) GitHub Actions
workflow for your organization. Maintain the automation here once; every repo
across the company consumes it through a thin caller — the `@claude` responder
and the automatic PR reviewer.

> **Before you start:** replace `<YOUR_ORG>` in the examples below with your
> GitHub organization name (the org that owns this `claude-workflows` repo).

## How it works

- [`.github/workflows/claude.yml`](.github/workflows/claude.yml) is a
  [reusable workflow](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
  (`on: workflow_call`) containing all the logic — the `@claude` responder and
  the automatic PR reviewer.
- Each repo adds a ~20-line **caller** workflow that declares the event triggers
  and delegates to this one. Update this repo → all repos get the change (they
  track `@main`).

## Add Claude to a new repo

Create `.github/workflows/claude.yml` in the target repo, replacing
`<YOUR_ORG>` with your organization name:

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
    uses: <YOUR_ORG>/claude-workflows/.github/workflows/claude.yml@main
    secrets: inherit
    permissions:
      contents: read
      pull-requests: write
      issues: read
      id-token: write
      actions: read
```

## Choosing a model

Plain `@claude` uses the default model. To pick a specific model, just extend
the mention — no extra syntax to remember:

| You type | Model used |
| --- | --- |
| `@claude fix this test` | default |
| `@claude-opus fix this test` | Opus |
| `@claude-haiku fix this test` | Haiku |

The same works for on-demand PR reviews: `@claude review` uses the default
model, while `@claude-opus review` (or `-haiku`) reviews with that model.
Type mentions in lowercase (as with plain `@claude`); the automatic review
that runs when a PR is opened always uses the default model.

The workflow passes Claude Code's model *aliases* (`opus`, `haiku`), so each
always resolves to the current model in that family — nothing to update here
when new versions ship.

## Secrets

Each developer needs a long-lived Claude token stored as a secret named
`CLAUDE_TOKEN_<USERNAME>` (GitHub username uppercased, `-` → `_`). For example,
a developer with the username `jane-doe` needs a secret named
`CLAUDE_TOKEN_JANE_DOE`.

> **Note on usernames with hyphens:** GitHub secret names may only contain
> letters, digits, and underscores — hyphens are rejected. Many GitHub
> usernames contain hyphens (e.g. `jane-doe`), so the workflow normalizes the
> username to build the secret name: it uppercases it and replaces every `-`
> with `_`. Create the secret using that normalized name, **not** the raw
> username:
>
> | GitHub username | Secret name to create |
> | --- | --- |
> | `jane-doe` | `CLAUDE_TOKEN_JANE_DOE` |
> | `some-user` | `CLAUDE_TOKEN_SOME_USER` |
> | `johndoe` | `CLAUDE_TOKEN_JOHNDOE` |
>
> This mapping is unambiguous because GitHub usernames cannot contain
> underscores, so two different users can never resolve to the same secret name.
> If a developer's token is missing, the workflow's error message prints the
> exact normalized secret name to create.

Define these once as **organization secrets** (Settings → Secrets and variables
→ Actions) and grant them to all repos; `secrets: inherit` passes them through
to this workflow. Managing them at the org level means a developer's token works
across every repo in the company without per-repo setup.

Generate a token: <https://code.claude.com/docs/en/authentication#generate-a-long-lived-token>

## Versioning

Callers reference `@main` for automatic updates. For stability, cut a tag
(e.g. `v1`) and have callers reference `@v1` instead, then move the tag when you
want to roll out a change.
