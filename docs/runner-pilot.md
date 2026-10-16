# GitHub runner pilot

October 14, 2026. The owner selected the GitHub runner before GitLab, using the
private TypeScript repository identified in Homelab. The dedicated runner is now
online; GitLab, mirroring, and native GitLab CI remain planned.

## Deployed allocation

- Ubuntu 24.04 VM, 2 vCPU, 4 GiB RAM, 60 GiB virtual disk, one trusted job at a time.
- Repository-scoped GitHub registration and explicit pilot labels.
- Non-administrative job account; separate network blocking guest access to the
  host and private networks, while allowing public web/package traffic.
- Lower CPU priority, a 3 GiB runner-service memory cap, and a host guard that
  stops the runner VM when host MemAvailable falls below 6 GiB.
- Guarded manual startup requires at least 10 GiB host MemAvailable. VM and network
  autostart remain disabled until peak workload budgets are settled.

The existing VM/container limits were preserved. The preceding sampled 24-hour
minimum available memory was 23.35 GiB; the short deployment/job sample stayed
above 20 GiB. These measurements support this pilot, not an always-on combined
GitLab/runner budget or a guarantee against every rapid load spike.

## Validation and scope

The quality command passed on the local runner: lint, type checking, 11 unit tests,
and build. A deliberate failing test produced a failed GitHub job. The first job
identified missing `libatomic1`; provisioning now includes it. VM stop/start and
runner reconnection and a clean post-restart job succeeded; network denial checks
also passed after restart.
The pilot workflow is in a reviewable PR; default-branch cutover awaits merging.

Existing Node/pnpm versions and quality checks are retained. The workflow uses
local package storage instead of GitHub's cache service and does not persist
checkout credentials. It accepts only private, same-repository code. This
persistent runner is not approved for public or fork contributions.

GitHub still provides scheduling, checks and logs. This step moves compute to the
homelab; it does not create an offline CI coordinator. GitLab CI is a later step.
The VM is disposable, has no independent backup, and its disk/cache use must be
monitored. No GitLab-only source code is stored on it as an authoritative copy.

## Operational ownership

[Homelab's private runbook](https://github.com/melliott18/homelab)
owns exact host addresses, VM identity, network/guard configuration, access,
maintenance and rollback. The sibling checkout's `docs/runbooks/github-runner.md`
is available locally; new documentation remains on working branches until
published. Its `docs/logs/2026-10-14-github-runner.md` holds detailed validation and
private workflow links. Homelab's operational issue owns remaining task status.

This repository keeps the reusable [runner service limits](../configs/github-runner/resources.conf).
Install the drop-in for the registered runner's actual service name inside its VM;
do not apply it to a physical-host service. Private repository identities, runner
authentication files and raw outputs remain outside this public repository.

Next: merge the validated pilot workflow, observe normal usage, settle other
workloads' peak memory budgets, then decide GitLab sizing/access and backup scope.
