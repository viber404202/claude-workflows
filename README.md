# dp-claude-workflows

The central, reusable [Claude Code](https://code.claude.com) GitHub Actions
workflow for our engineering repos. The automation is maintained here once; repos
across all of our GitHub organizations consume it through a thin caller.

You get an automatic PR reviewer, plus an on-demand `@claude` responder where
**you say what you want** — `review`, `fix`, or `ask` — instead of hoping Claude
infers it.

> **Before you start:** in the examples below, replace `<ORG>` with the GitHub
> organization that owns *this* repo — not the org of the repo you are adding the
> caller to. A repo in any of our organizations can call this workflow, as long as
> Actions access is enabled for it (see [Cross-organization use](#cross-organization-use)).

## How it works

- [`.github/workflows/claude.yml`](.github/workflows/claude.yml) is a
  [reusable workflow](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
  (`on: workflow_call`) containing all the logic.
- Each repo adds a small **caller** workflow (~70 lines, mostly comments) that
  declares the event triggers and delegates to this one. Update this repo → all
  repos get the change (they track `@main`).
- A single `dispatch` job parses the mention into **exactly one mode**, and every
  downstream job is gated on that mode. One mention can never start two Claude
  runs.

```
event  ─→  resolve (caller: which CLAUDE_TOKEN_* secret?)
             └─→  dispatch  ─→  mode = review | fix | ask | help | none
                                 ├─ review → claude-review   (read-only)
                                 ├─ fix    → claude-fix      (can commit)
                                 ├─ ask    → claude-ask      (read-only)
                                 ├─ help   → claude-help     (no Claude run)
                                 └─ none   → nothing runs
```

Because `mode` is a single value, the jobs are mutually exclusive by
construction — there is no combination of comment text that starts two of them.

## Commands — telling Claude what to do

Say what you want after the mention. There are three commands:

| You type | What happens | Can it change code? |
| --- | --- | --- |
| `@claude review` | Reviews the PR against your repo's docs, leaves inline comments plus a summary | No |
| `@claude fix <description>` | Implements the change and commits it | **Yes** |
| `@claude ask <question>` | Investigates and answers in a comment | No |
| `@claude <anything else>` | Treated as `ask` | No |
| `@claude` on its own | Replies with this command list | No |

**Nothing but a mention gets you the menu.** `@claude` — or `@claude please`,
or any mention followed only by filler — posts the table above as a comment so
you can see the options without leaving the PR. No Claude run is started and no
developer token is needed, so it costs nothing to ask.

**A mention with a question never writes anything.** `@claude the auth test is
broken` gets you an explanation of the cause, not a commit. Landing a change
always requires typing `fix` explicitly, so nothing surprising appears on your
branch.

Conversational filler between the mention and the command is ignored, so all of
these work as you'd expect:

```
@claude please review this
@claude can you fix the flaky auth test
@claude go ahead and fix the lint error
@claude just review it
```

Recognized aliases: `fix` also accepts `implement` and `apply`; `ask` also
accepts `explain`, `question`, and `answer`; `review` also accepts `reviews`.
Matching is case-insensitive.

The command must be the **first non-filler word** after the mention. This is
deliberate — it means `@claude why did the review fail?` is an `ask` (a
question about a review), not a request to run one.

### Where `fix` commits

This is the action's behavior, not something the workflow chooses:

| Triggered from | Result |
| --- | --- |
| A comment on an **open PR** | Commits straight to that PR's branch |
| An **issue** | Creates a new timestamped branch, pushes there, and comments with a pre-filled PR link for you to open |

So `@claude fix` on an issue never writes to your default branch, and branch
protection rules are respected either way.

### Enforcement, not just instructions

Each mode is constrained at the GitHub-permissions level, so a mode cannot do
another mode's job even if the prompt is subverted by a crafted comment:

| Job | `contents` | Editing tools |
| --- | --- | --- |
| `claude-help` | not granted | no Claude run at all |
| `claude-ask` | `read` | blocked (`--disallowedTools`) |
| `claude-review` | `read` | not granted |
| `claude-fix` | `write` | allowed |

The mode instructions given to Claude are guidance; the permissions are the
actual guarantee. `claude-ask` has no write access to push with, regardless of
what a comment asks it to do.

> **Note:** anyone who can comment on a PR can invoke `@claude fix`, and the
> comment text becomes part of Claude's prompt. The action's own actor checks
> gate this to users with write permission by default — do not set
> `allowed_non_write_users` on a public repo unless you have thought that
> through.

## Choosing a model

Plain `@claude` uses the default model. To pick a specific model, just extend the
mention — no extra syntax to remember. Model selection **composes with every
command**:

| You type | Command | Model used |
| --- | --- | --- |
| `@claude fix this test` | fix | default |
| `@claude-opus fix this test` | fix | Opus |
| `@claude-haiku review` | review | Haiku |
| `@claude-opus what does this do?` | ask | Opus |

Mentions are matched case-insensitively, and the longest one wins — so
`@claude-opus` is never mis-read as a bare `@claude`. If a comment contains more
than one form, the more specific one takes effect. The automatic review that runs
when a PR is opened always uses the default model, since there is no mention to
read.

The workflow passes Claude Code's model *aliases* (`opus`, `haiku`), so each
always resolves to the current model in that family — nothing to update here when
new versions ship.

## What runs when

| Event | Text | Mode | Job | Token used |
| --- | --- | --- | --- | --- |
| `pull_request` (opened, reopened, ready_for_review) | *n/a* | `review` | `claude-review` | PR **author** |
| `issue_comment` on a PR | `@claude review` | `review` | `claude-review` | commenter |
| `issue_comment` on a PR | `@claude fix …` | `fix` | `claude-fix` | commenter |
| `issue_comment` | `@claude` alone | `help` | `claude-help` | none — uses `GITHUB_TOKEN` |
| `issue_comment` | anything else with `@claude` | `ask` | `claude-ask` | commenter |
| `pull_request_review_comment` | `@claude …` | per command | per mode | commenter |
| `pull_request_review` (submitted) | `@claude …` in the review body | per command | per mode | commenter |
| `issues` (opened, assigned) | `@claude` in title or body | per command | per mode | actor |
| any | mention posted by a **bot** | `none` | *nothing runs* | — |
| any | no `@claude` mention | `none` | *nothing runs* | — |

Notes on the token column:

- `claude-review` reads the `pr_author_token` secret and the other two read
  `actor_token`. On a **comment** event there is no `pull_request` payload, so the
  caller's `pr_author_secret_name` falls back to `github.actor` — which is why a
  comment-triggered review bills to the commenter, while the automatic review on
  PR-open bills to the PR author.
- When `CLAUDE_AUTO_PR_REVIEW` is disabled, a `pull_request` event resolves to
  `none`: `dispatch` still runs and logs the decision, but no Claude job does.
- Mentions written by an app (`sender.type == 'Bot'`) resolve to `none`, so the
  workflow can never answer its own comments — the `help` reply quotes every
  command, and a review summary suggests `@claude fix …`. The check sits after
  the `pull_request` branch, so PRs opened by bots such as Dependabot are still
  reviewed automatically.

## Add Claude to a new repo

Create `.github/workflows/claude.yml` in the target repo, replacing `<ORG>` with
the organization that owns this repo. Everything else is copied as-is — the caller
is identical in every repo:

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
    # Cheap pre-filter so the workflow doesn't spin up on every comment. The
    # reusable workflow's `dispatch` job does the real parsing.
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
    uses: <ORG>/dp-claude-workflows/.github/workflows/claude.yml@main
    with:
      # Repo (or org) variable controlling the automatic PR review. Unset means
      # enabled; set it to `false` to stop reviews running on every PR. The
      # reusable workflow interprets the value.
      auto_pr_review: ${{ vars.CLAUDE_AUTO_PR_REVIEW }}
    # Ceiling for the called workflow — its jobs can never hold more than this,
    # so `contents`/`issues` must be `write` for `@claude fix` to commit and for
    # Claude to post comments. Each job in the reusable workflow narrows this
    # further (e.g. `@claude ask` runs with `contents: read`).
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    secrets:
      actor_token: ${{ secrets[needs.resolve.outputs.actor_secret_name] }}
      pr_author_token: ${{ secrets[needs.resolve.outputs.pr_author_secret_name] }}
```

## Configuration

Both are repository or organization **variables** (Settings → Secrets and
variables → Actions → *Variables*), not secrets:

| Variable | Default | Effect |
| --- | --- | --- |
| `CLAUDE_AUTO_PR_REVIEW` | unset (enabled) | Set to `false` to stop reviewing every PR automatically |
| `CLAUDE_REVIEW_DOCS` | unset (auto-discover) | Space- or newline-separated paths/globs to use as review context |

### Turning the automatic PR review on or off

The `claude-review` job runs **automatically** whenever a PR is opened, reopened,
or marked ready for review. That is the default and needs no configuration.
Disabling it does not affect on-demand `@claude review`.

| Variable | Value | Effect |
| --- | --- | --- |
| `CLAUDE_AUTO_PR_REVIEW` | *unset* | Automatic review runs (default) |
| `CLAUDE_AUTO_PR_REVIEW` | `false` | Automatic review is skipped |

`0`, `off`, `no`, and `disabled` work as well, and the match is
case-insensitive. Any other value — including an empty one — leaves the automatic
review enabled, so a typo fails safe towards reviewing.

### Documentation context for PR reviews

PR reviews are grounded in your repo's own docs. The **Prepare documentation
context** step gathers them and prepends them to the review prompt, so Claude
reviews against your documented conventions instead of generic best practices.

By default it picks up `README`, `CONTRIBUTING.md`, `ARCHITECTURE.md`,
`CLAUDE.md`, `AGENTS.md`, and everything under `docs/`.

To use a specific set instead, set `CLAUDE_REVIEW_DOCS` to space- or
newline-separated paths or globs:

```
docs/architecture/*.md
CONTRIBUTING.md
```

Context is capped at 256 KB (`MAX_CONTEXT_KB`); extra files are skipped and noted
in the job log. Applies to `review` only — the `ask` and `fix` modes read
whatever they need from the checked-out repo directly.

## Secrets

Each developer needs a long-lived Claude token stored as a secret named
`CLAUDE_TOKEN_<USERNAME>` (GitHub username uppercased, `-` → `_`). For example, a
developer with the username `jane-doe` needs a secret named
`CLAUDE_TOKEN_JANE_DOE`.

> **Note on usernames with hyphens:** GitHub secret names may only contain
> letters, digits, and underscores — hyphens are rejected. Many GitHub usernames
> contain hyphens (e.g. `jane-doe`), so the workflow normalizes the username to
> build the secret name: it uppercases it and replaces every `-` with `_`. Create
> the secret using that normalized name, **not** the raw username:
>
> | GitHub username | Secret name to create |
> | --- | --- |
> | `jane-doe` | `CLAUDE_TOKEN_JANE_DOE` |
> | `some-user` | `CLAUDE_TOKEN_SOME_USER` |
> | `johndoe` | `CLAUDE_TOKEN_JOHNDOE` |
>
> This mapping is unambiguous because GitHub usernames cannot contain
> underscores, so two different users can never resolve to the same secret name.
> If a developer's token is missing, the workflow's error message prints the exact
> normalized secret name to create.

Define these once as **organization secrets** (Settings → Secrets and variables →
Actions) and grant them to all repos. The caller's `resolve` job works out which
secret name applies to the current run and passes that one token to this workflow
as `actor_token` / `pr_author_token`, so no other secret is exposed to it.
Managing them at the org level means a developer's token works across every repo
in that organization without per-repo setup.

Secrets are **not** shared between GitHub organizations. Because we have several,
a developer who works in more than one needs their `CLAUDE_TOKEN_<USERNAME>`
secret defined in each org whose repos they use Claude from — the same token value
is fine.

Generate a token: <https://code.claude.com/docs/en/authentication#generate-a-long-lived-token>

## Reusable workflow interface

What this workflow accepts from a caller:

| Kind | Name | Required | Purpose |
| --- | --- | --- | --- |
| input | `auto_pr_review` | No (default `'true'`) | Falsy value disables the automatic PR review |
| secret | `actor_token` | No | Token for `github.actor`; used by `ask` and `fix` |
| secret | `pr_author_token` | No | Token for the PR author; used by `review` |

Both secrets are declared optional on purpose: a missing token then produces this
workflow's actionable "add a secret named `CLAUDE_TOKEN_…`" error instead of an
opaque caller-side failure.

## Versioning

Callers reference `@main` for automatic updates. For stability, cut a tag (e.g.
`v1`) and have callers reference `@v1` instead, then move the tag when you want
to roll out a change.

The `anthropics/claude-code-action` version is pinned inside this workflow (all
three jobs use the same pin), so bumping the action is a one-line change here that
rolls out everywhere.
