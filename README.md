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
    uses: BenceDarazsSNPS/github-ci-templates/.github/workflows/semantic-release.yml@v1
    with:
      force: ${{ github.event.inputs.force != 'auto' && github.event.inputs.force || '' }}
```

Pin to a tag (`@v1`) rather than `@main` so this repo can change without
silently breaking every consumer; bump the tag consumers use when you
intentionally want them to pick up a change.

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

## 3. `pyproject.toml` semantic-release config

Not reusable via reference (it's static TOML, not executable), so it's
just kept here as a canonical snippet to copy:
[`snippets/pyproject.semantic-release.toml`](snippets/pyproject.semantic-release.toml).

## Versioning this repo

Tag releases (`v1`, `v1.1`, ...) so consumers can pin a stable reference
in their `uses:` line instead of tracking `main` directly.
