# fixer

Commit the changes made by your fix commands (formatters, linters with autofix…) back to the pull request, with
your own GitHub App. A self-hosted take on [autofix.ci](https://autofix.ci)'s design.

```
fix job (pull_request, read-only, no secrets)
  your setup + fix commands → nicebots-xyz/fixer (diff → patch artifact)
push job (reusable workflow, never runs pull request code)
  download patch → refuse .github/ changes → app token → apply, commit, push
```

The job that runs pull request code (and installs its dependencies, e.g. a Renovate lock file update) never holds a
write token. Commits pushed with the app token trigger CI again, unlike commits pushed with `GITHUB_TOKEN`.

## Usage

```yaml
name: Fixer

on:
  pull_request:

permissions:
  contents: read

jobs:
  fix:
    runs-on: ubuntu-latest
    outputs:
      changed: ${{ steps.fixer.outputs.changed }}
    steps:
      - uses: actions/checkout@<sha> # vX.Y.Z

      # Your setup and fix commands, for example:
      - uses: pdm-project/setup-pdm@<sha> # vX.Y.Z
      - run: pdm install -d --no-default
      - run: pdm run ruff check --fix --exit-zero .
      - run: pdm run ruff format .

      - id: fixer
        uses: nicebots-xyz/fixer@<sha> # vX.Y.Z

  push:
    needs: fix
    if: needs.fix.outputs.changed == 'true'
    uses: nicebots-xyz/fixer/.github/workflows/push.yaml@<sha> # vX.Y.Z
```

Fix commands must not fail the job when something is left to fix by hand (hence `--exit-zero`): the push job only
runs when the fix job succeeds.

### Inputs

| Where | Input | Default |
| --- | --- | --- |
| action | `artifact-name` | `fixer-patch` |
| `push.yaml` | `artifact-name` | `fixer-patch` |
| `push.yaml` | `commit-message` | `style: Apply automated fixes` |
| `push.yaml` | `environment` | `fixer` |

## Setup

1. Create a GitHub App for the organization: no webhook, repository permission **Contents: Read and write** only.
   Install it on the repositories that use fixer.
2. In each repository that uses fixer, create a `fixer` environment with the secrets `FIXER_APP_CLIENT_ID` and
   `FIXER_APP_PRIVATE_KEY`. Only the push job runs in that environment (without creating deployments), so the
   caller doesn't pass any secrets. Passing them explicitly with `secrets:` also works, e.g. from repository
   secrets.
3. While this repository is private, allow other repositories to use it: *Settings → Actions → General → Access →
   Accessible from repositories in the organization*. Public repositories can only use it once it is public.
4. With Renovate, add the app's commit email to `gitIgnoredAuthors`, so a fix commit does not stop Renovate from
   updating or automerging its branch.

## Behavior

- Pull requests from forks are skipped: the app token can only push to branches of the repository itself.
- Patches touching `.github/` are refused.
- When the last commit is already a fixer commit and the fixes still change files, the job fails instead of looping.
- When the branch moved while the fixes ran, nothing is pushed: the run for the new head fixes it.
