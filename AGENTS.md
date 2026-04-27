# AGENTS.md

Guidance for AI coding agents working in this repository.

## Repository purpose

This is the **GitHub profile README repo** for user `D3strukt0r` (repo `D3strukt0r/D3strukt0r`). Its `README.md` is rendered on the user's profile page at github.com/D3strukt0r. There is no application code, no build, no tests — only Markdown, YAML workflows, and two generated SVG cards. Changes are typically content tweaks to `README.md` or workflow/config adjustments.

## Architecture — how the profile stays updated

Three independent daily automations keep the profile fresh. Understanding each is essential before editing anything they touch.

### 1. Activity list (`.github/workflows/update-readme.yml`)

Runs daily via `jamesgeorge007/github-activity-readme@master`. The action reads `README.md`, locates the marker pair

```
<!--START_SECTION:activity-->
<!--END_SECTION:activity-->
```

fetches the user's public events via the GitHub API, filters to `IssueCommentEvent,IssuesEvent,PullRequestEvent,ReleaseEvent`, and splices up to 5 formatted entries **between** the markers. Then it commits with message `docs(readme): update profile` and pushes directly to `master`.

A follow-up "Squash into previous update commit if applicable" step soft-resets `HEAD~2` and re-commits whenever both `HEAD` and `HEAD~1` share the `docs(readme): update profile` subject, then `git push --force-with-lease`. This collapses consecutive bot commits into one rolling update commit so the history stays clean.

Key constraints:
- The markers MUST remain in `README.md` and MUST have a newline between them (or lines with content the action will overwrite). Removing either marker fails the workflow with "Couldn't find the …comment. Exiting!".
- Inputs to the action (`COMMIT_MSG`, `EMPTY_COMMIT_MSG`, etc.) must go under `with:`, not `env:` — the action reads them via `core.getInput()`. A past bug had them under `env:` and the defaults were silently used.
- The action's error messages only include stdout, never stderr. Cryptic `Error: Exit code: 1` with no detail almost always means `git push` was rejected (check branch protection).
- Both checkout and the action authenticate with `secrets.GH_PAT` so pushes are attributed to a user with Maintain+ role, satisfying the ruleset's force-push bypass (see "Branch protection" below).

### 2. Stats cards (`.github/workflows/grs.yml`)

Runs daily via `readme-tools/github-readme-stats-action@v1` (a thin wrapper around `anuraghazra/github-readme-stats`). Regenerates two SVGs committed under `profile/`:

- `profile/stats.svg` — overall stats card
- `profile/top-langs.svg` — top languages card (compact layout)

`README.md` references them as `./profile/stats.svg` and `./profile/top-langs.svg`. Commit message: `docs(readme): update profile` — intentionally identical to the activity workflow so both can squash against each other.

The `Commit cards` step performs the same squash logic inline: if `HEAD~1` already has the update subject, it soft-resets `HEAD~2`, recommits, and force-pushes; otherwise a normal push.

Requires a repo secret `GH_PAT` — used both as the API token for anuraghazra's stats backend AND as the checkout token so the resulting `git push --force-with-lease` bypasses the non-fast-forward rule. The default `github.token` doesn't work for either purpose: the API needs a user-scoped token, and the Actions bot can't bypass the ruleset.

### 3. Badge stack (static — manually edited in `README.md`)

Lines 3–34 of `README.md` are hand-written shields.io badges in `for-the-badge` style. When adding a tech badge, follow the pattern:

```markdown
[![Name](https://img.shields.io/badge/Label-HEX?style=for-the-badge&logo=<simple-icons-slug>&logoColor=white)](https://link)
```

Use exact [simple-icons slugs](https://github.com/simple-icons/simple-icons/blob/master/slugs.md) — e.g. `nodedotjs` not `node`, `gnubash` not `bash`, `visualstudiocode` not `vscode`. Brand hex colors from simple-icons. For techs without a simple-icons slug (like generic "SQL"), omit the `logo=` param entirely.

The top Twitter follower badge (line 3) is a live-count shield and is intentionally a different style from the static stack — do not touch unless asked.

## Auxiliary workflows

- `dependabot-automerge.yml` — auto-approves and enables auto-merge on Dependabot PRs for minor/patch bumps only. Major bumps require manual review.
- `dependabot-validate.yml` — validates `.github/dependabot.yml` on PR.
- `greetings.yml` — first-time contributor messages.
- `label.yml` — applies labels from `.github/labeler.yml` to PRs by branch/path rules.
- `stale.yml` — marks stale issues/PRs; only labels `awaiting-feedback,awaiting-answers`.

## Branch protection / rulesets

Two ruleset JSONs live at the repo root (`_master_ no force push.json`, `_master_ not deletable.json`). These are GitHub Ruleset exports meant to be imported via **Settings → Rules → Rulesets → Import**. They apply:

- `non_fast_forward` on `~DEFAULT_BRANCH` (blocks `git push --force`)
- `code_quality` (severity: errors) on `~DEFAULT_BRANCH`
- `deletion` block on `~DEFAULT_BRANCH`

`bypass_actors` includes the `Maintain` repository role. The update workflows force-push during squash, so they authenticate via `GH_PAT` (a PAT belonging to the repo admin, which is ≥ Maintain) — the default `github.token` represents `github-actions[bot]` and does NOT inherit role-based bypass, so it would be rejected with a silent `Exit code: 1`.

If you change the workflows to push without `GH_PAT`, either re-grant bypass to the new actor or remove the squash/force-push logic.

## Known config mismatches

`.github/dependabot.yml` has `target-branch: develop` but this repo does **not** have a `develop` branch (only `master`). Dependabot will either silently no-op or error on that config. Either create `develop`, or change the target to `master`.

`dependabot-validate.yml` triggers on pushes to `[master, develop]` — same mismatch, but harmless since the push simply never happens on the nonexistent branch.

## Conventions

- **Commit messages**: Conventional Commits. Automation uses `docs(readme):` for README-facing changes and `chore:` for meta/keep-alive commits. The two automated workflows share the exact subject `docs(readme): update profile` so the squash detection works — if you rename one, rename both.
- **Indentation** (from `.editorconfig`): 2-space, LF line endings, UTF-8, final newline required. Trailing whitespace is **preserved** in Markdown (it carries linebreak semantics) — do not strip it.
- **CODEOWNERS**: `* @D3strukt0r` — everything is owned by the profile owner.

## Commands

There is no build, test, or lint toolchain. Typical operations:

```bash
# Trigger workflows manually (requires gh CLI auth)
gh workflow run update-readme.yml
gh workflow run grs.yml

# Inspect recent workflow runs and logs
gh run list --limit 10
gh run view <run-id> --log

# View a specific failing job's logs
gh run view <run-id> --log-failed
```

To preview `README.md` as it renders on the profile, push to a branch and view on GitHub — local Markdown preview does not render GitHub-specific image sizing or dark-mode behavior identically.
