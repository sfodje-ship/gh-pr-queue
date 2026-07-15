# Agent notes

User-facing behavior (flags, config keys, tiers) lives in [README.md](README.md) — keep it in sync when behavior changes. This file covers only what an agent needs to change the code safely.

## Layout

The whole extension is one executable bash script, `gh-pr-queue`. `tests/self-check.sh` just runs `./gh-pr-queue --self-check`.

## Compatibility constraints

The script targets macOS system bash 3.2. Do not introduce:

- `array+=(elem)` — use `arr[${#arr[@]}]=elem` as the existing code does
- `mapfile` / `readarray`, associative arrays, `&>`, `|&`
- process substitution or pipes into `while read` loops that must mutate variables — the code uses heredocs (`<<EOF ... EOF`) to keep loops in the current shell

Multi-line data is carried in newline-separated string variables (`RECORDS`, `SELECTED_TEAMS`, `TEAMMATES`), not arrays, with the `append_line` / `append_unique_line` / `line_has` helpers.

## Data flow

`main` runs: `preparse_config_path` → source config → `parse_args` → `validate_config` → `load_user` → `load_teams` → `fetch_records` → `render_queue`.

`fetch_records` runs one GitHub search per tier/org (`fetch_direct`, `fetch_team`, `fetch_teammate` — the last ORs `author:` qualifiers into the query, chunked under `QUERY_LIMIT`); each hit goes through `record_pr`, which drops PRs authored by the current user and non-teammate authors in the teammate tier. Surviving PRs append to `RECORDS` as tab-separated lines with fields: source, repo, number, created, age, author, title, url. `filter_approved` then batches all unique PRs into aliased GraphQL queries (50 per call) and removes any whose latest review by the current user is `APPROVED`; with `SKIP_TEAM_APPROVED=1` it also removes PRs actively approved by any name in `TEAMMATES`, unless the current user's latest review is `CHANGES_REQUESTED`.

`render_queue` walks `PRIORITY` tiers in order and dedupes on `repo#number` — a PR renders in the first tier that claims it.

## Testing

Run `./tests/self-check.sh` after every change. `run_self_check` is fully offline: it seeds `RECORDS` directly and stubs `gh` with a temporary shell function to exercise `record_pr`. If you change output format, tier logic, or filtering, update the assertions in `run_self_check` to match. Never add tests that hit the GitHub API.

## Conventions

- Errors go through `die` (exits) or `warn` (continues), both to stderr.
- A new config key needs three touches: a default at the top of the script, validation in `validate_config`, and a line in the `usage` heredoc plus README.
- `shellcheck gh-pr-queue` should stay clean.
