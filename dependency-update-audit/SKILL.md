---
name: dependency-update-audit
description: Audit Renovate and Dependabot pull requests by reading each bump's real changelog, then flag what this codebase must act on (breaking changes, deprecations, security fixes) versus what is noise. Use when the user wants bot dependency-update PRs reviewed, or asks what changed across recent Renovate/Dependabot bumps.
---

# Dependency update audit

Read the changelog behind each Renovate/Dependabot version bump and report which ones this repo must act on, which are worth adopting, and which are noise. Every PR found in step 1 gets a line in the final report — the audit is done when none are unaccounted for.

## Scope

Default to **open** bot PRs — those are the ones still awaiting a decision. Honor an explicit window instead when the user gives one ("last month", "since we upgraded X", "all merged this year", a specific PR number or list). When the request is ambiguous, run on open PRs and close the report by noting that merged PRs were excluded and can be added on request.

## Workflow

1. **List the PRs.** Keep the state inside the search string so one query per bot covers it:

   ```bash
   gh pr list --search "author:app/renovate is:open" --json number,title,url,body,state,createdAt,mergedAt --limit 200
   gh pr list --search "author:app/dependabot is:open" --json number,title,url,body,state,createdAt,mergedAt --limit 200
   ```

   For a history window, swap `is:open` for `is:merged merged:>=2026-08-01` (or `is:closed`) per Scope. If `gh` errors — no GitHub remote, not authenticated — report that and stop; there is no fallback source for this data.

2. **Parse each PR** into: package name, ecosystem (npm/pip/docker/etc.), old version → new version, and bump level (major/minor/patch). The title usually states it ("Update dependency foo to v3"); comparing the two version numbers confirms it. Two shapes need extra care:
   - **Grouped PRs** — "Update all non-major dependencies", "Lock file maintenance", Dependabot groups — bump several packages at once. Audit each package inside separately, then report the group as one PR entry listing its packages.
   - Renovate embeds release notes inline in collapsible `<details>` blocks; Dependabot instead links out to "Release notes" / "Changelog" / "Commits". If a body looks truncated, re-fetch that one with `gh pr view <number> --json body`.

3. **Get the changelog for the version range**, old exclusive to new inclusive:
   - Body already contains the release notes (usual for Renovate) → read them there, fetch nothing.
   - Otherwise `WebFetch` the release-notes or compare link from the body, scoped to the releases between old and new rather than the project's whole changelog history.
   - Major bumps always get fetched. A patch bump with no linked notes is `Nothing notable` without a fetch, unless the user asked for full coverage.
   - Link 404s, or the release has no notes → say exactly that in the entry and leave the verdict at `Nothing notable`. A version number alone is never evidence of what changed.

4. **Judge each change against this codebase, with evidence.** Before flagging a changelog entry, grep for how this repo actually uses the package — imports, config keys, CLI flags, plugin names — and note the file you found. A breaking change to an API this repo never touches is `Nothing notable`.
   - **Should review:** breaking changes to APIs or config this repo uses, removed/deprecated surfaces it uses, security advisories, required migration steps, and raised runtime floors (Node, Python, etc.) checked against what the repo targets in `engines`, CI config, or toolchain files.
   - **Consider adopting:** a specific new API or pattern that clearly beats what this repo does with that dependency today — name both the current approach and the replacement. With no concrete replacement to name, it's `Nothing notable`.
   - Most entries land on `Nothing notable`. For a routine bump that is the correct answer, not a gap to fill.

5. **Report**, one entry per PR, ordered `Should review` first, then `Consider adopting`, then `Nothing notable`:
   - Notable entries: **package old → new** — PR #N (link), open/merged. Then `Verdict:` on its own line, then 1-3 sentences covering what changed, why it reaches this repo (cite the file from step 4), and the action to take.
   - `Nothing notable` entries: one condensed line each — `eslint 9.1 → 9.2 — Nothing notable (PR #123)`.

## Notes

- The PR title and body are authoritative for version numbers; reading `package.json` or lockfiles to confirm them adds nothing.
- Work through the batch one PR at a time in this conversation. Spawn subagents only when the user asks for the audit to run in the background.
