# Org composite actions

Reusable composite actions hosted in `promptLM/.github`. Each
participating repo opts in by adding a thin caller workflow under its
own `.github/workflows/`. Sibling to the three org reusable release
workflows (`release-java-central.yml`, `release-python-pypi.yml`,
`release-ts-npm.yml`); same topology, just at action granularity
rather than whole-workflow granularity. See
[ci-workflow-design.md §1](https://github.com/promptLM/promptlm-release/blob/main/docs/ci-workflow-design.md#1-canonical-workflow-topology-all-repos)
in `promptlm-release` for the full picture.

## `audit-action-pins` — fail CI on unpinned `uses:` refs

Walks every `.github/workflows/*.{yml,yaml}` in the caller and flags
any `uses: owner/repo@<ref>` where `<ref>` is not a 40-char commit
SHA. Skips local refs, docker actions, and (by default) same-org
reusable workflow refs under `promptLM/.github/`. Exits with the
drift count so the calling job fails on any unpinned reference.

**Why pin to SHAs.** Tag refs (`@v4`, `@main`) are movable. A
compromised upstream tag is enough to inject a malicious step into a
release pipeline. OpenSSF Scorecard's `Pinned-Dependencies` check
and SLSA L3 both require commit-SHA pins.

### Caller workflow

```yaml
# .github/workflows/action-pin-audit.yml
name: Action pin audit
on:
  pull_request:
    paths: ['.github/workflows/**']
  schedule:
    - cron: '17 6 * * 1'   # Mondays, 06:17 UTC
  workflow_dispatch:
permissions:
  contents: read
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>  # pin this, obviously
      - uses: promptLM/.github/.github/actions/audit-action-pins@<sha>
```

## `pin-action-shas` — one-shot fixer that opens a PR

Rewrites every unpinned `uses:` ref in the caller's workflows to
`uses: owner/repo@<sha> # <orig-ref>`. The trailing comment preserves
the semver context that Dependabot's `github-actions` ecosystem needs
to recognise the action for future bumps. Idempotent — re-running
on an already-pinned workflow is a no-op.

### Caller workflow

```yaml
# .github/workflows/pin-actions.yml
name: Pin actions to SHAs
on:
  workflow_dispatch:
permissions:
  contents: write
  pull-requests: write
jobs:
  pin:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
      - id: pin
        uses: promptLM/.github/.github/actions/pin-action-shas@<sha>
      - if: steps.pin.outputs.changed != '0'
        uses: peter-evans/create-pull-request@<sha>
        with:
          branch: chore/pin-action-shas
          title: 'chore(ci): pin GitHub Actions to commit SHAs'
          body: |
            Automated SHA pinning via `promptLM/.github/.github/actions/pin-action-shas`.
            Rewrote ${{ steps.pin.outputs.changed }} reference(s); original ref kept as a trailing comment so Dependabot can still propose bumps.
          commit-message: 'chore(ci): pin GitHub Actions to commit SHAs'
```

## `auto-merge-deps` — enable GitHub auto-merge on Dependabot PRs

Triggered on `pull_request_target` from the calling repo. Applies the
safety filters (recognised bot author, no `do-not-merge`/`blocked`/
`breaking-change` label, no major-version bump unless opted in) and
calls `gh pr merge --auto`. **Does not bypass any required check or
approval** — `--auto` just queues the merge to fire when the
calling repo's branch-protection requirements are satisfied.

### Caller workflow

```yaml
# .github/workflows/auto-merge-deps.yml
name: Auto-merge dependency PRs
on:
  pull_request_target:
    types: [opened, synchronize, reopened, ready_for_review, labeled, unlabeled]
permissions:
  contents: write
  pull-requests: write
jobs:
  auto-merge:
    if: github.actor == 'dependabot[bot]' || github.actor == 'renovate[bot]'
    runs-on: ubuntu-latest
    steps:
      - uses: promptLM/.github/.github/actions/auto-merge-deps@<sha>
        # with:
        #   method: squash         # merge | squash | rebase
        #   include-major: 'true'  # default 'false'
```

### Why `pull_request_target`, not `pull_request`?

`pull_request` runs with a read-only token on PRs from forks, which
includes every Dependabot PR. `pull_request_target` runs with the
base-repo's token and so can enable auto-merge. The action never
checks out PR code, so the standard `pull_request_target` security
risk (running attacker-controlled code with elevated permissions) does
not apply — see GitHub's guidance on the safe pattern.

## Rollout

The cross-repo state and the per-repo gaps are catalogued in
[ci-sync-audit-2026-05-30.md §4](https://github.com/promptLM/promptlm-release/blob/main/docs/ci-sync-audit-2026-05-30.md#action-version-pins--the-single-biggest-sweep).
The shape: one tiny caller-workflow PR per participating repo. Order:

1. `audit-action-pins.yml` to every repo — fails CI on drift, makes
   the gap visible without changing behaviour.
2. `pin-action-shas` `workflow_dispatch` per repo, once — closes the
   drift in a single PR per repo. Approve and merge.
3. `auto-merge-deps.yml` to every repo, paired with enabling
   Dependabot's `github-actions` ecosystem in each `.github/dependabot.yml`.
   Steady state: SHA bumps land themselves on green CI.

The fan-out catch-up sweep is small enough that no orchestrator
script is needed — `gh workflow run` per repo is sufficient.
