# Project state

Updated October 14, 2026, America/Los_Angeles.

The owner requested a 24/7 GitLab VM on the Homelab server, one-way copies of
GitHub repositories subject to exceptions, separate GitLab-only private projects,
and local CI that does not depend on hosted compute-minute allowances. The owner
selected the GitHub runner first, using the private project identified in Homelab.
The isolated runner VM is deployed; see [pilot status](docs/runner-pilot.md).
GitLab and mirroring remain future work.

## Design direction

- GitLab Free in a dedicated Ubuntu VM; an outbound scheduled worker mirrors
  GitHub branches, tags, reachable history, and separately transferred LFS objects.
- GitLab-only projects are outside the mirror namespace and worker permissions.
- Local GitHub Actions runners preserve existing workflows and PR checks during
  migration. Local GitLab Runner supports GitLab-only projects and fully local CI.
- Runners execute in separate VMs from GitLab and host administration. Public
  untrusted contributions require a separate disposable execution design.
- Sibling repositories with reciprocal documentation; submodule decision deferred.

See the [plan](docs/plan.md) and [dated assessment](docs/assessment-2026-10-14.md)
for VM proposals, pilot selection, validation criteria, and open decisions.

## Completed assessment

October 14 read-only inspection found 30.28 GiB host RAM, at least 23.35 GiB
available across the preceding sampled 24 hours, and 1.19 TiB free disk space.
Existing container limits still permit more peak demand than a combined GitLab
and runner allocation can accommodate; reconcile workload budgets before rollout.

The GitHub scan covered 44 owned repositories and found 14 default-branch workflow
files in eight repositories. Recommended pilot: the private TypeScript project
identified in Homelab's private CI inventory. Begin with a 2-vCPU/4-GiB runner,
then a 4-vCPU/8-GiB GitLab trial; defer the separate GitLab runner until validation.
The runner allocation is now deployed; the GitLab allocation remains a proposal.
Both projects now contain reciprocal
links, and detailed private evidence remains outside this public repository.

## Ownership and recovery dependencies

[Homelab](https://github.com/melliott18/homelab) owns server/VM/network/storage
changes. Read `../homelab/AGENTS.md` and `../homelab/PROJECT_STATE.md` before host work.
Its existing recovery and host-autostart work must be coordinated with GitLab;
local snapshots on the same physical disk do not provide independent backups.

## Next steps

1. Validate and merge the pilot workflow, then observe normal job/resource behavior.
2. Confirm GitHub owner scope and repository exceptions; inspection visibility is
   not authorization to mirror every accessible organization or collaborator repo.
3. Define private hostname/access, scoped credentials, backup destination,
   deletion/force-push behavior, and runner trust boundary before deployment.
4. Reconcile peak workload memory budgets before adding GitLab or another runner.

Repository visibility and branch protection remain unchanged. Infrastructure and
pilot workflow changes are recorded separately from the initial assessment.
