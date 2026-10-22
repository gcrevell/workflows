# workflows

Reusable GitHub Actions workflows, shared by the repositories under
[gcrevell](https://github.com/gcrevell). Each one lives in `.github/workflows/`
and is called with `workflow_call`, so the logic is written once rather than
copied into every repository.

Callers use them at `@main`, so a change here reaches every caller on its next
run. That's deliberate, and it's also why changes here are worth testing first
(see [Testing a change](#testing-a-change)).

## Workflows

### `calver.yml`: CalVer tagging

Tags the pushed commit with the next version in the form `vYYYY.M.N`, where `N`
counts up from 1 each month in UTC. For example, `v2026.10.1`, `v2026.10.2`,
then `v2026.11.1`. The month isn't zero-padded.

- **Idempotent.** If the commit already has a version tag, it reports that tag
  and stops, so re-running a job never makes a second tag.
- **Serialised per repository.** Runs share a concurrency group, so two merges
  in quick succession can't pick the same number.
- **Output:** `version`, the commit's tag, for later jobs to use.

Call it from a workflow that runs on every push to `main`:

```yaml
name: Version

on:
  push:
    branches: [main]

jobs:
  tag:
    uses: gcrevell/workflows/.github/workflows/calver.yml@main
    permissions:
      contents: write
```

The caller has to grant `contents: write`, because a called workflow can't have
more permissions than its caller gives it.

#### Publishing a release on the tag

A tag pushed with the automatic `GITHUB_TOKEN` doesn't trigger other workflows,
so an `on: push: tags` workflow never sees these tags. To build a release, add a
job to the same workflow that runs after `tag` and reads its output:

```yaml
  release:
    needs: tag
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ needs.tag.outputs.version }}
      # build...
      - env:
          GH_TOKEN: ${{ github.token }}
        run: gh release create "${{ needs.tag.outputs.version }}" dist/*
```

## Dependencies

Actions used here are pinned to a major version (`actions/checkout@v7`).
[Renovate](https://docs.renovatebot.com/) opens PRs when a new major is out,
using the config in `renovate.json`.

## Testing a change

There's no test suite. A change is checked by linting it and then running it
from its branch before it merges.

1. **Lint.** Run [actionlint](https://github.com/rhysd/actionlint) on the
   workflows, which also runs shellcheck on the `run:` scripts:

   ```sh
   brew install actionlint
   actionlint
   ```

2. **Run it from the branch.** In a throwaway repository, not a real caller
   (each run pushes a real tag), point a caller at the branch and push to its
   `main` a few times:

   ```yaml
   uses: gcrevell/workflows/.github/workflows/calver.yml@<branch>
   ```

   For `calver.yml`, check that consecutive pushes tag `.1`, `.2`, and so on;
   that re-running a finished job reports the existing tag and doesn't make a
   new one; and that the `version` output reaches a job that `needs: tag`.

Once it merges, every caller picks it up on its next push to `main`.
