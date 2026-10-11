# TunaOS org defaults

This is the configuration defaults repository for the `tuna-os` organization.
Nothing here builds a product. Every change here applies to other repositories.

## What this repo is

`tuna-os/.github` holds the shared CI/CD defaults that the rest of the
organization consumes:

- Workflows and composite actions that repos `use:` directly.
- A copyable project starter for new repositories.
- Org-wide templates, issue and PR forms, and contribution policy.
- Dependency-check scripts that enforce org policy in consumer repos.
- The org profile that renders on the public landing page.

## The blast radius

A change here reaches many repositories at once. Four mechanisms carry it:

| Path | Reaches |
| --- | --- |
| `.github/ISSUE_TEMPLATE/`, `PULL_REQUEST_TEMPLATE.md`, `DISCUSSION_TEMPLATE/`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md` | Every org repo that does not ship its own copy — immediately, with no merge in that repo |
| `profile/README.md` | The organisation's public landing page |
| `.github/workflows/*.yml` (`on: workflow_call`) and `.github/actions/*` | Every repo that `uses:` them. Callers pin `@main`, so a merge is live for them at once |
| `project-starter/` | A template **copied** into new repos. Not used by this repo |

There is no version to hold a consumer back and no rollback except another
commit. The inputs of `publish-flatpak.yml` and of each action's `inputs:`
block are a public API. Renaming one, or changing what a default means, breaks
callers silently at their next run.

## Repository map

### `.github/workflows/`

Two kinds of workflow live here.

**Reusable** (`on: workflow_call`). Consumer repos call these from their own
CI:

- `reusable-fork-safety.yml` — Fork Safety Checks. Holds fork PRs for
  maintainer review before any write. This is the write gate: your own PR will
  sit as *do-not-merge* until a maintainer approves.
- `reusable-ci-contract.yml` — CI Contract Verification. Asserts every green
  criterion is covered by a reachable job.
- `reusable-lint.yml` — Lint. Runs the shared formatters and linters.
- `reusable-first-contributor.yml` — First Contributor Greetings. Welcomes new
  contributors.
- `reusable-pr-nudges.yml` — PR Reminders. Nudges stale pull requests.
- `reusable-scorecard.yml` — Scorecard. Runs the supply-chain security report.
- `reusable-add-help-wanted.yml` — Add Help Wanted Label. Labels PRs a
  maintainer declines.

**Run in this repo** (`on: push` / `pull_request` / `schedule`):

- `workflow-permissions-check.yml` — every workflow must declare a top-level
  `permissions:` block. This is the org least-privilege baseline
  ([`tuna-os/.github#155`](https://github.com/tuna-os/.github/issues/155)).
- `renovate-policy-check.yml` — enforces the automerge policy against
  `renovate.json`.
- `flatpak-tooling-drift-check.yml` — weekly guard that application repos have
  not diverged from the canonical flatpak index script.
- `drop-bot-review-requests.yml` — drops automated review requests.
- `publish-flatpak.yml` and `ste-lint.yml` — reusable. Consumer repos call
  them from their own publish and CI workflows.

### `.github/actions/`

Composite actions that repos `use:`:

- `publish-flatpak-index` — clones the central index repo, updates one app
  entry, and retries against concurrent writers so a race never drops a
  publish.
- `update-flatpak-index` — updates a Flatpak `index/static` file with a
  published image digest, architecture, and labels. Inputs travel through
  `env:`, never through `${{ }}` interpolation (see [Security](#security)).
- `ste-lint` — checks Markdown prose against ASD-STE100 with a per-repo
  budget.

### `project-starter/`

A curated, copyable baseline for new repos. A new repository copies this
directory to its root. It ships a compliant `ci.yml`, the two check scripts, a
`Justfile`, `renovate.json`, and `ROADMAP.md`.

### `profile/`

`profile/README.md` is the org's public landing page. It renders on
[tunaos.org](https://tunaos.org).

### `scripts/`

Dependency-free checkers that consumer repos copy in and run:

- `check-workflow-permissions.py` — enforces the top-level `permissions:`
  baseline.
- `check-renovate-automerge-policy.py` — enforces the automerge policy.
- `test_check_workflow_permissions.py` and `test-renovate-automerge-policy.py`
  — unit tests for the two checks above.

### `workflow-templates/`

`ci.yml` is a compliant starting point to copy into a consumer repo's
`.github/workflows/ci.yml`. The only mandatory line is the top-level
`permissions:` block.

### `docs/`

`docs/OBSERVABILITY.md` — metrics and observability notes for the org's shared
tooling.

### Root config

- `renovate.json` and `default.json` — the shared Renovate policy. Repos
  consume it with `"extends": ["local>tuna-os/.github"]`.
- `ROADMAP-INDEX.md` — the authoritative inventory of active repositories and
  their default branches.
- `codecov.yml`, `ruff.toml`, `.editorconfig` — shared tooling config.

## Common tasks

### Add a new org-wide workflow

1. Decide whether it is reusable or run in this repo. Call it from consumer
  repos, or trigger it here.
2. Declare a top-level `permissions:` block. Ask only for what each job does.
  See [Permissions model](#permissions-model).
3. Reuse a reusable workflow if one already covers the job. Do not fork it.
4. Add a contract check if the workflow has behaviour worth guarding.

### Add or change a composite action

1. Put the logic in a file under the action directory (`.mjs` or `.py`). Keep
  `action.yml` thin.
2. Route every input through `env:`, never through `${{ }}` interpolation into
  the script body. See [Security](#security).
3. Add tests next to the logic. The `ste-lint` action runs its own rule tests
  by default.

### Adopt org CI in a consumer repo

1. Copy `workflow-templates/ci.yml` into `.github/workflows/ci.yml`.
2. Replace the placeholder build and test commands.
3. Copy `scripts/check-workflow-permissions.py` and run it:

   ```bash
   python3 scripts/check-workflow-permissions.py .github/workflows
   ```

4. Enable branch protection with `CI / required-checks` as the required check.

### Test changes before merging

This repo verifies itself with three workflows on every pull request:

- **Workflow permissions** (`workflow-permissions-check.yml`) — fails if any
  workflow lacks a top-level `permissions:` block.
- **Renovate policy** (`renovate-policy-check.yml`) — fails if `renovate.json`
  automerges a major update.
- **Fork safety** (`reusable-fork-safety.yml`) — holds fork PRs for maintainer
  review before any write.

Run the checkers locally first:

```bash
python3 scripts/check-workflow-permissions.py .github/workflows
python3 scripts/check-renovate-automerge-policy.py renovate.json
python3 scripts/test_check_workflow_permissions.py
python3 scripts/test-renovate-automerge-policy.py
```

## Permissions model

Every workflow must declare an explicit top-level `permissions:` block. A
workflow with no block inherits the repository default token scope, which is
almost always broader than the jobs need. This is the org least-privilege
baseline ([`tuna-os/.github#155`](https://github.com/tuna-os/.github/issues/155)).

```yaml
permissions:
  contents: read
```

Start from what each job does:

| Job does | Ask for |
| --- | --- |
| Check out code, build, read the repo | `contents: read` |
| Push commits, create releases, upload assets | `contents: write` |
| Push images to GHCR, update the flatpak index | `packages: write` |
| Open or edit PRs (bot steps) | `pull-requests: write` |
| Open or edit issues (bot steps) | `issues: write` |

## Security

Actions substitute expressions into the script **text** before bash parses it.
An input routed through `${{ }}` interpolation is parsed as shell. Quoting
narrows the hole but does not close it — a value that contains a double quote
ends the quoted region. An unquoted value can execute shell.

`update-flatpak-index` routes every input through `env:` instead, so a value is
data and never code. This is the migration target for the org's publish
workflows, which hold `packages: write` and the flatpak index token. When you
write a composite action, route its inputs through `env:` the same way.

## Branch and PR policy

- **Base branch:** `main`. Open every PR against `main`.
- **Branch names:** use the `arch/`, `fix/`, `feat/`, `chore/` prefix
  convention.
- **Sign your commits:** every commit must be DCO-signed-off
  (`git commit -s`). The DCO check blocks the merge without it. The
  `Signed-off-by` email must match the commit author.
- **Never merge your own PR.** Fork safety holds your PR for a maintainer.
- **One source of truth per artifact.** Reuse the canonical script or action
  instead of copying it.

## Related documents

- [`CONTRIBUTING.md`](CONTRIBUTING.md) — how to contribute to TunaOS.
- [`AGENTS.md`](AGENTS.md) — agent guide for this repo, including the
  blast-radius rules and the security model.
- [`ROADMAP-INDEX.md`](ROADMAP-INDEX.md) — authoritative inventory of active
  repos and their default branches.
- [Workflow permissions check](`.github/workflows/workflow-permissions-check.yml`)
  and [`scripts/check-workflow-permissions.py`](scripts/check-workflow-permissions.py).
- [`renovate-policy-check.yml`](.github/workflows/renovate-policy-check.yml).
- [`SECURITY.md`](SECURITY.md) — report security issues.
- [`tuna-os/docs`](https://github.com/tuna-os/docs) — architecture and
  build-pipeline reference documentation.
