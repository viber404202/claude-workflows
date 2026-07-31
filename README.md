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
  # Developer tokens are per-person secrets named CLAUDE_TOKEN_<USERNAME>, so the
  # name is only known at run time. Resolve it here, in the caller, where every
  # org/repo secret is in scope — then the tokens can be passed explicitly and no
  # `secrets: inherit` is needed (linters such as ghalint reject it).
  resolve:
    # Superset of the gates inside the reusable workflow; the called jobs still
    # re-check their own conditions.
    if: |
      github.event_name == 'pull_request' ||
      contains(github.event.comment.body, '@claude') ||
      contains(github.event.review.body, '@claude') ||
      contains(github.event.issue.body, '@claude') ||
      contains(github.event.issue.title, '@claude')
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions: {}
    outputs:
      actor_secret_name: ${{ steps.resolve.outputs.actor_secret_name }}
      pr_author_secret_name: ${{ steps.resolve.outputs.pr_author_secret_name }}
    steps:
      - name: Resolve developer token secret name
        id: resolve
        env:
          ACTOR: ${{ github.actor }}
          PR_AUTHOR: ${{ github.event.pull_request.user.login || github.actor }}
        run: |
          secret_name() {
            printf 'CLAUDE_TOKEN_%s' "$(printf '%s' "$1" | tr '[:lower:]-' '[:upper:]_')"
          }
          echo "actor_secret_name=$(secret_name "$ACTOR")" >> "$GITHUB_OUTPUT"
          echo "pr_author_secret_name=$(secret_name "$PR_AUTHOR")" >> "$GITHUB_OUTPUT"

  claude:
    needs: resolve
    uses: <YOUR_ORG>/claude-workflows/.github/workflows/claude.yml@main
    with:
      # Repo (or org) variable controlling the automatic PR review. Unset means
      # enabled; set it to `false` to stop reviews running on every PR. The
      # reusable workflow interprets the value.
      auto_pr_review: ${{ vars.CLAUDE_AUTO_PR_REVIEW }}
    permissions:
      contents: read
      pull-requests: write
      issues: read
      id-token: write
      actions: read
    secrets:
      actor_token: ${{ secrets[needs.resolve.outputs.actor_secret_name] }}
      pr_author_token: ${{ secrets[needs.resolve.outputs.pr_author_secret_name] }}
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

## Turning the automatic PR review on or off

The `claude-pr-review` job runs **automatically** whenever a PR is opened,
reopened, or marked ready for review. That is the default and needs no
configuration.

To stop it running automatically in a repo, add a repository **variable**
(Settings → Secrets and variables → Actions → *Variables*):

| Variable | Value | Effect |
| --- | --- | --- |
| `CLAUDE_AUTO_PR_REVIEW` | *unset* | Automatic review runs (default) |
| `CLAUDE_AUTO_PR_REVIEW` | `false` | Automatic review is skipped |

`0`, `off`, `no`, and `disabled` work as well, and the match is
case-insensitive. Any other value — including an empty one — leaves the
automatic review enabled, so a typo fails safe towards reviewing.

## Documentation context for PR reviews

PR reviews are grounded in your repo's own docs. The **Prepare Documentation
Context** step gathers them and prepends them to the review prompt, so Claude
reviews against your documented conventions instead of generic best practices.

By default it picks up `README`, `CONTRIBUTING.md`, `ARCHITECTURE.md`,
`CLAUDE.md`, `AGENTS.md`, and everything under `docs/`.

To use a specific set instead, add a repo or org **variable**
`CLAUDE_REVIEW_DOCS` with space- or newline-separated paths or globs:

```
docs/architecture/*.md
CONTRIBUTING.md
```

Context is capped at 256 KB (`MAX_CONTEXT_KB`); extra files are skipped and
noted in the job log. Applies to the PR reviewer only, not the `@claude` job.

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
→ Actions) and grant them to all repos. The caller's `resolve` job works out
which secret name applies to the current run and passes that one token to this
workflow as `actor_token` / `pr_author_token`, so no other secret is exposed to
it. Managing them at the org level means a developer's token works across every
repo in the company without per-repo setup.

Generate a token: <https://code.claude.com/docs/en/authentication#generate-a-long-lived-token>

## Versioning

Callers reference `@main` for automatic updates. For stability, cut a tag
(e.g. `v1`) and have callers reference `@v1` instead, then move the tag when you
want to roll out a change.
