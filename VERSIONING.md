# How versioning works for consumers of this repo

This repo is a shared "recipe" for automated versioning, used by other
repos (consumers) via the reusable workflow and Renovate preset defined
here. This document explains the pieces for anyone unfamiliar with this
kind of tooling.

## The big picture

Two kinds of repos work together:

- **This repo (`github-ci-templates`)** — defines the release logic
  once.
- **Consumer repos** (e.g. `test_versioning`) — just reference this
  repo and supply their own version number and commit history.

When a consumer repo pushes a commit to `main`, GitHub Actions reads
the commit messages, decides whether they warrant a new version (and
how big a bump), updates the version number in the code, and publishes
a Git tag + GitHub Release — all without a human manually deciding
"this is now v1.2.0."

## Files in this repo

| File | Purpose |
|---|---|
| **`.github/workflows/semantic-release.yml`** | The actual release logic, written once and shared. It checks out the consumer repo, runs the `python-semantic-release` tool, which: reads commit messages since the last release, decides the version bump, updates `pyproject.toml`, creates a git tag, and publishes a GitHub Release. |
| **`default.json`** | The shared Renovate preset — rules like "auto-merge dependency PRs." Consumer repos extend this from their own `renovate.json`. |
| **`snippets/pyproject.semantic-release.toml`** | Not "live" config — just a copy/paste template showing consumer repos what to put in their own `pyproject.toml`. (Can't be referenced remotely because it's static TOML, not executable.) |
| **`README.md`** | Step-by-step instructions for wiring a new repo up to this pattern. |

## What a consumer repo needs

| File | Purpose |
|---|---|
| **`pyproject.toml`** | Python's standard project file. Holds the current version number (`project.version`) and the semantic-release settings (`[tool.semantic_release]`) — e.g. `allow_zero_version = true` lets versions start below 1.0.0 instead of jumping straight there. |
| **`.github/workflows/release.yml`** | The trigger. Runs on every push to `main` (or manually via "Run workflow"). It doesn't contain any release *logic* itself — it just calls out to this repo's reusable workflow, pinned to a tag like `@v1`. |
| **`renovate.json`** | Config for Renovate (a dependency-update bot). Points at this repo's shared preset so the consumer automatically gets dependency-update rules without maintaining its own. |

## How a release actually gets decided

This is the part that trips people up: **the version bump is driven by
commit message prefixes** (a convention called
[Conventional Commits](https://www.conventionalcommits.org/)):

- `feat!: ...` or a `BREAKING CHANGE:` footer → major bump (1.0.0 → 2.0.0)
- `feat: ...` → minor bump (1.0.0 → 1.1.0)
- `fix:`, `chore:`, `docs:`, `refactor:`, `ci:`, etc. → patch bump
  (1.0.0 → 1.0.1)
- an unrecognized prefix, or none at all → no release

Grouping `chore`/`docs`/`refactor` into patch bumps is a deliberate
departure from `python-semantic-release`'s defaults, which release only
on `feat`/`fix`/breaking. It comes from the `commit_parser_options` block in
[`snippets/pyproject.semantic-release.toml`](snippets/pyproject.semantic-release.toml),
and it means **any merged PR with a recognized prefix produces a release**.

Why: without it, a `chore:` or `refactor:` PR lands on `main` and silently
produces no tag, so the code on `main` and the newest release drift apart.
That bites hardest with the dependency bumps Renovate automerges — they are
`chore(deps):` commits, and a repo whose release artifacts are attached to
"the latest release" would keep building from a stale tag.

So: no commits, or only unprefixed ones since the last release → nothing
happens, no new tag. Anything else produces a version.

## The 0.x version gotcha

`python-semantic-release` refuses to let the *first automated release*
stay below 1.0.0 unless the consumer repo explicitly opts in
(`allow_zero_version = true` in its `pyproject.toml`). Without that
setting, the very first automated run jumps straight to `v1.0.0` even
if the version file says `0.0.1`.

## Why consumers pin to `@v1` instead of `@main`

Consumer repos reference this repo's reusable workflow at a tag
(`@v1`), not `@main`. That means this repo can change freely, and
consumers are unaffected until someone deliberately moves the `v1` tag
forward to a new commit. This protects every consuming repo from being
silently broken by a change made here.
