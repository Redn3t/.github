# Project Lifecycle

The standard lifecycle every project follows from first commit to retirement.
It governs the path between source code and a running service.

## Phases

| Phase | Purpose | Trigger | Output |
|---|---|---|---|
| Develop | Produce working code | Feature branch | Merged commit |
| Release | Cut an immutable artifact | Git tag `vX.Y.Z` | Versioned artifact in a registry |
| Stage | Rehearse the upgrade against real state | New tag exists | Verified upgrade path |
| Deploy | Promote artifact to production | Stage verification passed | Running service at `vX.Y.Z` |
| Operate | Run, observe, roll back if needed | Continuous | Logs, metrics, rollback decisions |

## Invariants

These hold in every project, every phase. Violation of any one is the root
cause of most update-time breakage.

1. **Repo is the source of truth.** Anything running in any environment is
   fully reconstructible from a tag in the repo. Manual changes on a host do
   not outlive the debugging session that produced them.
2. **Artifacts are immutable.** A version is built once, tagged, published,
   never rebuilt. A change requires a new version.
3. **Pin versions.** Production references an exact version (`vX.Y.Z`). Never
   a moving tag. Never a locally-built artifact.
4. **State boundary.** Persistent data lives in storage that outlives any
   single deployment. The boundary is documented in the README. Runtime
   instances are disposable; state is not.
5. **Build once, deploy many.** The same artifact runs in stage and
   production. No environment-specific builds.

## Phase 1: Develop

Work happens on feature branches. Merges to an integration branch are
integration; merges to the release branch are release candidates. Direct
commits to the release branch are forbidden.

Exit criteria:

- CI passes on the merged commit.
- CHANGELOG `[Unreleased]` section captures user-visible changes.

## Phase 2: Release

Cutting a release is a discrete, named event.

Required actions:

- Promote `[Unreleased]` to `[X.Y.Z]` in the CHANGELOG with the date.
- Tag the release commit `vX.Y.Z`. Tags are annotated and signed.
- Push the tag. CI builds the artifact and publishes it to the registry,
  addressed by the exact version.

After this phase the artifact exists independently of the source tree and can
be deployed without rebuilding.

Required CI behavior:

- The artifact build runs on tag push only, not on every commit.
- The artifact is addressed in the registry by the exact version. No floating
  tags are used as the deployable reference.
- The build is deterministic. Dependencies pinned. Lockfiles committed. Build
  environments pinned by digest where applicable.

## Phase 3: Stage

Stage tests the upgrade path, not the fresh install. The fresh install is the
trivial case; the upgrade is what breaks.

Required actions:

- Restore the most recent production state snapshot to the staging environment.
- Update the staging deployment configuration to point at the new version.
- Apply. Watch the upgrade execute.
- Verify: service starts, state intact, smoke tests pass, observability shows
  expected signals.

The staging environment does not need to be a perfect replica of production.
It needs to exercise the upgrade against real state. A parallel deployment on
the same host with isolated storage is sufficient at small scale.

Exit criteria:

- Upgrade succeeds without manual intervention.
- Rollback path verified: downgrade to the previous version works and state
  is intact.

## Phase 4: Deploy

Promote the verified artifact to production. The deploy itself is mechanical:
change the pinned version in the production deployment configuration, commit,
apply.

Required actions:

- Update the production deployment configuration to the new version in the
  repo.
- Commit: `chore(deploy): promote <project> to vX.Y.Z`.
- Apply on the production host.
- Verify: service starts, state intact, observability shows expected signals.

The deploy commit is the audit trail. Every production version change is a
commit in the repo.

## Phase 5: Operate

Production is a steady state interrupted by deploys, incidents, and
rollbacks. The operator's job is to keep it boring.

Rules:

- Never edit live deployment configuration to fix a problem. The fix path is:
  reproduce in stage, fix in repo, cut a new version, deploy.
- Hotfixes follow the same path on a compressed timeline. They do not skip
  stage.
- Monitor logs, restart counts, and service-specific metrics. Failed deploys
  are caught here, not by user reports.

## Rollback

Rollback is a first-class operation, not a panic recovery. Every deploy must
have a known previous version that can be redeployed.

Procedure:

- Identify the last known good version.
- Update the production deployment configuration in the repo to that version.
- Commit: `chore(deploy): rollback <project> to vX.Y.Z`.
- Apply.

Rollback is safe only if invariants 2-4 hold: the old artifact still exists
in the registry, state lives in storage that the old version can still read,
and the deployment configuration fully describes how to run it.

If rollback requires a state migration, document this in the release notes.
Schema changes that prevent rollback are a release blocker until a
forward-compatible migration path exists.

## Hotfix

A hotfix is an emergency release. The process is identical to a normal
release on a faster clock:

1. Branch from the current production tag, not from an integration branch.
2. Apply the minimum fix.
3. Cut a patch release (`vX.Y.(Z+1)`).
4. Stage. Verify the upgrade path even if the change is one line.
5. Deploy.
6. Merge the hotfix branch back into the integration branches so the fix
   does not regress.

A hotfix that skips staging is a production experiment.

## Retire

When a project is decommissioned:

- Cut a final tag with a CHANGELOG entry noting end of support.
- Archive the repo.
- Stop and remove the production deployment.
- Preserve state for the documented retention period before deletion.

## Decision rules

When in doubt:

- If the change is not in the repo, it does not exist.
- If the artifact is not in the registry, it cannot be deployed.
- If the upgrade was not rehearsed, the deploy is the rehearsal.
- If rollback was not tested, there is no rollback.
