# Hetzner runner platform

Org-wide alternative to `runs-on: ubuntu-latest` for long-running CI jobs.
~80% cheaper per run than GitHub-hosted for jobs longer than ~3 minutes.

## When to use this

**Use Hetzner self-hosted runners for:**
- Acceptance / integration test suites (20+ min runs)
- Heavy Maven / Gradle builds with Testcontainers
- Anything that runs frequently and takes more than 3 minutes per run

**Stay on `ubuntu-latest` for:**
- Linting, formatting, fast unit tests (under 3 min)
- Anything that uses GitHub-hosted features the self-hosted runner doesn't provide

The structural break-even is ~3-4 min per run. Hetzner bills by the hour
(rounded up); GitHub-hosted bills per minute. A 1-minute job costs more on
Hetzner than on `ubuntu-latest`.

## What lives where

This `.github` repo hosts the platform:

| File | Role |
|---|---|
| `.github/workflows/hetzner-runner.yml` | **Reusable workflow.** Downstream projects `uses:` this. Three-job pattern: provision → acceptance → teardown. |
| `.github/workflows/hetzner-snapshot-bake.yml` | Nightly bake. Produces the snapshot ID consumed by every caller. One bake serves the entire org. |
| `.github/workflows/hetzner-orphan-sweep.yml` | Daily backstop. Deletes any `gh-runner-*` Hetzner VM older than 6 hours. |
| `workflow-templates/hetzner-acceptance.yml` | Starter template — appears in the "New workflow" picker for any project in the org. |

## Quick start (downstream project)

1. **In your project**, go to the **Actions** tab → **New workflow**.
2. Scroll to the **"By promptLM"** section. Pick **"Hetzner acceptance tests"**.
3. Edit the generated file: change `./scripts/acceptance-tests.sh` to your actual test entry point.
4. Commit. The first run produces a Hetzner runner, executes the test command on it, destroys the runner.

Or write the workflow manually:

```yaml
name: Acceptance tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  acceptance:
    uses: promptLM/.github/.github/workflows/hetzner-runner.yml@main
    with:
      test-command: ./scripts/acceptance-tests.sh
    secrets: inherit
```

That's the entire per-project addition.

## One-time org setup (admin)

The platform needs three pieces of state at the org level. Set them once and
all repos can use the runner.

### Org-level secrets

