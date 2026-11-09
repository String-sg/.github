# Claude PR auto-review

When `ghostleek` opens a pull request in an opted-in repo, Claude reviews it,
posts one review comment, and opens a separate **fix PR into that PR's branch**
for the findings that are safe to fix. Merge the fix PR to take the fixes;
close it to drop them. Claude never pushes to your branch.

- Reusable workflow: [`.github/workflows/claude-pr-autofix.yml`](../.github/workflows/claude-pr-autofix.yml)
- Each repo opts in with a short caller workflow, `.github/workflows/claude-autofix.yml`

## When it runs

| Event | Runs? |
|---|---|
| PR opened by `ghostleek` (not a draft) | yes |
| Draft marked ready for review | yes |
| `claude-review` label added (by `ghostleek`) | yes, then the label is removed so it can be added again |
| New commits pushed to the PR | no (add `claude-review` to re-run) |
| PR has the `no-autofix` label | no |
| PR by anyone else, including Claude's own fix PRs | no |
| PR from a fork | no |

Model: Opus 5.5 (`claude-opus-5-5`), at most 40 turns, 30-minute job timeout.
A caller can override `model` and `max_turns`.

## What gets fixed, and what is only reported

**Fixed** (in the fix PR): local, unambiguous corrections whose intended
behaviour is clear from the PR itself. That covers bugs, failing checks,
missing guards, tests for changed code, and stale comments or docs.

**Report only** (in the review comment, for you to decide): product behaviour
or copy, UX, security, auth, permissions, crypto or secrets, data model, schema
or migrations, dependency changes, public API contracts, CI and workflow files,
and anything Claude is unsure about.

The prompt sets that boundary. Two mechanical checks back it up, in
`open-fix-pr.sh`, the only route for code to leave the runner:

1. It refuses any change under a protected path: always `.github/` and
   `.claude/`, plus each repo's `protected_paths` (for example `src/db/`
   for schemas).
2. It runs the repo's `verify` commands and refuses to push if any fail.

Claude cannot run `git commit`, `git push`, or any shell command beyond the
read-only `gh`/`git` commands, its repo's verify commands, and that script.

## One-time setup (you do these; Claude never handles the token)

1. **Create a token.** On your machine, run:

   ```bash
   claude setup-token
   ```

   It prints a long-lived OAuth token for your Claude subscription. Runs count
   against your plan's usage limits; there is no separate bill.

2. **Add it to each opted-in repo** as the Actions secret
   `CLAUDE_CODE_OAUTH_TOKEN`. String-sg is on the Free plan, so an org secret
   would not reach private repos; each repo needs its own. `gh` prompts for
   the value once per repo and doesn't echo it:

   ```bash
   for r in ghostleek/string-id ghostleek/stringclaw String-sg/str-skills String-sg/string-sg-landing String-sg/plexo; do gh secret set CLAUDE_CODE_OAUTH_TOKEN -R "$r"; done
   ```

   Check with `gh secret list -R <owner/repo>`. A repo without the secret
   skips the review with a notice on the run; nothing fails.

3. **Install the Claude GitHub app** where it isn't installed yet. It is
   already on all String-sg repos. For `ghostleek/*` repos, check
   <https://github.com/settings/installations>, or install from
   <https://github.com/apps/claude>, and give it at least `string-id` and
   `stringclaw`. The workflow authenticates as `claude[bot]` through this app.
   That makes fix PRs come from `claude[bot]`, so they trigger each repo's
   own CI and never re-trigger the review.

4. **Merge this repo's workflow first, then the caller PRs.** Callers use
   `@main`, so a caller merged before the reusable workflow is on `main`
   fails every PR check with "workflow file issue". The order of caller and
   secret doesn't matter.

## Off switches

From narrowest to widest:

| Scope | How |
|---|---|
| One PR | Add the `no-autofix` label |
| One repo, temporarily | Set the Actions variable `CLAUDE_AUTOFIX` to `off`: `gh variable set CLAUDE_AUTOFIX -R <owner/repo> --body off` (delete it to turn back on) |
| One repo, permanently | Delete `.github/workflows/claude-autofix.yml` |
| One repo, by credentials | Delete its `CLAUDE_CODE_OAUTH_TOKEN` secret: runs skip with a notice |
| Every repo at once | Change the `gate` job's `if:` in the reusable workflow to `false` (callers use `@main`) |

## Adding a repo

Copy a caller from an opted-in repo into `.github/workflows/claude-autofix.yml`.
Set `setup`, `node_version`, `verify` (one command per line, each exactly as
Claude may run it), and any `protected_paths`. Add the secret, and create the
two labels:

```bash
gh label create claude-review -R <owner/repo> --color 7057ff --description "Run the Claude auto-review again"
```

```bash
gh label create no-autofix -R <owner/repo> --color cfd3d7 --description "Skip the Claude auto-review"
```

Inputs: `author` (default `ghostleek`), `node_version` (default `22`), `setup`
(default `npm ci`), `verify` (required), `allowed_bash` (extra Claude
permission patterns), `protected_paths`, `postgres_env` (starts Postgres 17 and
exports its URL under that variable name), `model`, `max_turns`.

## Known limits

- Verification covers what each caller lists, nothing more. Not covered:
  string-id's Playwright e2e, stringclaw's Bun container tests.
- A repo whose CI only runs on PRs into `main` (for example stringclaw) won't
  run CI on a fix PR, because the fix PR targets your feature branch.
  `open-fix-pr.sh` has still run the verify commands, and CI runs on your PR
  once you merge the fix in.
- Old open PRs are not swept. Add `claude-review` to any you want reviewed.
- Verify commands are matched exactly. Don't put double quotes in them.
- Claude Code also loads the repo's `.claude/settings.json`, so hooks and
  permission rules committed there apply in CI too.
