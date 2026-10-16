# Git Mirror

Plan and operate a private, locally hosted GitLab server that mirrors selected
GitHub repositories and hosts GitLab-only code, with local CI runners for testing.
GitHub remains authoritative for mirrored repositories. GitLab-only repositories
have no automatic outbound mirror to GitHub.

Status: the first isolated GitHub runner VM is deployed for a private CI pilot.
GitLab and mirroring remain planned. See the [runner pilot](docs/runner-pilot.md).

## Start here

- [Shared project registry](https://github.com/melliott18/homelab/blob/main/docs/reference/projects.md) and
  [workflow](https://github.com/melliott18/homelab/blob/main/docs/reference/project-workflow.md): ownership, context
  lookup and handoffs (private Homelab checkout required).
- [Project state](PROJECT_STATE.md): current findings and next steps.
- [Mirroring and CI plan](docs/plan.md): scope, architecture, policy, and rollout.
- [Capacity and CI assessment](docs/assessment-2026-10-14.md): VM sizing and pilot selection.
- [Agent guide](AGENTS.md): maintenance and cross-project responsibilities.
- [Decisions](docs/decisions/README.md) and [results](docs/logs/README.md).

## Related project

[Homelab](https://github.com/melliott18/homelab) owns the physical host, VM
provisioning, network boundaries, storage, and independent backup destination.
Its local checkout is normally `../homelab`; read its `AGENTS.md` and
`PROJECT_STATE.md` before host work. The repository is private and requires access.

This project owns GitLab configuration, mirroring policy and automation, runner
configuration, CI migration, and service-level recovery procedures. Keep the
repositories separate with explicit cross-references for now. A Homelab submodule
is undecided; no submodule has been added.

This repository is public. Keep credentials, raw API/host output, private repository
names and workflow contents in ignored `local/` storage or appropriate private
Homelab records. Publish only reviewed summaries.
