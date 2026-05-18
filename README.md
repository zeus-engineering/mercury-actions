# mercury-actions

> Reusable GitHub Actions workflows for the Zeus org. Mercury = the messenger that carries CI patterns across repos.

## Available workflows

| Workflow | Purpose | Caller invokes via |
|---|---|---|
| `.github/workflows/go-publish.yml` | Tag the caller's repo with a semver tag so its Go module / sub-packages are consumable via `go get` | `uses: zeusml-ai/mercury-actions/.github/workflows/go-publish.yml@main` |

More to come (npm-publish, container-publish, ...).

## Go module publishing

### One-time setup in the producing repo

Create `.github/workflows/release.yml`:

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      bump:
        type: choice
        options: [patch, minor, major]
        default: patch
      dry_run:
        type: boolean
        default: false

jobs:
  publish:
    permissions:
      contents: write
    uses: zeusml-ai/mercury-actions/.github/workflows/go-publish.yml@main
    with:
      bump: ${{ inputs.bump }}
      dry_run: ${{ inputs.dry_run }}
```

That's it. Push to main, then Actions → Release → Run workflow → pick `patch`/`minor`/`major` → tag lands.

### One-time setup in the consuming repo

For private Zeus repos, Go needs to bypass the public module proxy and authenticate to GitHub directly. In **CI** (workflow env):

```yaml
env:
  GOPRIVATE: github.com/zeusml-ai/*
  GH_TOKEN: ${{ secrets.Z_GITHUB_API_TOKEN }}
steps:
  - run: git config --global url."https://x-access-token:${GH_TOKEN}@github.com/".insteadOf "https://github.com/"
```

In **local dev** (run once per machine):

```sh
export GOPRIVATE='github.com/zeusml-ai/*'   # add to ~/.zshrc
gh auth refresh --scopes repo                # personal token must have repo scope
git config --global url."https://oauth2:$(gh auth token)@github.com/".insteadOf "https://github.com/"
```

### Consuming a published package

```sh
go get github.com/zeus-engineering/z-infra@v0.1.0
```

Then `import "github.com/zeus-engineering/z-infra/pkg/flyapp"` and use as normal.

## Versioning policy

Use semver. The reusable workflow expects you to pass the bump type:

- `patch` — bug fix, no API change
- `minor` — new function/type, backwards-compatible
- `major` — breaking change, removed/renamed API

For Go submodule paths (e.g. `github.com/foo/bar/sub` with its own `go.mod`), use `tag_prefix: 'sub/v'` so tags look like `sub/v0.1.0`.
