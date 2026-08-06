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

- `fix: ...` → patch bump (1.0.0 → 1.0.1)
- `feat: ...` → minor bump (1.0.0 → 1.1.0)
- `feat!: ...` or a `BREAKING CHANGE:` footer → major bump (1.0.0 → 2.0.0)
- `chore:`, `docs:`, `refactor:`, etc. → no release at all

So: no commits, or only `chore`/`docs` commits since the last release →
nothing happens, no new tag. Commits with `feat`/`fix` prefixes will
trigger a new version.

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
