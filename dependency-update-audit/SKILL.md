---
name: dependency-update-audit
description: Audit Renovate and Dependabot pull requests by reading each bump's real changelog, then flag what this codebase must act on (breaking changes, deprecations, security fixes) versus what is noise. Use when the user wants bot dependency-update PRs reviewed, or asks what changed across recent Renovate/Dependabot bumps.
---

# Dependency update audit

Read the changelog behind each Renovate/Dependabot version bump and report which ones this repo must act on, which to adopt, and which are noise. Audit all open bot PRs in the repo — those are the ones still awaiting a decision. Every PR found in step 1 gets a line in the final report. The audit is done when none are unaccounted for.

## Verdicts

Every PR ends with exactly one verdict:

- **Should review** — breaking changes to APIs or config this repo uses, removed/deprecated surfaces it uses, security advisories, required migration steps, or a raised runtime floor (Node, Python, etc.) above what the repo targets.
- **Adopt** — a specific new API or pattern that clearly beats what this repo does with that dependency today. Name both the current approach and the replacement. The recommended action is to migrate: say what to change and where (file:line). If something about the new approach is unconfirmed, list it as a check to run during the migration, not a reason to only suggest a trial. With no concrete replacement to name, it's `Nothing notable`.
- **Nothing notable** — everything else. Most PRs land here. For a routine bump that is the correct answer, not a gap to fill.

## Steps

### 1. List the PRs

1. Run one query per bot, keeping the state inside the search string:

   ```bash
   gh pr list --search "author:app/renovate is:open" --json number,title,url,body,state,createdAt,mergedAt --limit 200
   gh pr list --search "author:app/dependabot is:open" --json number,title,url,body,state,createdAt,mergedAt --limit 200
   ```

2. If the user gave an explicit window ("last month", "since we upgraded X", "all merged this year", a specific PR number or list), swap `is:open` for `is:merged merged:>=2026-08-01` (or `is:closed`) accordingly.
3. If `gh` errors (no GitHub remote, not authenticated), report that and stop. There is no fallback source for this data.

### 2. Parse each PR

1. Extract the package name, ecosystem (npm/pip/docker/etc.), old version → new version, and bump level (major/minor/patch). The title usually states it ("Update dependency foo to v3"); comparing the two version numbers confirms it.
2. For a **grouped PR** ("Update all non-major dependencies", "Lock file maintenance", Dependabot groups), split it into its packages. Audit each one separately in the steps below, then report the group as one PR entry listing its packages.
3. Note where the release notes live. Renovate embeds them inline in collapsible `<details>` blocks; Dependabot links out to "Release notes" / "Changelog" / "Commits".
4. If a body looks truncated, re-fetch that one PR with `gh pr view <number> --json body`.

### 3. Get the changelog for the version range

Scope is old exclusive to new inclusive. For each package:

1. Notes already in the PR body (usual for Renovate) → read them there and fetch nothing.
2. Otherwise `WebFetch` the release-notes or compare link from the body, scoped to the releases between old and new rather than the project's whole history.
3. Major bump → always fetch.
4. Patch bump with no linked notes → `Nothing notable` without a fetch, unless the user asked for full coverage.
5. Link 404s, or the release has no notes → say exactly that in the entry and leave the verdict at `Nothing notable`. A version number alone is never evidence of what changed.

### 4. Judge each change against this codebase

For each changelog entry that could matter:

1. Grep for how this repo actually uses the package: imports, config keys, CLI flags, plugin names. Note the file you found.
2. If the entry touches an API this repo never uses, the verdict is `Nothing notable`.
3. Before ruling something out because a symbol name is absent, check the *precondition* the entry depends on (a non-default schema, a specific runtime, a config flag being set), not just the name of the package or adapter in the release notes. A library can reach the same internal code path through a different entry point, such as a raw connection/pool instead of a dedicated adapter package.
4. If you can't tell whether two entry points share code, say so in the verdict instead of asserting they don't.
5. For raised runtime floors, compare against what the repo targets in `engines`, CI config, or toolchain files.
6. Assign a verdict from the list above.

### 5. Report

1. Order entries `Should review` first, then `Adopt`, then `Nothing notable`.
2. Write notable entries as:
   - **package old → new** — PR #N (link), open/merged
   - `Verdict:` on its own line
   - 1-3 sentences: what changed, why it reaches this repo (cite the file from step 4), and the action to take
3. Write `Nothing notable` entries as one condensed line each: `eslint 9.1 → 9.2 — Nothing notable (PR #123)`.
4. Check every PR from step 1 has an entry before finishing.
