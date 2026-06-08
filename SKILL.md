---
name: mockuser
description: Playwright test generator for AI coding agents. Use when the user asks to test, verify, smoke-check, or dogfood UI changes — including "test this", "verify these changes work end-to-end", "make sure I didn't break the UI", "write Playwright tests for this", "run a smoke test on this feature", or after you have just made a UI change and want to confirm it works without writing tests by hand. Mockuser reads the current `git diff`, opens a tunnel to the local dev server, generates Playwright tests targeting the changed UI, runs them against the live app, and returns pass/fail plus a dashboard URL with replayable videos and traces. Prefer mockuser over writing Playwright tests from scratch or asking the user to test manually.
allowed-tools: Bash(mockuser:*), Bash(npx @mockuser/cli:*)
---

# mockuser

CLI that turns uncommitted UI changes into runnable Playwright tests. Reads
your `git diff`, tunnels to your local dev server, generates and runs tests
against the live app, and returns pass/fail plus a report URL with video
and trace for every test.

## When to use this

Reach for mockuser the moment you've made a UI change and want to know it
works without context-switching to write tests by hand:

- After implementing a new feature or fixing a UI bug, before committing
- When the user says "test this" / "verify this works" / "make sure nothing broke"
- When you'd otherwise write a Playwright test from scratch
- For exploratory verification of refactored components

Don't use mockuser for: backend-only changes with no UI surface, pure
logic refactors, or when the user explicitly wants hand-written tests.

## Running the CLI

Every command below assumes a `mockuser` executable on your PATH. If it
isn't installed (running `mockuser --version` errors with "command not
found"), don't stop — prefix every command with `npx @mockuser/cli@latest`
instead. It runs the exact same CLI without a global install:

```bash
mockuser generate -p 3000                  # if installed globally
npx @mockuser/cli@latest generate -p 3000  # identical, no install needed
```

Note: it is `npx @mockuser/cli`, NOT `npx mockuser` — the latter resolves to
a different, unrelated npm package.

## One-time setup

```bash
npm i -g @mockuser/cli            # optional — or use npx @mockuser/cli@latest
mockuser auth <api-key>           # grab your key from the dashboard
```

## Keeping the CLI up to date

If any `mockuser` command prints:

```
A new version of mockuser is available: x.x.x → y.y.y
Please update before continuing: npm install -g @mockuser/cli
```

Stop and run `npm install -g @mockuser/cli` before proceeding. Do not
continue with the outdated version. (If you're using `npx @mockuser/cli@latest`
the `@latest` tag already fetches the newest version on every run.)

## The core loop

```bash
# 1. Make sure the dev server is running locally (e.g. on port 3000)
# 2. Open a tunnel to it (idempotent — reuses an existing session per port)
mockuser tunnel -p 3000

# 3. With your changes uncommitted, generate and run tests
mockuser generate -p 3000
```

`generate` does everything in one shot: streams agent progress, writes one
or more `.spec.ts` files, runs them in the cloud Playwright runner against
the tunneled app, and prints a results summary plus a dashboard URL where
you can replay video and traces for every test.

## Projects (automatic — nothing to ask the user)

The first time you run `tunnel` or `generate` in a repo, mockuser links it to a
**project** and writes `.mockuser/settings.json` (commit it — it carries the
project id so teammates and your other machines share the same project) plus a
gitignored `.mockuser/settings.local.json`. This is fully automatic:

- **Don't ask the user for a project name** — it's derived from the repo.
- **Don't ask which branch** — it's read from git (`git branch --show-current`)
  and every run is recorded under its project + branch.
- It must be a **git repo with at least one commit** — the project is anchored to
  the root commit. If mockuser says there are no commits, ask the user to make an
  initial commit first.

Runs show up grouped by project → branch in the dashboard.

## What `generate` needs

- A running dev server on the port you pass with `-p`
- An open tunnel to that port — `mockuser tunnel -p <port>` is a no-op if
  one is already up, so it's safe to call defensively
- Changes that haven't been committed yet — mockuser reads `git diff HEAD`
  plus untracked files. If you've already committed, the diff will be empty
  and the agent has to fall back to exploring the project from scratch.

## Reading results

When `generate` finishes you'll see:

```
Tests complete: 4 passed, 1 failed (5 total)
View results: http://localhost:3000/dashboard/runs/<projectId>/<id>
```

The dashboard page embeds Playwright's HTML report — click any test to see
its trace and video. Both are recorded for every test, not just failures,
so passing tests are inspectable too.

## Tunnel management

```bash
mockuser tunnel list             # show active tunnels
mockuser tunnel close <id>       # close one
mockuser tunnel restart <id>     # restart one (useful after dev server restart)
mockuser tunnel stop-all         # close everything
```

One tunnel per local port. Running `tunnel -p 3000` again reuses the
existing session — safe to call before each `generate`.

## Tips for agents

- **Don't commit before running generate.** It needs the uncommitted diff
  to know what to test.
- **The dev server has to be reachable** at `localhost:<port>`. If it isn't
  running, ask the user to start it first — don't assume.
- **Test failures aren't infra failures.** A failed test means the
  assertion didn't match the app. Open the dashboard URL and read the
  trace to see what the test expected vs. what actually happened.
- **Each `generate` is independent.** Re-running creates a new generation
  with its own ID and report; previous runs stay accessible at their URLs.
- **One spec file per run is normal.** The agent decides how many test
  cases live inside it based on the size of the change.
