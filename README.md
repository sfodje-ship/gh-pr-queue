# gh-pr-queue

A [gh CLI](https://cli.github.com/) extension that lists open pull requests waiting on your review, grouped into priority tiers and stripped of noise: PRs you authored, PRs you already approved, and duplicates across tiers are removed.

## Install

```sh
gh extension install sfodje-ship/gh-pr-queue
```

Requires an authenticated `gh` (`gh auth login`).

## Usage

```sh
gh pr-queue [options]
```

| Option | Effect |
|---|---|
| `--drafts` | Include draft pull requests |
| `--all` | Disable the created-date cutoff (default: last `CUTOFF_DAYS` days) |
| `--verbose` | Show age, author, and title per PR instead of the compact table |
| `--org ORG` | Limit to an organization; repeatable |
| `--team ORG/TEAM` | Limit to a team and include teammate-authored PRs; repeatable |
| `--config PATH` | Source an alternate config file |
| `--self-check` | Run the offline self-test and exit |

Default output is a compact table of PR URLs and authors. `--verbose` trades the table for one block per PR with age, author, and title.

## Priority tiers

Each PR lands in the first tier that matches, oldest first within a tier:

- **direct** — your review was requested by name (`user-review-requested:@me`).
- **teammate** — authored by a member of a team named in `TEAMMATE_TEAMS`, or of a team selected with `--team` / `TEAMS_INCLUDE`. Teams larger than `TEAM_SIZE_GUARD` members are not expanded (a warning explains how to include them). `TEAMMATE_TEAMS` only picks whose PRs count as teammate-authored — unlike `TEAMS_INCLUDE`, it does not narrow which team review requests are shown.
- **team** — review requested from one of your teams.

`PRIORITY` controls tier order. Space-separated tokens are ranked tiers; `+` merges tokens into one tier. The default is `"direct team"`; setting `TEAMMATE_TEAMS` switches it to `"direct teammate team"`, and passing `--team` switches it to `"direct+teammate team"` — unless you set your own.

## Configuration

The config file is plain shell `KEY=VALUE` syntax, read from the first of:

1. `--config PATH`
2. `$GH_PR_QUEUE_CONFIG`
3. `~/.config/gh-pr-queue/config`

Command-line flags override config file values. All keys are optional.

| Key | Default | Effect |
|---|---|---|
| `CUTOFF_DAYS` | `14` | Ignore PRs created more than N days ago. Disable per run with `--all`. |
| `ORGS` | _(all orgs)_ | Comma-separated organizations to search; everything else is ignored. Same as repeating `--org`. |
| `TEAMS_INCLUDE` | _(all your teams)_ | Comma-separated `ORG/TEAM` slugs. Only these teams' review requests are searched, and their members count as teammates. Same as repeating `--team`. |
| `TEAMS_EXCLUDE` | _(none)_ | Comma-separated `ORG/TEAM` slugs whose review requests are never shown. |
| `TEAMMATE_TEAMS` | _(none)_ | Comma-separated `ORG/TEAM` slugs whose members' PRs count as teammate-authored. Unlike `TEAMS_INCLUDE`, does not narrow which team review requests are shown — use this to float your immediate team's PRs to the top while still seeing every team request. |
| `TEAM_SIZE_GUARD` | `25` | Teams with more members than this are not expanded into teammates (guards against org-wide umbrella teams). A warning names any team it skips. |
| `PRIORITY` | `direct team` | Tier order as space-separated tokens (`direct`, `teammate`, `team`); `+` merges tokens into one tier — see [Priority tiers](#priority-tiers). |

Example — see all team requests, but float your own team's PRs to the top:

```sh
# ~/.config/gh-pr-queue/config
TEAMMATE_TEAMS=acme/platform
```

Example — restrict everything to one org and one team, with a tighter cutoff:

```sh
# ~/.config/gh-pr-queue/config
ORGS=acme
TEAMS_INCLUDE=acme/platform
CUTOFF_DAYS=7
```

## Limits

GitHub search caps results at 1000 per query and may return incomplete results under load; the tool warns when either happens. Narrow the scope with `--org`, a shorter `CUTOFF_DAYS`, or a tighter team config.

## Testing

```sh
./tests/self-check.sh
```

Runs entirely offline — no GitHub API calls.
