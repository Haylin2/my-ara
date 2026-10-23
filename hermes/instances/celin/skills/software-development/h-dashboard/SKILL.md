---
name: h-dashboard
description: Use for any h-dashboard repo work; verify, debug, PR.
version: 1.0.0
author: Celin (Hermes Agent)
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [h-dashboard, laravel, maryui, verification, css]
    category: software-development
    related_skills: [systematic-debugging, github, dogfood]
---

# h-dashboard (Laravel 13 health dashboard, Persian/RTL)

## When to Use

Use whenever a task touches the h-dashboard repo: expert phase-2 verification of issues, root-causing UI/db bugs, implementing fixes or PRs, running the app locally for browser checks, or querying the local database. Skip for unrelated repos.

AGENTS.md in the repo is authoritative for project rules, commands and conventions — this skill only carries what a session must know BEFORE opening the repo, plus field-verified procedures AGENTS.md does not teach. UI/CSS cascade debugging has its own depth file: references/ui-cascade-debug.md.

## Session setup order

1. `cd /home/runner/h-dashboard` (the working copy on this box; forks differ per box — `git remote -v` before assuming owner).
2. Working branch is `celin`, tracks origin/beta. `git status` prints `celin...origin/beta`; for pushes use explicit refspec `HEAD:refs/heads/celin`, never a bare `git push`.
3. Read the issue on `asgarimehdi/h-dashboard` (canonical upstream), not on this box's fork: `gh issue view N -R asgarimehdi/h-dashboard --comments`.
4. Verify the beta tip before starting: `git fetch origin beta && git log origin/beta -1 --format='%H %s'` — re-read late in the session too; other agents merge into beta concurrently.

## Expert (phase-2) verification tasks

Role constraints when dispatched as verifier: verify the finding by measurement, fix NO code, create no PR, add no labels, never close the issue. Deliverable is one expert comment on the issue with: verify verdict + full measurement table, root cause with exact file/class/selector evidence, step-by-step fix plan, required test evidence, and forbidden regressions. If the fix requires a design decision (e.g. which color identity the active state should get), present options with a recommendation — do not decide for the owner.

Treat the issue body's candidate root cause as a hypothesis, never as fact: anchor every claimed mechanism in matched-rule evidence (see references/ui-cascade-debug.md). Past expert premises in this repo were wrong more than once.

Progress reporting: on long multi-step work, give an initial full status report, then a terse Persian status update roughly every 5 minutes until done; labelled sections, no filler, stop reporting once finished.

## Running the app for browser verification

- `php artisan serve --host=0.0.0.0` on :8000 is a SHARED long-running process — never kill or pkill it; reuse it. Other stray `artisan serve --port=8001` orphans are e2e leftovers.
- Login is by `n_code` + password (no email). Seeded admin users: n_code `4411015056`, `4400176134`, `4400176143`, password `12345678` (UsersTableSeeder, non-production only). Users 1–3 are role admin. Throttle: 5 attempts per session token — do not spam the submit button; one fill+click+~6 s wait per attempt, then inspect the page for an error field.
- Login is a Livewire form: fill both fields in the same browser call as the click, wait ≥5 s after clicking, then read `location.pathname`. A failed/lost login shows as /login again with the form still mounted.

## Database access

Use the CLI bridge, not hand-built SQL joins via table guesses:

    php scripts/boost_tool.php db-query '{"query":"SELECT ..."}'   # key is 'query', not 'sql'
    php scripts/boost_tool.php tinker '{"code":"; ...PHP code..."}'

When a join fails on an unknown column, query that table's columns first (`select * from <table> limit 5`) instead of guessing Spatie pivot names.

## Frontend truth chain (for any visual bug report)

1. Read the Blade composition first (which MaryUI components, what attrs passed) — e.g. the sidebar lives in `resources/views/components/layouts/app.blade.php`, `<x-menu activate-by-route>`.
2. Read the vendor component source under `vendor/robsontenorio/mary/src/View/Components/` for the exact classes it emits; then read the built CSS `public/build/assets/*.css` for what actually shipped.
3. Only then reproduce in the browser and record computed styles. See references/ui-cascade-debug.md for the measured tracing recipe (CDP) and the getComputedStyle traps.

Theme mechanism: the app stores `mary-theme` in localStorage and applies it as `data-theme` on `<html>` via an inline script in both app and auth layouts; there is no server-side theme. daisyUI themes configured in `resources/css/app.css` (`@plugin "daisyui"`): light, dark, synthwave, coffee, cupcake, fantasy (`--default`), luxury. Seven active themes is the acceptance-scope for color-contrast work.

## E2E environment rules

- `scripts/e2e-test.sh` is NOT concurrency-safe: never run two runs at once; `.env.dev.bak` may already be swapped after a run. After any e2e run verify `grep DB_DATABASE .env` still says `h_dashboard`.
- If the `.env` backup was lost: rebuild `.env` from `.env.example-github` + secrets in `.env.e2e`, drop the `secrets.*` lines, set `APP_URL=http://127.0.0.1:8000`, `DB_DATABASE=h_dashboard`; then verify `php artisan about --only=environment`. `.env` is gitignored, so this rebuild is a recurring recovery path.
- Kill orphan e2e servers with `kill $(pgrep -f 'artisan serve --port=800[1]')` — never `pkill -f 'artisan serve'` (that kills the shared :8000).
- `.env.e2e` must set `APP_LOCALE=fa` — without it Persian-text specs fail in a batch.
- `parse_ini_file()` on `.env` fails (unquoted parens) — read it with a regex scan or via Laravel `config()`, never `parse_ini_file`.

## MaryUI component pitfalls

- `x-select` defaults to `optionValue='id'`/`optionLabel='name'`. Options keyed `value`/`label` need explicit `option-value="value" option-label="label"`, or every `<option>` renders empty (blank control).
- Pass options from a component method (`:options="$this->myOptions()"`) — a bare `$myOptions` variable is undefined inside the anonymous-class Blade view.

## Boost MCP quirks

- The stdio transport occasionally dies on the FIRST call ('lost its stdio subprocess') — just call it again; the CLI bridge `php scripts/boost_tool.php <tool> '<json>'` always works as fallback.
