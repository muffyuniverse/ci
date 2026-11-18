# Shared CI: agent operating instructions

This repository runs workflows for multiple projects. It is shared infrastructure: a run, branch,
workflow or artifact may belong to another person or agent. Do not cancel, rerun, overwrite, push
or delete someone else's work. Ask or coordinate if ownership or intent is unclear.

## Before dispatching

1. Inspect active and queued runs, their workflow names and run IDs:
   `gh run list --repo muffyuniverse/ci --limit 100 --json databaseId,workflowName,event,status,headBranch,createdAt --jq '.[] | select(.status != "completed") | [.databaseId,.workflowName,.event,.status,.headBranch,.createdAt] | @tsv'`.
   Inspect a specific run with `gh run view RUN_ID --repo muffyuniverse/ci`.
2. Coordinate with any other agent intending to dispatch **the same workflow**. A check of the
   run list followed by a dispatch is not an atomic lock: two agents can both see an empty list.
   Do not dispatch until the other agent has finished or you have agreed on an order. The safest
   default for Build and release is one dispatch at a time, including queued runs.
3. Different workflows have independent concurrency groups, but they share runner capacity and
   may queue each other. Respect ongoing expensive work. If a run is queued, wait; do not cancel
   another run to obtain runners.
4. Dispatch once, record the returned/discovered **exact run ID**, and monitor that ID. Confirm
   the checked-out project revision and artifact metadata, not merely a run number or the newest
   run. Do not associate a run with your candidate just because its timestamp is recent.

## Pechka: Build and release

`build-and-release.yml` accepts `repository_dispatch` of type `project-push`, with
`client_payload.sha` set to an accessible commit in `muffyuniverse/pechka-ai`. Push the candidate
to an appropriate project branch first; do not move project main merely to make a candidate
available. After coordinating with other dispatchers, the explicit dispatch is:

```
gh api repos/muffyuniverse/ci/dispatches -f event_type=project-push \
  -f 'client_payload[sha]=FULL_PROJECT_COMMIT_SHA'
```

The dispatch API does not return a run ID. Find and record the corresponding run, verify its
checkout log says the full candidate SHA, and then use that ID for monitoring and downloads. If
ownership cannot be established, ask rather than guessing. `tools/ci.sh --run RUN_ID` and
`tools/wait-runners.sh RUN_ID` in Pechka can watch a known run. `tools/ci.sh --push` and
`tools/ship.sh` can push/dispatch and do not themselves coordinate with other agents; do not use
them blindly. Verify sealed bundle contents, source SHA and signature before delivery.

**Important:** this workflow currently uses `group: ${{ github.workflow }}-${{ github.ref }}` and
`cancel-in-progress: true`. Repository dispatches run on the CI repository's default ref, so a new
Build and release dispatch can cancel an active run in that same group. A manual dispatch can
collide too. Wait for both running and queued work and coordinate *before* dispatch; never send a
second dispatch merely to see whether the first worked. Retrying failed jobs is also an action on
that exact run: only its owner should do so, after checking the failure.

## Motion and probe workflows

`motion-design-render.yml`, `motion-design-capture.yml`, `tool-probes.yml` and
`tool-probes-heavy.yml` use a run-specific concurrency group and do not intentionally cancel one
another. Their `push` triggers are path-filtered to their own workflow file; they also support
manual dispatch. Editing one of those workflow files triggers its push workflow: check for and
coordinate with whoever is working on it before pushing. A documentation-only change does not
match those path filters. Their jobs currently check out motion-design's default branch, not an
explicit project SHA; record the actual checked-out revision and do not claim a pinned result
without verifying it.

## Workflow changes and artifacts

- Namespace each workflow and its concurrency group. Do not add an unfiltered `push` trigger in
  this shared repository. Prefer run-specific groups with `cancel-in-progress: false` for new
  independent workloads; explicitly coordinate any intentional serialization.
- Do not change Build and release in this repository independently of its rendered source in
  Pechka. Their copies are checked for consistency. Inspect cross-project effects before altering
  a workflow's triggers, checkout ref, runner consumption, permissions or artifact names.
- Keep artifact names run-scoped where possible. Download by exact run ID. The Pechka bundle is
  sealed; motion and probe artifacts may be readable by anyone with access to this public repo.
  Do not put credentials or private content in logs, screenshots or unsealed artifacts.
- Do not enable SSH debugging, delete runs/artifacts, cancel active runs, or force-push shared
  branches without the relevant owner's explicit authorization. If an action's result is
  uncertain, inspect it before retrying rather than issuing it again blindly.
