# Tackle Review

**AI code review for Laravel pull requests.** Inline comments with severity
levels, incremental re-reviews on every push, and optional merge gating —
powered by [Laravel Tackle](https://github.com/JordanDalton/laravel-tackle),
running inside your own app.

> 📚 **Full documentation: [tackle.jordandalton.com](https://tackle.jordandalton.com/integrations/review-action)**

## Usage

Your app must have Tackle installed:

```bash
composer require jordandalton/laravel-tackle
```

Then add `.github/workflows/tackle-review.yml`:

```yaml
name: Tackle Review
on:
  pull_request:

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: JordanDalton/tackle-review@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

That's it. Every pull request gets reviewed: findings are posted as inline
review comments anchored to the exact file and line, graded 🔴 Critical /
🟡 Warning / 🟢 Suggestion, under a summary with an overall verdict.

Pushing new commits to the PR triggers a **follow-up review of just the new
changes** — Tackle finds its previous review and doesn't repeat itself.

## Blocking merges

Fail the check when serious findings exist:

```yaml
      - uses: JordanDalton/tackle-review@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          fail-on: critical   # or: warning | suggestion
```

Combine with branch protection to require the check before merging. Leave
`fail-on` unset for a purely advisory review.

## Acting on review comments: `/tackle fix this`

The `respond` sub-action closes the loop. When a maintainer replies to any
review comment with `/tackle <instruction>`, Tackle applies the change, pushes
a commit to the PR branch, and replies in the thread. Questions get answered
in-thread without touching code.

Add `.github/workflows/tackle-respond.yml`:

```yaml
name: Tackle Respond
on:
  pull_request_review_comment:
    types: [created]
  issue_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write

jobs:
  respond:
    if: >
      contains(github.event.comment.body, '/tackle') &&
      contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.comment.author_association)
    runs-on: ubuntu-latest
    steps:
      - uses: JordanDalton/tackle-review/respond@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Safety, enforced twice (workflow `if:` and again inside the action):

- **Only maintainers can trigger it** — comment `author_association` must be
  `OWNER`, `MEMBER`, or `COLLABORATOR` (configurable via `allowed-associations`).
  Comment events fire for *anyone* on public repos; never remove this gate.
- **Fork PRs are refused** — Tackle never pushes to branches in other repos.
- **The reply always arrives** — success (with the pushed SHA), no-op, or a
  clear failure. Threads never dangle.

The `respond` action accepts the same provider inputs as the review action
(`provider`, `model`, `api-key`, prices), plus `trigger` (default `/tackle`)
and `budget`.

## Upgrading dependencies from an issue: `run-type: upgrade`

Tackle can watch your dependencies and turn "a new major is out" into a
reviewed pull request, with humans only at the two decision points.

In the app, schedule the audit (`jordandalton/laravel-tackle` >= 1.27). It
maintains exactly one GitHub issue listing every direct dependency with a new
major and what blocks it — updated in place, closed when nothing remains:

```php
// routes/console.php
Schedule::command('ai:upgrade --audit --issue')->daily();
```

Then add a workflow that fires when a maintainer labels that issue, and runs
the headless upgrade — one isolated session and one PR per package:

```yaml
name: Tackle Upgrade
on:
  issues:
    types: [labeled]

permissions:
  contents: write
  pull-requests: write
  issues: read

concurrency: tackle-upgrade

jobs:
  upgrade:
    if: github.event.label.name == 'tackle-upgrade'
    runs-on: ubuntu-latest
    steps:
      - uses: JordanDalton/tackle-review@v1
        with:
          run-type: upgrade
          packages: pestphp/pest        # or "pkg-one pkg-two" — one PR each
          ref-issue: ${{ github.event.issue.number }}
          budget: '3'
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Each PR carries the plan, the honest verification summary (including what the
test suite did **not** cover), and `Refs #<issue>`. Merge the PRs and the next
scheduled audit closes the issue by itself.

Why the label gate matters: issue bodies are untrusted input on any repo
others can open issues in. Only a maintainer action — a label they applied —
should hand an agent write access. The workflow's `if:` enforces that, and
`concurrency` stops two labels from racing.

**CI on agent-opened PRs:** GitHub's recursion guard means pull requests
created with the workflow's default `GITHUB_TOKEN` do **not** trigger
`pull_request` workflows — the upgrade PR arrives with no test run and no
Tackle review attached. The in-session verification still ran, but if you want
your normal PR checks too, pass a fine-grained PAT (or GitHub App token) with
`contents: write` + `pull-requests: write` as `github-token` instead.

Safety posture, inherited from `ai:upgrade --headless`: composer lifecycle
scripts can never run (enabling them requires an interactive approval that
headless mode cannot give), the budget applies per package, and nothing lands
without a human merging the PR.

## Using other providers

Tackle is built on [`laravel/ai`](https://github.com/laravel/ai) and runs on
any provider it supports with tool calling. Pass `provider`, `model`, and the
generic `api-key` — it's exported as the provider's conventional env var
(`openai` → `OPENAI_API_KEY`, `gemini` → `GEMINI_API_KEY`, …):

```yaml
      - uses: JordanDalton/tackle-review@v1
        with:
          provider: openai
          model: gpt-5.2
          api-key: ${{ secrets.OPENAI_API_KEY }}
          price-input: '1.75'    # per-Mtok rates keep the budget cap accurate
          price-output: '14.00'
```

The provider must exist in your app's `config/ai.php`. If your provider needs
env vars beyond a single API key, set them with a job-level `env:` block —
composite action steps inherit it.

## Inputs

| Input | Default | Description |
|---|---|---|
| `api-key` | *(empty)* | API key, exported as `<PROVIDER>_API_KEY`. Required unless using `anthropic-api-key` or a local provider |
| `provider` | *(app default)* | `laravel/ai` provider name: `anthropic`, `openai`, `gemini`, `groq`, `ollama`, … |
| `model` | *(app default)* | Model to use, e.g. `claude-sonnet-4-6`, `gpt-5.2` |
| `anthropic-api-key` | *(empty)* | Anthropic key shorthand (backwards-compatible) |
| `price-input` | *(app default)* | Input $/Mtok for budget estimation; `0` for local models |
| `price-output` | *(app default)* | Output $/Mtok for budget estimation; `0` for local models |
| `github-token` | `${{ github.token }}` | Token used to fetch the PR and post the review |
| `comment` | `true` | Post findings as inline PR review comments |
| `fail-on` | *(empty)* | Fail on findings at or above: `critical` \| `warning` \| `suggestion` |
| `full` | `false` | Force a full review instead of an incremental follow-up |
| `focus` | *(empty)* | Comma-separated focus areas, e.g. `security,performance` |
| `php-version` | `8.3` | PHP version to set up — match your app's requirement (Laravel 13 apps typically need `8.4`) |
| `working-directory` | `.` | Laravel app directory, for monorepos |
| `pr-number` | *(from event)* | Explicit PR number for dispatch mode — checks out `refs/pull/<n>/head` instead of relying on a `pull_request` event |
| `run-type` | `review` | `review` runs `ai:review`; `task` runs `ai:run` with the `task` prompt; `upgrade` runs `ai:upgrade --headless` for `packages` |
| `task` | *(empty)* | Task prompt for `run-type: task` |
| `packages` | *(empty)* | Packages for `run-type: upgrade`, space- or comma-separated — one headless session and PR each |
| `ref-issue` | *(empty)* | Issue number upgrade PRs reference as `Refs #N` (e.g. the scheduled audit issue) |
| `budget` | *(app default)* | Spend limit in USD, passed as `--budget` on task and upgrade runs (per package for upgrades) |
| `report-url` | *(empty)* | URL to POST the JSON run result to when the run finishes (see Tackle Cloud below) |
| `report-token` | *(empty)* | Bearer token for the `Authorization` header when posting to `report-url` |

The `respond` sub-action also accepts `report-url` / `report-token` and reports
its result the same way.

## Tackle Cloud / repository_dispatch

The action can also be driven remotely — e.g. by Tackle Cloud, a control plane
that triggers runs via `repository_dispatch` and collects the results. In this mode the PR number, run type, and reporting endpoint arrive in
the event's `client_payload` instead of a `pull_request` event:

```yaml
name: Tackle Cloud
on:
  repository_dispatch:
    types: [tackle-run]

permissions:
  contents: read
  pull-requests: write

jobs:
  tackle:
    runs-on: ubuntu-latest
    steps:
      - uses: JordanDalton/tackle-review@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          pr-number: ${{ github.event.client_payload.pr_number }}
          run-type: ${{ github.event.client_payload.type }}
          task: ${{ github.event.client_payload.task }}
          fail-on: ${{ github.event.client_payload.fail_on }}
          focus: ${{ github.event.client_payload.focus }}
          full: ${{ github.event.client_payload.full }}
          budget: ${{ github.event.client_payload.budget }}
          report-url: ${{ github.event.client_payload.ingest_url }}
          report-token: ${{ github.event.client_payload.ingest_token }}
```

How dispatch mode differs from the normal `pull_request` flow:

- **`pr-number` replaces the event context.** The action checks out
  `refs/pull/<pr-number>/head` (full history) and reviews that PR, so it works
  from any event type.
- **`run-type` selects the command.** `review` (the default) runs `ai:review`;
  `task` runs `php artisan ai:run "<task>" --output=json --allowlist`
  with the optional `--budget` cap — an autonomous task run instead of a review.
- **Results are reported back.** The command runs in JSON mode and writes its
  result to `tackle-result.json`; when `report-url` is set the file is POSTed
  there with `Authorization: Bearer <report-token>` after the run — even when
  the run fails. If no result was produced, an error stub
  (`{"ok":false,"outcome":"error","error":"no result produced"}`) is sent
  instead. A failed POST only logs a warning; it never fails the workflow.
- **The check-run verdict is preserved.** The action exits with the command's
  original exit code after reporting, so `fail-on` gating still marks the
  check red.

## How it works

The action checks out the PR, sets up PHP, installs your app's Composer
dependencies (with caching), and runs:

```bash
php artisan ai:review --pr=<number> --comment [--fail-on=…]
```

The review agent is **read-only by construction** — its tools can read, glob,
and search your code, nothing else. It reads the full files around the changes
for context before commenting, and every inline comment is validated against
the diff before posting. Cost is roughly $0.05–0.30 per PR against your
Anthropic key, with a hard per-run budget cap enforced by Tackle.

## Requirements

- The repository is a Laravel app with `jordandalton/laravel-tackle` installed
- An API key secret for your chosen provider (only local providers like Ollama
  run without one)
- `permissions: pull-requests: write` in the workflow (to post comments)

## License

MIT
