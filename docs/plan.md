# Git mirroring and local CI plan

Planning baseline: October 14, 2026. The initial read-only assessment is complete.
The owner then selected the GitHub runner VM and private CI pilot; see
[deployment status](runner-pilot.md). GitLab and native GitLab CI remain proposed.

## Outcomes

1. Keep a private homelab copy of code pushed to selected GitHub repositories,
   including new repositories within configured owner scope unless excluded.
2. Host private GitLab-only repositories without publishing them to GitHub.
3. Run CI locally for private projects without a provider's monthly compute-minute
   ceiling, while preserving useful existing GitHub pull-request checks.
4. Recover repository data, configuration, and credentials after a service or host
   failure, with tested backups independent of the host's physical disk.

Local commits become eligible for mirroring only after a push to GitHub. Mirroring
copies original Git objects and commit IDs; it does not create duplicate commits.

## Ownership and topology

| Component | Responsibility | Project owner |
| --- | --- | --- |
| Physical host, KVM/libvirt, VM disks/network, power policy | Stable infrastructure and capacity | Homelab |
| `gitlab-01` VM | GitLab Linux package, repositories, database, CI coordination, artifacts | Homelab provisions; Git Mirror configures service |
| Mirror service in GitLab VM | Outbound GitHub discovery/fetch and scoped local pushes | Git Mirror |
| `ci-github-01` VM | Existing GitHub Actions jobs | Homelab provisions; Git Mirror configures runner |
| `ci-gitlab-01` VM | Native GitLab jobs | Homelab provisions; Git Mirror configures runner |
| Independent backup destination | Storage and access outside the GitLab host disk | Homelab |
| GitLab backup/restore and CI retention | Application-consistent recovery and bounded disk use | Git Mirror |

Provision incrementally after the [capacity assessment](assessment-2026-10-14.md).
Runner VMs are not assumed to have dedicated physical CPU cores. The Pi retains
its NAS role. Keep GitLab outside the experimental Kubernetes cluster.

Use a private HTTPS hostname and controlled LAN/VPN access. The mirror worker and
GitHub runner make outbound connections; public ingress is not required for the
polling design. Define and verify IPv4/IPv6 and guest-to-host access rules during
deployment instead of assuming that NAT or Tailscale alone isolates jobs.

## Mirroring policy

GitHub is authoritative for `github-mirrors/<owner>/<repository>`. Ordinary users
do not push independent changes to those destinations. GitLab-only projects use a
separate namespace and never enter the worker's destination scope.

- Begin with one explicit pilot repository. Later discover repositories using
  paginated API calls within explicitly configured GitHub owners and credentials.
- Keep repository exceptions, scope, destination mapping, and sync interval in
  version-controlled configuration. Proposed interval: five minutes.
- Authenticate with read-only GitHub access and a GitLab identity limited to the
  mirror namespace. Store secrets outside Git and redact them from logs.
- Map source repository identity as well as name so renames and transfers need
  reconciliation rather than creating or deleting arbitrary destinations.
- Transfer branches and tags with explicit refspecs. Do not blindly copy
  GitHub-specific pull-request refs or GitLab internal refs with `push --mirror`.
- Transfer Git LFS objects explicitly and verify a fresh LFS checkout. Discover
  submodule dependencies separately; copied `.gitmodules` URLs retain their original
  destinations unless clone configuration provides a deliberate local mapping.
- Create mirrors private by default. Missing permissions, API errors, deleted
  sources, and excluded repos must never automatically delete destination projects.
- Start with branch/tag pruning and destructive ref replacement disabled. Before
  enabling exact deletion/force-push propagation, decide retention and demonstrate
  recovery. Until then, surface divergent/stale refs instead of claiming exact sync.
- Use per-repository locks, bounded retries, last-success timestamps, and alerts
  for stale mirrors. A failed source fetch must not trigger destination pruning.

GitHub issues, PR discussions, release assets, Actions artifacts, and settings are
not Git objects and are outside the first rollout. A wiki is a separate Git
repository. Decide metadata archival separately. Periodic branch-tip polling can
miss commits whose only references were deleted or force-rewritten between polls;
this is a mirror of observed reachable history, not a complete GitHub event archive.

GitLab's built-in pull mirroring requires Premium/Ultimate, so the free design
uses an external scheduled worker. Built-in pull mirroring also has its own
deletion and scheduling semantics; do not assume it is interchangeable.

## Local CI

### Existing GitHub workflows

Use local GitHub Actions runners to retain GitHub triggers, PR checks, and results
while moving compute to the homelab. Audit every job's `runs-on`, tools, services,
actions, secrets, artifact/cache use, OS matrix, and permissions before changing
runner selection. GitHub-hosted images include tools a new VM will not have.

Personal-account repository runners are registered per repository; organization
runners can serve permitted organization repositories. Do not assume a single
personal-account registration covers every repository. Start with a pilot-specific
registration and labels. Local execution still relies on GitHub for scheduling
and any GitHub-hosted artifacts/caches used by the workflow.

### Native GitLab CI

