# Tackle Review

**AI code review for Laravel pull requests.** Inline comments with severity
levels, incremental re-reviews on every push, and optional merge gating —
powered by [Laravel Tackle](https://github.com/JordanDalton/laravel-tackle),
running inside your own app.

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

## Acting on review comments: `@tackle fix this`

The `respond` sub-action closes the loop. When a maintainer replies to any
review comment with `@tackle <instruction>`, Tackle applies the change, pushes
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
      contains(github.event.comment.body, '@tackle') &&
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
(`provider`, `model`, `api-key`, prices), plus `trigger` (default `@tackle`)
and `budget`.

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
