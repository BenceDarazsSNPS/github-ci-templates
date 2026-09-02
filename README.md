# github-ci-templates

Central home for the versioning/release pattern shared across repos
(python-semantic-release + Renovate). Update once here, every consuming
repo picks up the change.

## 1. Release workflow (reusable)

In the consuming repo, replace the release job body with a call to this
repo's reusable workflow:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      force:
        description: "Force a specific version bump instead of using conventional commits"
        required: false
        type: choice
        options: [auto, patch, minor, major, prerelease]
        default: auto

permissions:
  contents: write

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  release:
    uses: BenceDarazsSNPS/github-ci-templates/.github/workflows/semantic-release.yml@v2
    with:
      force: ${{ github.event.inputs.force != 'auto' && github.event.inputs.force || '' }}
```

Pin to a tag (`@v2`) rather than `@main` so this repo can change without
silently breaking every consumer; bump the tag consumers use when you
intentionally want them to pick up a change.

If other repos depend on this one through `tool.uv.sources`, list them
so they get told the moment a release happens (see section 4):

```yaml
jobs:
  release:
    uses: BenceDarazsSNPS/github-ci-templates/.github/workflows/semantic-release.yml@v2
    with:
      force: ${{ github.event.inputs.force != 'auto' && github.event.inputs.force || '' }}
      dependents: |
        BenceDarazsSNPS/hps_config_editor
        BenceDarazsSNPS/pyhps_job_submission
    secrets: inherit
```

The workflow also exposes `released`, `version` and `tag` as outputs if
the caller needs to hang further jobs off a release.

## 2. Renovate config (shared preset)

Replace the consuming repo's `renovate.json` with:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>BenceDarazsSNPS/github-ci-templates"]
}
```

This resolves to [`default.json`](default.json) in this repo. Renovate
must have access to this repo (same org/owner is enough by default).

Repos that use the internal dependency chain in section 4 should also
turn Renovate off for `tool.uv.sources`, so the two mechanisms do not
both raise a PR for the same bump:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>BenceDarazsSNPS/github-ci-templates"],
  "packageRules": [
    {
      "description": "Owned by .github/workflows/sync-deps.yml, not Renovate.",
      "matchDepTypes": ["tool.uv.sources"],
      "enabled": false
    }
  ]
}
```

## 3. `pyproject.toml` semantic-release config

Not reusable via reference (it's static TOML, not executable), so it's
just kept here as a canonical snippet to copy:
[`snippets/pyproject.semantic-release.toml`](snippets/pyproject.semantic-release.toml).


## 4. Internal dependency chain (push-based)

Renovate polls. For dependencies between your own repos that is the wrong
shape: a release in an upstream repo can sit for hours before the
downstream PR appears, and a three-repo chain multiplies that wait by
three. [`sync-uv-git-sources.yml`](.github/workflows/sync-uv-git-sources.yml)
replaces the poll with a push.

How one hop works:

1. Upstream repo releases. `semantic-release.yml` tags it, then its
   `notify-dependents` job sends a `repository_dispatch` of type
   `upstream-release` to every repo named in `dependents`.
2. The downstream repo's `sync-deps.yml` wakes up, points **every**
   `tool.uv.sources` git-tag entry at the newest semver tag its remote
   has, runs `uv lock`, and runs the test suite against the result.
3. If the tests pass it force-pushes `deps/uv-sources-sync` and opens or
   updates one PR. With `merge: true` it squash-merges immediately,
   which releases that repo and dispatches to *its* dependents, so the
   chain continues on its own.

The sync always syncs *all* sources to their newest tag rather than
applying the single version from the payload. That makes the job
idempotent: whatever triggered it, the branch ends up as "the default
branch with every internal dependency current", so concurrent upstream
releases converge instead of fighting.

Consumer side:

```yaml
# .github/workflows/sync-deps.yml
name: Sync internal dependencies

on:
  repository_dispatch:
    types: [upstream-release]
  schedule:
    # Safety net only, for a dispatch that never arrived.
    - cron: "0 4 * * *"
  workflow_dispatch: {}

jobs:
  sync:
    uses: BenceDarazsSNPS/github-ci-templates/.github/workflows/sync-uv-git-sources.yml@v2
    with:
      merge: true   # false on the leaf repo you want to review by hand
    secrets: inherit
```

### The token

Both halves need a `DEPS_PAT` secret in **every** repo in the chain
(GitHub free plan, personal account: there are no org-level secrets, so
it has to be set per repo). A fine-grained PAT scoped to the repos in
the chain with:

- **Contents: read and write** — clone the private sources, push the
  sync branch, dispatch to dependents
- **Pull requests: read and write** — open and merge the sync PR

It must be a PAT and not `GITHUB_TOKEN`: pushes made with `GITHUB_TOKEN`
deliberately do not start further workflow runs, which would stop the
chain dead at the first hop.

```bash
gh secret set DEPS_PAT -R OWNER/REPO
```

### Why the tests run before the PR is opened

These repos are private on the free plan, so branch protection and
required status checks are not available, which makes `gh pr merge
--auto` meaningless. Running the suite inside the sync job before it
pushes gives the same guarantee without them: a bump that breaks the
downstream repo never becomes a PR, let alone a merged one.

## Versioning this repo

Tag releases (`v1`, `v1.1`, ...) so consumers can pin a stable reference
in their `uses:` line instead of tracking `main` directly.
