# T001 — Record initial supported deployment assumptions

**Status:** Ready

Source task: `TELCHAR_IMPLEMENTATION_PLAN.md` — T001

## Scope

Create one architecture decision record establishing the initial Telchar deployment assumptions. Documentation only. Do not implement runtime behavior, project scaffolding, Nix configuration, PostgreSQL schema, or service configuration.

## Dependencies

None.

## Required outcome

Create an ADR recording these decisions:

- Linux-first initial support.
- Initial deployment is single-active: one Telchar daemon owns scheduling, durable state, gateway-store coordination, and backend reconciliation.
- OpenSSH provides network-facing SSH ingress.
- A restricted forced command starts one `telchar serve-stdio` frontend per connection.
- The frontend communicates with the daemon through authenticated local IPC.
- PostgreSQL is the durable control-plane database.
- PostgreSQL does not imply multiple active schedulers or Telchar high availability.
- Persistence is exposed to Telchar through domain-specific state operations with explicit transaction ownership.
- Database interchangeability is not an initial goal.
- TOML is the initial human-readable service configuration format.
- The gateway runs on a dedicated host or VM whose system Nix store is controlled by Telchar and not shared with unrelated workloads.
- Authenticated clients initially share one mutually trusted store domain; hostile client multi-tenancy and per-path client authorization are deferred.

Use an evergreen ADR title and domain-focused names. Do not describe these choices as new, improved, legacy, or replacements for historical behavior.

## Required reading

Read before editing:

- `telchar-design.md`
- `TELCHAR_IMPLEMENTATION_PLAN.md`, especially T001 and the deferred-work section

## Checklist

- [ ] Read required documents completely enough to identify every section affected by T001.
- [ ] Establish the red condition by recording contradictions, omissions, or ambiguous wording between the intended ADR and `telchar-design.md`.
- [ ] Choose the repository ADR location and naming convention without creating speculative ADR tooling.
- [ ] Write the deployment-assumptions ADR.
- [ ] Update `telchar-design.md` only if needed to remove a real contradiction with the ADR.
- [ ] Add a repository documentation-consistency check appropriate to this documentation-only task. Keep it small; a focused script or command is sufficient.
- [ ] Run the consistency check and record exact evidence below.
- [ ] Mark T001 complete in `TELCHAR_IMPLEMENTATION_PLAN.md` only after verification passes.
- [ ] Commit the logical changeset with `jj`.
- [ ] Confirm the working copy contains no unrelated changes.

## Red condition

Before writing the ADR, produce a concrete list of any contradiction, omission, or ambiguous deployment assumption found in `telchar-design.md`. If no contradictions exist, record that result and identify the missing independent ADR as the failing acceptance condition.

Do not manufacture a failing code test for a documentation-only decision. The initial failure is absence of the required ADR and any documentation inconsistency found by the focused check.

## Verification contract

The task must leave one exact command that an external monitor can rerun from repository root in a fresh shell. It must verify at least:

- The ADR exists.
- The ADR records every required outcome listed above.
- `telchar-design.md` does not contradict those decisions.
- T001 is checked in `TELCHAR_IMPLEMENTATION_PLAN.md`.
- No stale SQLite-as-database choice remains; intentional statements rejecting SQLite substitution are allowed.
- PostgreSQL is not described as automatically providing Telchar scheduler high availability.

Passing output must be clean. Expected failures used during the red step must be captured and described, not left in final verification output.

## Evidence

Record during loop execution:

### Red

- Command:
- Working directory:
- Exit status:
- Output summary:
- Contradictions or missing artifact:

### Green

- Command:
- Working directory:
- Exit status:
- Output summary:
- ADR path:
- Design changes, if any:

### Final verification

- Exact command:
- Required environment:
- Exit status:
- Output summary:

### Changeset

- `jj` change ID:
- Commit ID:
- Description:

## Completion gate

Do not emit `<promise>COMPLETE</promise>` unless all conditions hold:

- ADR exists and records every required decision.
- Documentation consistency verification passes from repository root.
- T001 is checked in `TELCHAR_IMPLEMENTATION_PLAN.md`.
- Verification evidence is recorded in this file.
- Changes are committed with `jj`.
- Working copy has no unrelated changes.
- Final verification command can be rerun externally from the same worktree.

If work exposes a materially different deployment architecture, stop. Add a decision task to the master plan and report the blocker rather than silently widening T001.