Use `.gitlab-ci.yml` for GitLab-only projects and projects transitioning to local
CI coordination. Reuse shared build/test scripts to keep the GitHub and GitLab
definitions thin. GitLab does not execute `.github/workflows/*.yml` directly.

Mirror updates can trigger pipelines only when the destination has suitable CI
configuration and push/pipeline rules. Keep CI configuration upstream or use an
explicit external configuration; do not add GitLab-only commits to mirror branches.
Choose one trigger mechanism to avoid duplicate pipelines. Report GitHub PR
statuses and test synthetic merge commits only through an explicitly designed
integration; a mirrored branch pipeline does not automatically supply either.

### Execution boundaries

- Keep runner VMs separate from GitLab, backups, and privileged host maintenance.
  Jobs receive no host-admin key, mirror credential, or broad tailnet identity.
- Start with one concurrent job per runner and bounded CPU, RAM, disk, duration,
  cache, logs, and artifact retention. Coordinate capacity across runner VMs.
- Prefer clean disposable job environments. Containers are not a sufficient trust
  boundary for hostile code; Docker daemon access or privileged execution can
  compromise the runner VM. VM network restrictions must remain effective.
- Start with reviewed owner-controlled revisions. Public fork PRs stay on hosted
  CI or a separately designed disposable runner pool with no sensitive access.
- Separate deployment credentials and privileged hardware/kernel tests from
  routine unit tests. Do not run arbitrary repository code on the hypervisor.
- A Linux VM cannot replace macOS jobs; inventory platform requirements first.

Current official documentation lists GitHub self-hosted execution as free;
GitHub artifact/cache storage still has allowances. GitLab Self-Managed compute
quotas are disabled by default. Local capacity, disk retention, and configured
job timeouts remain real limits. Recheck provider terms before rollout.

## Backup and availability

Back up GitLab repositories, database, LFS/artifacts as selected, configuration,
and secrets using supported procedures. Keep an independent encrypted copy and
test restoration to a separate target using compatible GitLab version/edition.
Source deletion and force pushes can propagate into mirrors, so mirroring alone
does not preserve past states. GitLab-only code has no GitHub recovery copy.

Verify service/VM autostart, controlled host reboot, power-loss recovery, private
access, and failure alerts. Coordinate with Homelab's existing recovery tasks.
No physical reboot, firmware change, or outage test is part of this assessment.

## Rollout and acceptance

| Stage | Work | Acceptance |
| --- | --- | --- |
| Planning | Cross-project docs, capacity and workflow inventory | Dated evidence, proposed VM budget, named pilot, open decisions |
| GitHub runner pilot | Dedicated runner VM; selected trusted workflow | Same tests/check names, known dependencies, bounded cleanup, repeatable clean run; no fork execution |
| GitLab pilot | Private GitLab VM and one mirror | Branch/tag commit IDs match; LFS/submodule checkout where applicable; retry after outage |
| Local CI pilot | Translate pilot test job to GitLab | Equivalent tests and reports, deliberate triggers, resource measurements |
| Recovery | Independent backup and restore; scheduled restart validation | Fresh checkout from restored GitLab; GitLab-only test repository recovered |
| Expansion | Owner discovery, exceptions, more workflows | Newly eligible repos included, exclusions respected, stale mirrors/runners alerted |

Before provisioning, select source owners/exceptions, private hostname/access,
credential approach, initial VM budget, backup destination/retention, and the pilot
trust policy. Before exact mirroring, select ref deletion/force-push policy.

## Project relationship

Keep this repository and [Homelab](https://github.com/melliott18/homelab) as sibling
checkouts with reciprocal README/agent-guide links. Homelab owns its operational
issues and infrastructure evidence; this project owns service behavior and CI.
A future `services/git-mirror` submodule could pin a deployment revision. It would
retain independent history and require explicit updates; it would not merge this
repository's history into Homelab. This decision remains deferred.

## References

Official documentation reviewed October 14, 2026:

- [GitLab installation requirements](https://docs.gitlab.com/install/requirements/)
- [GitLab Linux package](https://docs.gitlab.com/install/package/)
- [GitLab repository mirroring](https://docs.gitlab.com/user/project/repository/mirror/)
- [GitLab pull mirroring](https://docs.gitlab.com/user/project/repository/mirror/pull/)
- [GitHub Git and LFS mirroring](https://docs.github.com/en/repositories/creating-and-managing-repositories/duplicating-a-repository)
- [GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)
- [GitHub self-hosted runners](https://docs.github.com/en/actions/concepts/runners/self-hosted-runners)
- [GitHub runner security guidance](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners)
- [GitLab compute minutes](https://docs.gitlab.com/ci/pipelines/compute_minutes/)
- [GitLab CI migration](https://docs.gitlab.com/ci/migration/github_actions/)
- [GitLab runner security](https://docs.gitlab.com/runner/security/)
- [GitLab backups](https://docs.gitlab.com/administration/backup_restore/backup_gitlab/)
- [Git submodules](https://git-scm.com/docs/gitsubmodules)
