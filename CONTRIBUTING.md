# Setting up a consumer repo: PRs and merging

Any repo that wires up automated versioning via this template (see
[README.md](README.md) and [VERSIONING.md](VERSIONING.md)) should also
adopt a specific merge strategy in its GitHub repo settings. This isn't
optional polish — without it, the versioning pipeline becomes
unpredictable.

## Required setting: squash-only merging

In the consumer repo, go to **Settings → General → Pull Requests** and
set:

- **Allow merge commits** — disabled
- **Allow squash merging** — enabled, with default commit message set
  to **Pull request title**
- **Allow rebase merging** — disabled

## Why

The reusable release workflow in this repo runs `python-semantic-release`
on every push to `main`, and it decides whether/how to bump the version
by parsing commit messages on `main` using the
[Conventional Commits](https://www.conventionalcommits.org/) convention
(`feat:`, `fix:`, `feat!:`/`BREAKING CHANGE:`, etc.).

If merge commits or rebase merging are allowed, every individual commit
from a contributor's feature branch lands on `main` as-is and gets
parsed individually — including messy, in-progress, or unprefixed
commits nobody intended to be release-worthy. That makes releases
unpredictable and hard to reason about.

With **squash-only** merging, every PR becomes exactly one commit on
`main`, and that commit's message defaults to the **PR title**. This
makes the PR title the single, predictable thing that decides whether
a release happens and how big it is — nothing on the contributor's
branch does.

## Contributing notes to pass on to consumer repo contributors

Once squash-only merging is set up, contributors should know:

- **Commit however you like on your feature branch.** None of it
  reaches `main` as-is, so prefixes/history there don't matter.
- **The PR title is what matters.** Write it as a Conventional Commit:
  - `feat!: ...` or a `BREAKING CHANGE:` footer → major release
  - `feat: ...` → minor release
  - `fix: ...`, `chore: ...`, `docs: ...`, `refactor: ...`, etc. →
    patch release
  - an unrecognized prefix, or none → no release

  Every recognized prefix releases something (see the
  [snippet](snippets/pyproject.semantic-release.toml)), so pick the one
  that describes the change rather than trying to avoid a version bump.
- **Double-check the squash commit message before confirming the
  merge.** GitHub pre-fills it from the PR title, but it's editable at
  merge time in the "Squash and merge" box. Whatever that final message
  says is what actually gets parsed — leave it as the PR title unless
  intentionally changing the release type.
- **No recognized prefix anywhere → no release.** semantic-release just
  skips it (`no_release`); no tag, no version bump, no error.

Feel free to copy the contributor-facing section above into the
consumer repo's own `CONTRIBUTING.md`.
