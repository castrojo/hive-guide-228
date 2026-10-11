# Validation Scripts

Two dependency-free Python scripts in `scripts/` enforce org policy in CI. They
have no third-party dependencies (no `pip install`), so they run on GitHub's
`ubuntu` runner as plain `python3`. Each ships a plain-`python3` self-test in
`scripts/test-*.py`, and each has a byte-identical copy under
`project-starter/scripts/` (see [Copying into project-starter and consumer repos](#copying-into-project-starter-and-consumer-repos)).

| Script | Enforces | Fails when |
|---|---|---|
| [`check-renovate-automerge-policy.py`](#check-renovate-automerge-policypy) | [tuna-os/.github#12](https://github.com/tuna-os/.github/issues/12) — which dependency updates may automerge | `renovate.json` would automerge a **major** update, directly or by rule layering |
| [`check-workflow-permissions.py`](#check-workflow-permissionspy) | [tuna-os/.github#155](https://github.com/tuna-os/.github/issues/155) — least-privilege token scope | a workflow has no non-empty top-level `permissions:` block |

Both are security gates. Their logic errs toward **flagging more, not less**: a
false positive is a human looking at something; a false negative is a bypass
getting merged. That bias is deliberate — it is the mistake class behind the
incidents each script exists to prevent.

---

## check-renovate-automerge-policy.py

### Purpose

Org policy ([#12](https://github.com/tuna-os/.github/issues/12)): **minor,
patch, `pin`, `pinDigest`, and `digest`** updates may automerge once a repo's CI
is green. **major** updates always stop for human review before merge.

The gate exists because a syntactically valid `renovate.json` can still
automerge a major update even when no single rule says so — once
`packageRules` are layered, a broad `automerge: true` rule can be overridden by
a later narrow one. A JSON schema validator cannot see that. Two broad-automerge
incidents in `tuna-os/tunaOS` ([#1612](https://github.com/tuna-os/tunaOS/issues/1612),
[#1636](https://github.com/tuna-os/tunaOS/issues/1636)) were exactly this class.

The blocked set was narrowed from `{major, minor}` to `{major}` on purpose:
minor is the bulk of the fleet's dependency traffic, and holding it for review
produced a review queue nobody read — its own failure mode. Major stays blocked.

### What it checks

It is a conservative simulation of how Renovate layers config, **not** a full
Renovate rule-matching engine:

- It starts from the top-level `automerge` default and walks
  `packageRules` in order, applying each **unscoped** rule that sets
  `automerge` (later rules win), exactly like Renovate.
- A rule with **no** `matchUpdateTypes` applies to every update type, major
  included.
- Rules that use a scoping key (`matchPackageNames`, `matchDatasources`,
  `matchManagers`, …) are judged **independently** and never get to *cancel* a
  violation an earlier broad rule introduced. A narrow override is not evidence
  that a broad bypass stopped applying to every other package.
- It ignores scoping for the actual gate: any rule whose `matchUpdateTypes`
  includes (or omits) `major` is treated as org-wide. This can flag a rule that
  in practice only matches one low-risk package — a false positive a human then
  has to clear.

### Usage

```bash
python3 scripts/check-renovate-automerge-policy.py [path/to/renovate.json]
```

The path defaults to `renovate.json`. CI runs it against three files at once
(`renovate-policy-check.yml`):

```bash
python3 scripts/check-renovate-automerge-policy.py default.json
python3 scripts/check-renovate-automerge-policy.py renovate.json
python3 scripts/check-renovate-automerge-policy.py project-starter/renovate.json
```

### Exit codes

| Code | Meaning |
|---|---|
| `0` | Compliant — no rule automerges `major`. Prints `OK: <path> does not automerge major updates.` to stdout. |
| `1` | Violation found, **or** the file could not be read/parsed. Prints `FAIL:` (violations) or `ERROR:` (parse failure) to stderr. |

A parse error and a policy violation share exit code `1` — both fail the build.
`ERROR: could not read/parse <path>: <e>` means the file is missing or invalid
JSON; `FAIL:` means the JSON parsed but a rule violates policy.

### Common failures and how to fix them

| Symptom | Cause | Fix |
|---|---|---|
| `FAIL: renovate.json violates the org automerge policy` with `... 'major' updates would automerge, set by packageRules[2]` | A rule with `matchUpdateTypes` that includes `"major"` sets `automerge: true` | Set that rule's `automerge: false`, or drop it — major must stop for review |
| `... set by packageRules[0]` and the rule has no `matchUpdateTypes` | A rule with `automerge: true` and no `matchUpdateTypes` applies to *every* type, major included | Scope the rule to non-major types, or set `automerge: false` |
| `... set by top-level automerge` | Top-level `"automerge": true` covers major | Set top-level `automerge: false` (or `platformAutomerge: false`) and gate major explicitly |
| A rule scoped with `matchPackageNames`/`matchDatasources`/… still fails | Scoped rules enabling major automerge are caught independently | Apply the same fix to the scoped rule |
| A narrow `major: false` rule does **not** clear an earlier broad `automerge: true` | Correct behaviour — the broad bypass is the real problem (the `protota` regression) | Fix the broad rule; the narrow override only documents the intent |
| `ERROR: could not read/parse renovate.json: Expecting value: line N column M` | Invalid JSON (trailing comma, single quotes, stray comment) | Fix the JSON; the file must be valid, parseable JSON |

### When and how to add a new policy

1. **Org-wide policy** — edit `default.json`'s `packageRules`. Repos consume it
   with `"extends": ["local>tuna-os/.github"]` and add only repo-specific
   overrides on top. This is the single source of truth.
2. **Repo-specific override** — add a rule to that repo's `renovate.json`.
3. After any change, run the check against `default.json`, `renovate.json`, and
   `project-starter/renovate.json`, then run the self-test:

   ```bash
   python3 scripts/test-renovate-automerge-policy.py
   ```

   The self-test pins the boundary — minor passes, major fails, and the
   scoping rules that stop a narrow override from cancelling a broad bypass
   still hold — so a future edit cannot erode the blocked set unnoticed. Never
   widen the blocked set past `{major}` without a maintainer decision.

---

## check-workflow-permissions.py

### Purpose

Org policy ([#155](https://github.com/tuna-os/.github/issues/155)): every GitHub
Actions workflow must declare a **non-empty top-level `permissions:` block**. A
workflow with none inherits the repository's configured default token scope,
which is almost always broader than the jobs actually need. This is the
least-privilege baseline for the whole org.

The script is intentionally dependency-free: it parses just enough YAML structure
(top-level mapping keys and their indentation) to answer one question — does this
file declare a non-empty `permissions:` mapping at the top level? — so it runs on
the ubuntu runner without a `pip install`.

### What it checks

A **non-empty top-level** `permissions:` block. "Non-empty" means the key exists
*and* has at least one child scope entry (an indented, non-comment line).

| Input | Compliant? |
|---|---|
| No `permissions:` key at all | ❌ |
| `permissions:` with no children | ❌ |
| `permissions: {}` (empty inline map) | ❌ |
| `permissions:` followed only by a comment | ❌ |
| `permissions:` nested under a job (`jobs:…: permissions:`) | ❌ — must be top level |
| `permissions:\n  contents: read` | ✅ |
| `permissions:\n  contents: write\n  packages: write` | ✅ |
| `permissions: read` (inline scalar) | ✅ — a non-empty inline value is read as declaring intent |

It splits multi-document YAML on top-level `---` / `...` and checks each
document, so a file with several documents is checked in full.

### Usage

```bash
python3 scripts/check-workflow-permissions.py .github/workflows [--json] [--quiet]
```

The positional argument is the directory to scan (only the `.yml` and `.yaml`
files directly inside it, sorted). Pass any directory of workflows.

| Flag | Effect |
|---|---|
| `--json` | Emit `{"violations": [<path>, …]}` to stdout instead of human-readable text. Exit code is unchanged. |
| `--quiet` | Emit no output at all; the exit code alone reports the result. |

### Exit codes

| Code | Meaning |
|---|---|
| `0` | Every scanned workflow is compliant. |
| `1` | At least one violation — a workflow missing the block, an empty block, **or** the scanned directory does not exist (reported as `"<dir>: no such directory"`). |

### Output

Human-readable (default):

```
❌ 2 workflow(s) missing an explicit top-level permissions block:
  • .github/workflows/flatpak.yml: missing top-level `permissions:` block
  • .github/workflows/lint.yml: missing top-level `permissions:` block

Add a top-level `permissions:` block declaring the least privilege
each workflow needs, e.g.:
  permissions:
    contents: read
```

Compliant:

```
✅ All workflows declare an explicit top-level permissions block.
```

JSON (`--json`):

```json
{"violations": [".github/workflows/flatpak.yml: missing top-level `permissions:` block"]}
```

### Common failures and how to fix them

| Symptom | Cause | Fix |
|---|---|---|
| `❌ … missing top-level \`permissions:\` block` | The workflow has no `permissions:` key | Add a top-level block with the least privilege each job needs (see below) |
| Block present but still flagged | `permissions:` is empty, `permissions: {}`, or followed only by a comment | Add at least one scope, e.g. `contents: read` |
| Block present but still flagged | `permissions:` is under a job, not the top level | Move it to the top level of the workflow |
| `"<dir>: no such directory"` | The path argument does not exist | Pass the correct workflows directory |

The least-privilege starting point, from `CONTRIBUTING.md`:

```yaml
permissions:
  contents: read
```

`publish-flatpak.yml` needs `contents: write` (checkout + `gh release upload`)
and `packages: write` (GHCR push + index update); `ste-lint.yml` needs
`contents: read`. Declare only what each workflow actually uses.

### When and how to add a new policy

This script answers a single question, so "adding a policy" means changing what
counts as compliant. When you do:

1. Edit the assertion logic in `check-workflow-permissions.py`.
2. Update its self-test, `scripts/test_check_workflow_permissions.py`, to cover
   the new boundary (the existing cases pin: present-with-children, empty
   mapping, empty inline map, comment-only block, job-level-only, inline scalar,
   multi-document, missing-file, and the `--json`/`--quiet` flags).
3. Run the self-test:

   ```bash
   python3 scripts/test_check_workflow_permissions.py
   ```

   (Also runnable as `python3 -m unittest scripts.test_check_workflow_permissions`.)

---

## Copying into project-starter and consumer repos

`project-starter/` is a template **copied verbatim** into every new tuna-os
repository — it is not used by `tuna-os/.github` itself. It ships both scripts
under `project-starter/scripts/`, a `renovate.json`, and a `ci.yml` that already
runs a `workflow-permissions` job. The two copies are **byte-identical** to the
ones in `scripts/` today.

To adopt a check in an **existing** repo:

```bash
# permissions gate — copy the script and the workflow, then run it:
cp ~/.github/scripts/check-workflow-permissions.py <repo>/.github/scripts/
cp ~/.github/workflows/workflow-permissions-check.yml <repo>/.github/workflows/
python3 <repo>/.github/scripts/check-workflow-permissions.py <repo>/.github/workflows
```

The renovate gate is copied the same way; consumers run it against their own
`renovate.json`.

**Caveat:** nothing enforces the byte-identity between `scripts/` and
`project-starter/scripts/`. If you change one, copy it to the other — a drifted
template propagates a stale or wrong policy to every repo scaffolded from it, and
no CI check currently catches the drift.
