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

## Inputs

| Input | Default | Description |
|---|---|---|
| `anthropic-api-key` | *(required)* | Anthropic API key for the review agent |
| `github-token` | `${{ github.token }}` | Token used to fetch the PR and post the review |
| `comment` | `true` | Post findings as inline PR review comments |
| `fail-on` | *(empty)* | Fail on findings at or above: `critical` \| `warning` \| `suggestion` |
| `full` | `false` | Force a full review instead of an incremental follow-up |
| `focus` | *(empty)* | Comma-separated focus areas, e.g. `security,performance` |
| `php-version` | `8.3` | PHP version to set up |
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
- An `ANTHROPIC_API_KEY` repository secret
- `permissions: pull-requests: write` in the workflow (to post comments)

## License

MIT
