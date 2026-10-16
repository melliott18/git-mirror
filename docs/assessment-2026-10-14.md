# Capacity and CI assessment

October 14, 2026. Recommendation: start with a private TypeScript CI pilot on one
small GitHub runner VM, then introduce GitLab and native GitLab CI incrementally.
Current measured headroom is sufficient for a small trial, but existing workload
memory limits must be reconciled before treating the combined VM plan as reserved
capacity. No VM or workflow was changed during the assessment.

Private pilot identity, workflow evidence, and detailed host observations are in
Homelab's `docs/logs/2026-10-14-git-mirror-capacity.md` and
`docs/reference/git-mirror-ci-inventory-2026-10-14.md`. The local checkout is
`../homelab`; [remote access](https://github.com/melliott18/homelab) is restricted.
This public document intentionally uses aggregate resource data and public examples.

## Host capacity

Read-only inspection: October 14, 15:24–15:28 PDT (22:24–22:28 UTC). The target
identity was verified before resource queries. Existing Prometheus supplied the
preceding 24 hours; its memory series contained 2,880 samples at 30-second cadence.

| Resource | Observation | Planning implication |
| --- | --- | --- |
| CPU | 8 physical cores, 16 logical CPUs; 94–98% idle in five two-second intervals | Small Linux CI pilot fits the sampled load; no dedicated cores reserved |
| Recent CPU load | Maximum sampled five-minute average across the host: 4.87% over 24 hours | Does not measure instantaneous peaks or future AI/benchmark activity |
| RAM | 30.28 GiB visible; 23.79 GiB available at inspection; 23.35 GiB minimum available over 24 hours | Available includes reclaimable cache; avoid adding it to container memory totals |
| Swap | 2.01 GiB occupied out of 8 GiB; 24-hour sampled peak 2.28 GiB | Not proof of current thrashing; short sample had almost no swap I/O |
| Memory pressure | Very low sampled pressure; no increase in host OOM-kill counter over 24 hours | Supports a bounded trial, not guaranteed simultaneous workload capacity |
| Storage | 1.19 TiB available on the single SSD-backed filesystem holding VM images | Room for initial disks; no independent redundancy or backup implied |
| Existing VM | One running guest configured for 2 vCPU and 4 GiB RAM | Preserve allocation and workloads |
| Existing containers | One 20 GB decimal memory cap (18.63 GiB); another uncapped; neither CPU-quota limited | Current utilization is not a safe worst-case budget |

The capped container's roughly 10 GiB memory charge was mostly file cache. Do not
subtract that full charge again from Linux `MemAvailable`. Conversely, its cap is
a real potential future demand: 18.63 GiB + the existing 4 GiB guest + 12 GiB of
proposed new guests already exceeds 30.28 GiB, before host overhead and the uncapped
container. Owner workload budgets, scheduling/exclusivity, or additional RAM are
needed before an always-on combined rollout can claim adequate peak capacity.

## Proposed VM budget

These are starting allocations to validate, not reservations or deployment results.

| Phase | Guest | vCPU | RAM | Virtual disk | Initial concurrency |
| --- | --- | --- | --- | --- | --- |
| 1 | `ci-github-01` | 2 | 4 GiB | 60 GiB | One trusted Linux job |
| 2 | `gitlab-01` | 4 | 8 GiB | 120 GiB | Serial initial imports/mirroring |
| 3 | `ci-gitlab-01` | 2 | 4 GiB | 60 GiB | One job; add after capacity review |

Use an Ubuntu release supported by both the selected GitLab package and runner
dependencies; Ubuntu 24.04 LTS is the initial candidate. Keep runner labels explicit
and do not assume the hosted `ubuntu-latest` image's toolset exists in the VM.
No GPU passthrough is needed for the pilot.

Phase 1 adds 4 GiB. Phases 1–2 together add 12 GiB. All three add 16 GiB and
240 GiB of virtual disks, in addition to existing workloads. Allocate incrementally,
budget actual disk growth and build caches, and preserve host/service overhead.
Running only one CI job across both runner systems initially prevents accidental
parallel load, but stopped/idle guest memory and container demand still require
an explicit budget.

GitLab's general single-node baseline is 8 vCPU/16 GB; the proposed 4 vCPU/8 GiB
pilot is a deliberate smaller trial, not that baseline. The current requirements
allow memory-constrained operation, but the separate low-memory guide's very small
repository assumptions do not cover every repository in this inventory. Measure
imports, background jobs, and UI responsiveness; increase resources or reduce scope
if the trial cannot meet them. [Requirements](https://docs.gitlab.com/install/requirements/),
[small-installation guidance](https://docs.gitlab.com/omnibus/settings/memory_constrained_envs/).

Before combined provisioning, reconcile the existing container limits under
Homelab's workload review, choose minimum host headroom, and verify available
memory immediately before allocation. Do not reduce another workload's limits or
stop it as an incidental GitLab installation step. Independent backup and host
restart recovery remain separate prerequisites for relying on GitLab-only code.

## GitHub workflow inventory

Inspected 44 repositories owned by the authenticated account: 17 public and 27
private. Eight repositories have 14 YAML workflow files on their default branches.
Two additional repositories have registered/historical or generated Actions
workflows without default-branch workflow files. Current organization membership
lookup returned none; collaborator repositories and other accounts were not scanned.

All 44 Actions workflow-list calls succeeded. Thirty-six default workflow-directory
queries returned 404; existing run metadata and selected active branches were used
to distinguish generated/history-only workflows from files on default branches.
The latest-run branches of `cshell` and one private repository were additionally
inspected. This is not an exhaustive scan of every branch. No workflow, installer,
or test script was executed.

| Workload category | Finding | Migration implication |
| --- | --- | --- |
| Private TypeScript pilot | One Linux job: lint, types, unit tests, build; current default SHA has a historical successful hosted run | Best controlled first runner and later GitLab CI pilot |
| Other private projects | Python quality gates, Node applications, static infrastructure validation, mixed Python/Rust/frontend/Docker jobs | Migrate in increasing complexity; retain named details privately |
| `cshell` | GCC/Linux and Clang/macOS; locale/PTY/sanitizer tests, Docker root/device operations; active branch adds filesystem qualification | Later Linux pilot; preserve macOS coverage; isolate privileged tests |
| `CogniStore` | Python 3.10–3.14 matrix, multiple service containers, Compose, package builds, Kubernetes acceptance and Terraform checks | Later capacity qualification, not the first runner smoke test |
| Public portfolio | GitHub-generated Pages build/deployment | Hosting integration, not an ordinary test workflow to relocate blindly |

The sampled [CogniStore CI run](https://github.com/melliott18/CogniStore/actions/runs/37078608181)
had 13 successful jobs totaling 6,427 seconds (107.1 job-minutes); its longest job
took 22m55s. Those hosted job intervals are not CPU-utilization measurements or a
local-runtime prediction. A single local runner serializes that parallel workload.
Its separate Kubernetes acceptance run failed, so the successful CI run does not
establish that all workflows pass.

The sampled [cshell run](https://github.com/melliott18/cshell/actions/runs/38611310273)
had three failed jobs. Resolve or characterize baseline failures before judging
local-runner equivalence. The additional active-branch workflow differs from main;
review the exact migration commit rather than assuming default-branch settings.

Selected private failed runs contain GitHub annotations identifying a billing or
spending-limit startup block. These are not evidence that test code failed. The
annotation does not distinguish payment failure from a spending-limit issue, and
this assessment did not change or inspect account billing settings.

## Pilot and acceptance

Recommended first repository: the private TypeScript project identified in the
private inventory. Its one `check` job uses Node 22.13.0 and pnpm 11.10.0 and runs
lint, type checking, Vitest, and a TypeScript build through `pnpm check`. A hosted
run at the current default commit succeeded in 26 seconds of job time on August 5
PDT (August 6 UTC). That is a historical baseline, not a fresh verification.

For the first runner trial:

1. Register a repository-scoped runner in a dedicated VM with a specific Linux
   label. Use a reviewed commit and manual pilot trigger; keep unrelated workflows
   and other repositories off this runner.
2. Install/audit the required tools, preserve lockfile-based installation and
   read-only repository permissions, and disable credential persistence in checkout.
   Add a 15-minute job timeout and one-job concurrency.
3. Run the same quality commands from a clean checkout and empty dependency cache;
   repeat to check cleanup and cache behavior. Validate a deliberately failing test
   on a temporary pilot branch reports failure correctly.
4. Measure peak memory, disk growth, elapsed time, cleanup, and denied access to
   private host/backup services. Verify safe queue behavior when the runner is offline.
5. Mirror the same repository privately to GitLab, then port the thin CI definition
   around the same commands. Keep configuration upstream or explicitly external;
   do not diverge the mirrored branch to add CI.
6. Move a billing-blocked private project next. Retain macOS jobs and defer public
   untrusted PR execution until a separate disposable trust boundary exists.

The pilot repository is recommended, not yet migrated or made public. No runner
registration, token creation, workflow dispatch, remote push, or visibility change
was performed. Raw evidence is ignored locally and checksum-indexed; reviewed
private details are in Homelab.