Under [https://github.com/organizations/promptLM/settings/secrets/actions](https://github.com/organizations/promptLM/settings/secrets/actions):

| Secret | Source | Visibility | Used by |
|---|---|---|---|
| `HCLOUD_TOKEN` | Hetzner Cloud console → Security → API Tokens (Read & Write) | All repos | Reusable workflow, bake, sweep |
| `RUNNER_PAT` | GitHub → fine-grained PAT, "Administration: Read and write" on the target repos | All repos | Reusable workflow |
| `GH_ORG_VAR_TOKEN` | GitHub → fine-grained PAT. Resource owner: **promptLM** org. Under **Organization permissions** → **Variables** → **Read and write**. (No repository permissions needed.) | Only `promptLM/.github` (Selected repositories) | Bake (to publish snapshot ID) |

### Org-level variable

Under [https://github.com/organizations/promptLM/settings/variables/actions](https://github.com/organizations/promptLM/settings/variables/actions):

| Variable | Source | Visibility |
|---|---|---|
| `HETZNER_RUNNER_SNAPSHOT_ID` | Auto-populated by the nightly bake | All repos |

Before the first nightly bake runs, this variable will be unset. The reusable
workflow's fallback installs the baseline at boot using `ubuntu-24.04`
(adds ~3-5 min per first run). Run `hetzner-snapshot-bake.yml` once manually
from the Actions tab to seed the variable; thereafter it stays fresh on its
own.

## Cost model

Per 30-minute acceptance test run:

| Item | Cost |
|---|---|
| Provision job on ubuntu-latest (~1 min) | ~$0.008 |
| Hetzner CX33 VM (1 hour billed, rounded up) | ~€0.009 |
| Teardown job on ubuntu-latest (~30s → 1 min minimum) | ~$0.008 |
| Primary IPv4 prorated | ~€0.001 |
| **Total per run** | **~€0.024** |

vs running the same job on ubuntu-latest directly:

| Item | Cost |
|---|---|
| 30 min × $0.008/min | **~€0.22** |

**Saving: ~89% per run.** At 100 runs/day per workflow that's ~€7.2K/year
saved per workflow.

## Server-type sizing

The reusable workflow defaults to **CX33** (4 vCPU AMD EPYC / 8 GB RAM /
80 GB NVMe / ~€6.49/mo cap, ~€0.009/h). The CX line is the current
generation; it's ~3× cheaper than the older CPX line at matched RAM/CPU
because Hetzner repriced the new generation in April 2026.

Override per call:

- **Lighter** workflows: `server-type: cx22` (2 vCPU / 4 GB) for builds/lint
- **Heavier** workflows: `server-type: cx43` (8 vCPU / 16 GB / 160 GB) or `cx53` (16 vCPU / 32 GB / 320 GB). These are the current-gen CX SKUs that replaced `cx42`/`cx52` after Hetzner's 2026 rename — the old `2`-suffix names alias to deprecated Intel SKUs and Hetzner rejects them (see Troubleshooting).
- **Older CPX line** still works if a workflow needs the bigger SSD: `cpx31` (160 GB), `cpx41` (240 GB)
- **ARM** workflows (~half the price but Docker images must be arm64-clean):
  `server-type: cax21` or `cax31`. Confluent Platform images (`cp-kafka`,
  etc.) are amd64-only and will NOT work on ARM; stick to CX for those.

## What's baked into the snapshot

- Ubuntu 24.04 LTS
- Docker Engine (latest stable)
- JDK 17 + JDK 21 (Eclipse Temurin)
- Node 22 LTS + npm/pnpm/yarn
- Maven 3.x, Gradle 8.x
- Common CLI: `git curl wget jq unzip tar gzip make htop vim`
- GitHub Actions runner binary
- `runner` user in `docker` group

If a project needs niche tools (Android SDK, GraalVM, specific Docker images
pre-pulled), use the Cyclenerd `pre_runner_script` input through the
reusable workflow's `pre-runner-script` input (currently not exposed —
add if needed) OR install them as the first step of the acceptance job.

## Troubleshooting

**A workflow run leaves a Hetzner VM behind.** The daily orphan sweep
(`hetzner-orphan-sweep.yml`) catches anything older than 6 hours with a name
matching `gh-runner-*`. To force a sweep, run that workflow manually from the
Actions tab. Worst-case orphan cost is ~€0.60 (24 hours of CX33).

**The provision job hangs.** Hetzner API outages or transient SSH boot
failures are the usual cause. The Cyclenerd Action retries internally
(default 360 × 10s = 1 hour). The `provision` job's 10-min timeout kills it
before that completes.

**`runs-on: ${{ needs.provision.outputs.label }}` fails to find a runner.**
The runner registration finished but the label didn't propagate yet. GitHub
typically resolves in seconds; if it's persistent, check that the `RUNNER_PAT`
has the right scope on the calling repo.

**ARM (CAX) workflow fails Docker pulls.** Some Docker images are amd64-only
and silently fall back to qemu emulation (slow + flaky) or fail outright.
Switch back to CPX or pin to multi-arch image variants.

**Provision fails with Hetzner 422 "server type 106 is deprecated" on `cx42`/`cx52`.**
The CX line was renamed in 2026 — the `2`-suffix SKUs (`cx42`/`cx52`) alias to
the older Intel generation and Hetzner now rejects them. Use the new `3`-suffix
names instead: `cx43` (8 vCPU / 16 GB) and `cx53` (16 vCPU / 32 GB). The smaller
end of the line (`cx22`, `cx32`, `cx33`) is still accepted.

## Roadmap / known limits

- Runner registration uses a fine-grained PAT (`RUNNER_PAT`). Migration to a
  GitHub App is a future enhancement; current scope is fine for single-org use.
- The reusable workflow does not yet expose `pre_runner_script` as an input
  — add via a small PR if a project needs per-call setup. Most projects
  don't.
- Single-region (fsn1) by default. Multi-region failover is not implemented.
  Falkenstein has good uptime; the orphan sweep is the only critical
  scheduled work and it's idempotent.

## Source

This platform is the output of a design dossier at
`promptics-hetzner-build-runner` (the repo where it was researched and
prototyped). The Cyclenerd Marketplace Action does the actual VM lifecycle:
[Cyclenerd/hcloud-github-runner](https://github.com/Cyclenerd/hcloud-github-runner)
(Apache-2.0). All Hetzner billing assertions cited from
[docs.hetzner.com/cloud/billing/faq](https://docs.hetzner.com/cloud/billing/faq/).
