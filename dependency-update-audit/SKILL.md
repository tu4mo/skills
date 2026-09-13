---
name: dependency-update-audit
description: Audit Renovate and Dependabot pull requests in this repo, read each dependency's changelog/release notes, and flag breaking changes, deprecations, or better new patterns relevant to this codebase. Use when the user wants a review of bot dependency-update PRs, a changelog audit, or asks "what changed" across recent Renovate/Dependabot bumps.
---

# Dependency update audit

Go through Renovate and Dependabot PRs in this repo, read the actual changelog for each version bump, and report anything the project should act on (breaking change, deprecated API still in use, security fix) or could consider (a newer/better recommended pattern). Skip PRs where nothing in the changelog is relevant — don't pad the report with routine patch bumps that have no notable content.

## Scope

Default to **open** Renovate/Dependabot PRs — those are the ones still awaiting a decision. If the user asks for history ("last month", "since we upgraded X", "all merged this year", a specific PR number/list), honor that instead. If it's ambiguous and nothing in context narrows it, proceed with open PRs by default rather than asking — mention in the final report that merged PRs weren't included and can be added on request.

## Workflow

1. **List the PRs.**
   ```
   gh pr list --search "author:app/renovate" --state <open|all|closed> --json number,title,url,body,createdAt,mergedAt,state --limit 200
   gh pr list --search "author:app/dependabot" --state <open|all|closed> --json number,title,url,body,createdAt,mergedAt,state --limit 200
   ```
   If the user gave a date/window, add it to the search query (e.g. `merged:>=2026-08-01`) or filter the JSON results afterward.

2. **Parse each PR** for: package name, ecosystem (npm/pip/docker/etc.), old version, new version, and whether it's a major/minor/patch bump (the title usually says, e.g. "Update dependency foo to v3"). Renovate PR bodies often already embed release-note excerpts in collapsible `<details>` blocks — extract those directly instead of re-fetching. Dependabot bodies usually list "Release notes" / "Changelog" / "Commits" links instead of inlining text.

3. **Get the real changelog** for each version range:
   - If the PR body already contains the changelog text (common for Renovate), use it as-is.
   - Otherwise, use `WebFetch` on the changelog/release-notes link(s) from the PR body, scoped to just the versions between old and new (a compare link or the specific release tags — don't fetch the whole changelog history if it's long, just the entries between old and new).
   - Skip fetching for trivial patch bumps with no linked notes unless the user asked for full coverage.

4. **Judge relevance to this project**, not just to the package in the abstract:
   - Grep the codebase for actual usage of the package (imports, config keys, CLI flags) before deciding a changelog entry matters — a breaking change in an API this project doesn't use isn't worth flagging.
   - Flag: breaking changes, removed/deprecated APIs or config options that this repo currently uses, security advisories, required migration steps, and language/runtime requirement bumps (Node, Python, etc.).
   - Optionally note: a newer recommended pattern or API that would be a clear improvement over what this repo currently does with that dependency (only if it's a real, specific improvement — not generic "check the docs" filler).
   - It's fine and expected for most entries to come back with "nothing notable" — that's a useful result, not a failure to find something.

5. **Report as a list**, one entry per PR, in this shape:
   - **Package @ old → new** (PR #, link, state: open/merged)
   - **Verdict:** one of `Nothing notable` / `Should review` / `Consider adopting`
   - If not "Nothing notable": 1-3 sentences on what specifically changed, why it matters to this repo (cite the file/usage found in step 4 where applicable), and the suggested action.

   Order the list so `Should review` items come first, then `Consider adopting`, then `Nothing notable` (these can be a single condensed line each, e.g. "eslint 9.1→9.2 — Nothing notable").

## Notes

- Don't open a browser or re-derive version numbers from `package.json`/lockfiles — the PR title and body already state the bump; trust them.
- For a large batch of open PRs, it's fine to work through them one at a time in this conversation rather than spawning subagents, unless the user asks for it to run in the background.
- If a PR's changelog link 404s or the release has no notes, say so plainly rather than guessing at what changed from the version number alone.
