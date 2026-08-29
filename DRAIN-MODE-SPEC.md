# Agent-safe drain mode / Agent 安全排空模式

> Status: proposal
>
> Tracking: [#411](https://github.com/dsh-market/dsh-market/issues/411)
>
> Related: [#162](https://github.com/dsh-market/dsh-market/issues/162),
> [#186](https://github.com/dsh-market/dsh-market/issues/186),
> [#199](https://github.com/dsh-market/dsh-market/pull/199), and
> [#320](https://github.com/dsh-market/dsh-market/issues/320)

## Summary

When an Agent is running, dshmarket currently rejects install, update, and
uninstall requests. That guard prevents a live Agent from mixing already-loaded
code with plugin files replaced underneath it, but it leaves the user with a
manual recovery loop: find every active Agent, wait or cancel, return to the
Market, and retry.

Drain mode keeps the same safety invariant and replaces the dead end with a
deferred operation. Existing Agent turns finish naturally. As each Agent
reaches true idle, dshmarket holds it at DSH's maintenance boundary, where new
inbox work remains queued. Once every live Agent is held, dshmarket performs the
mutation, validates it, releases the holds, and lets queued Agent work continue.

This is cooperative continuation at a defined runtime boundary. It is not
arbitrary suspension of a model stream, tool call, or operating-system process.

## Context and invariant

[#199](https://github.com/dsh-market/dsh-market/pull/199) established the
required invariant:

> dshmarket MUST NOT replace or remove plugin files while an Agent can still be
> executing a turn against those files.

A polling-only retry does not preserve that invariant. A new prompt can start
after the last idle check but before pnpm begins. Cancelling an Agent is also
not a substitute: an interrupted tool may have partial external side effects,
and the interrupted operation cannot generally resume from its exact boundary.

Current DSH Agent handles expose the two primitives needed for a cooperative
solution:

- `whenIdle()` resolves only after whole-Agent activity reaches quiescence;
- `runMaintenance()` synchronously claims a truly idle Agent, keeps public
  status idle, and latches later waking input until the maintenance task
  settles.

Drain mode composes those primitives with dshmarket's existing mutation lock.

## Terminology

- **Turn**: one Agent activity from wake to its next true idle state.
- **Safe boundary**: true Agent idle, with no model request, tool call, or
  between-turn task still active.
- **Maintenance hold**: a pending task started through `runMaintenance()`.
  New inbox work is accepted but cannot wake the Agent until the hold settles.
- **Drain**: wait for every running Agent to reach a safe boundary, then place
  every live Agent under a maintenance hold.
- **Deferred mutation**: an accepted plugin operation waiting for drain or for
  the existing mutation lane.

## Goals

1. Preserve the running-Agent safety invariant without asking users to cancel
   useful work.
2. Make “install/update/uninstall after Agents finish” a Host-owned task that
   survives closing or navigating away from Settings.
3. Accept new Agent messages during drain and continue them automatically after
   the plugin operation settles.
4. Close the check-to-mutation race: no new Agent turn may start between the
   final idle observation and the package-manager mutation.
5. Reuse the existing mutation, validation, rollback, progress, and operation
   reporting paths rather than creating a second installer.

## Non-goals

- Serializing and restoring an in-flight model stream or tool process.
- Cancelling running Agents automatically.
- Retrying a partially completed Agent step.
- Running more than one package mutation concurrently.
- Automatically restarting the DSH Host.
- Persisting a deferred mutation across a Host restart in v1.
- Replacing the lifecycle/gating proposal in #320. A lifecycle gate decides
  whether a mutation may proceed; drain mode decides when it is safe to do so.

## Version 1 scope

Drain mode covers the Market UI's install, update, and uninstall operations and
the public update API's update operation. “Update all” remains a sequence of
ordinary update operations.

Version 1 preserves the current single-mutation rule: one deferred or running
package mutation may exist at a time. A second mutation request receives the
existing `OPERATION_BUSY` outcome. General multi-operation queueing remains the
separate concern tracked by #162.

Profile restore, compatibility rollback, self-uninstall, and direct profile
writes keep their current behavior in v1. They MUST NOT be silently routed
through drain mode without their own recovery and lifecycle acceptance tests.

## User experience

### Request while no Agent is running

The normal confirmation and progress flow remains unchanged. Internally,
dshmarket still claims maintenance holds before entering the mutation lane so
an Agent created in the same window cannot race the operation.

### Request while Agents are running

After the existing source confirmation, the operation appears in Tasks as:

```text
等待 4 个 Agent 到达安全点…
Waiting for 4 Agents to reach a safe point…
```

The row MUST show:

- operation kind and plugin name;
- elapsed wait time;
- current count of Agents not yet safe;
- a disclosure containing the same local Agent ids the current 409 response
  reports;
- a Cancel action.

The plugin card MUST show “等待 Agent / Waiting for Agents”, not “安装失败”.
Closing Settings does not cancel the operation. Reopening Settings reconstructs
the row from the Host operation record.

### Input submitted during drain

New user prompts, scheduled wakes, and Agent follow-ups remain accepted by DSH.
They wait in their normal inboxes behind the maintenance hold. The Market MUST
NOT copy, persist, inspect, or replay those messages itself.

When the mutation and validation settle, releasing the hold lets DSH's existing
wake-latch behavior continue the queued work. The Market does not synthesize a
new prompt and does not rerun the completed turn.

### Operation needs refresh or restart

Drain mode does not change activation semantics. The task reports the existing
refresh/restart outcome after validation. The Market does not restart the Host
automatically. Releasing Agent holds is independent of a browser-only refresh.

## Operation lifecycle

```text
accepted
   |
   v
draining ---- cancel ----> cancelled
   |
   v
queued-for-mutation
   |
   v
running ---- cancel -----> existing cancel/rollback settlement
   |
   v
verifying
   |
   +---------------------> succeeded
   +---------------------> failed / rolled back

Every terminal path releases all maintenance holds.
```

For the public update API v1, the externally compatible projection remains
`queued | running | succeeded | failed | cancelled | rolled-back`:

- internal `draining` and `queued-for-mutation` project as `queued`;
- an additive `wait` object explains why the operation is queued;
- internal `verifying` projects as `running` with `progress.phase =
  "verifying"`.

Existing v1 clients therefore continue to work without learning new enum
values.

## Functional requirements

### FR-1: explicit busy policy

Private install/update/uninstall requests and public update API requests accept
an optional busy policy:

```json
{ "busyPolicy": "reject" }
```

or:

```json
{ "busyPolicy": "drain" }
```

`reject` preserves the current 409 behavior and is the default for callers
that omit the field. The first-party Market UI uses `drain` after source and
operation confirmation.

### FR-2: immediate acceptance

A deferred request returns HTTP 202 with a Host-generated operation id. The
HTTP request MUST NOT remain open while Agents drain.

The operation record is process-local, scoped by the current boot id, and
queryable after the initiating page unmounts. This matches the retention model
of the current public update API.

### FR-3: immutable intent

Before an operation enters `draining`, dshmarket MUST complete ordinary request
authorization and source validation, then capture immutable intent:

- install: catalog identity plus an exact npm version, tarball digest/identity,
  or Git commit;
- update: package name, installed-version baseline, and exact resolved target;
- uninstall: package name plus installed spec/version baseline.

If the installed baseline or confirmed target changes before execution, the
operation fails with `TARGET_CHANGED` and performs no mutation. Waiting MUST
NOT silently turn one approved release into another.

### FR-4: Host-owned drain coordinator

One process-local coordinator owns the deferred operation and all maintenance
holds. Browser state is only a projection.

The coordinator MUST:

1. subscribe to Agent creation before taking its first inventory snapshot;
2. enumerate every current Agent;
3. synchronously claim `runMaintenance()` on each idle Agent;
4. await `whenIdle()` for each running Agent, then retry the synchronous claim;
5. synchronously claim every Agent created while drain or mutation is active;
6. tolerate an Agent being disposed while it is being drained;
7. proceed only after every live Agent is maintenance-held;
8. retain those holds until mutation and validation settle.

If `runMaintenance()` loses a race because work began first, the coordinator
returns that Agent to the drain set and waits again. It MUST NOT cancel it.

### FR-5: atomic safety boundary

The Agent-creation subscription and maintenance claims form the admission
barrier. The existing mutation lock remains the package/profile serialization
boundary.

The coordinator MUST acquire the mutation lock only after all Agents are held,
then re-enumerate Agents before invoking the existing mutation executor. Any
new Agent is synchronously maintenance-held by the creation subscription, so it
cannot begin its first turn during the final check or mutation.

An implementation that only polls `status === "running"` does not satisfy this
requirement.

### FR-6: one mutation implementation

Drain mode MUST call the same install/update/uninstall executor used by the
immediate path. All existing source restrictions, allow-builds decisions,
trial validation, compatibility checks, rollback behavior, progress parsing,
and activation reporting remain authoritative.

### FR-7: release and continuation

All maintenance holds MUST be released in one `finally` boundary after the
operation reaches a terminal state. Release resolves the maintenance tasks;
DSH, not dshmarket, decides which latched inboxes wake.

The following paths MUST all release holds:

- success;
- validation failure;
- package-manager failure;
- rollback success or failure;
- user cancellation;
- coordinator or Market plugin disposal.

No terminal path may leave an Agent parked.

### FR-8: cancellation

Cancelling during `draining` performs no package/profile mutation and releases
all holds. Cancelling after the executor starts delegates to the existing
package-command cancellation and recovery path, then releases holds only after
that path settles.

Cancel never calls `Agent.cancel()`.

### FR-9: compatibility fallback

Drain mode is available only when the Host's Agent-service contract guarantees
`whenIdle()` and `runMaintenance()` handles and the Host can notify the Market
of Agents created during the drain window. The coordinator also verifies every
enumerated handle structurally before claiming the barrier; one incompatible
handle makes that request unsupported rather than partially drained.

If those capabilities are unavailable:

- `/status` and public capability discovery report `agentDrain: false`;
- `busyPolicy: "drain"` returns a structured unsupported response;
- the first-party UI falls back to the current wait/cancel guidance;
- the existing fail-open behavior for Hosts with no `agents` service is not
  silently reclassified as a safe drain.

### FR-10: process restart

Deferred operations are not durable across Host restart in v1. Operation ids
contain the boot id. After a boot-id change, clients mark an old deferred task
as cancelled by restart and invite an explicit retry.

The coordinator disposer releases holds before normal plugin unload. Abrupt
process death needs no release because the Agent process dies with it.

### FR-11: bounded observability

The operation record exposes only:

```json
{
  "wait": {
    "reason": "agents",
    "activeCount": 4,
    "activeAgentIds": ["session-..."],
    "since": 1787961600000
  }
}
```

It MUST NOT contain prompts, tool arguments/results, transcript content,
credentials, environment values, or session file paths. Exported logs record
state transitions and counts; they do not copy Agent content.

### FR-12: public update API compatibility

Capability discovery adds:

```json
{
  "features": { "agentDrain": true },
  "busyPolicies": ["reject", "drain"]
}
```

Public update requests may opt into `busyPolicy: "drain"`. Requests that omit
it preserve current behavior. The operation response gains only additive
fields; existing state meanings are not repurposed.

## Failure behavior

| Condition | Required result |
|---|---|
| Agent turn runs for a long time | Remain `draining`; show elapsed time; do not cancel it |
| New prompt arrives during drain | Accept into the Agent inbox; keep it behind maintenance |
| New Agent is created | Claim maintenance before its first turn starts |
| Maintenance claim loses a race | Wait for the new activity to become idle and retry |
| Target or installed baseline changes | Fail `TARGET_CHANGED`; mutate nothing |
| Another plugin mutation exists | Return `OPERATION_BUSY`; do not create a second deferred operation |
| User cancels before mutation | Release holds; mutate nothing |
| pnpm/validation/rollback fails | Report the existing failure; release holds after recovery settles |
| Market unloads | Release holds from the owning Cordis disposer |
| Drain capability is absent | Keep the current 409 safety UX; do not emulate drain with polling |

Drain has no automatic timeout. A long-running Agent is legitimate work. The UI
may add a non-terminal warning after a product-defined duration, but waiting,
cancellation, and failure semantics do not change.

## Acceptance criteria

### Coordinator tests

1. Given one running Agent, when a drain operation is accepted, the package
   executor is not called until that Agent reaches idle and is maintenance-held.
2. Given an idle Agent, when drain starts, `runMaintenance()` is claimed before
   a later wake can begin a turn.
3. Given an Agent created during drain, it is maintenance-held before its first
   model request.
4. Given a maintenance claim that loses a race, the coordinator retries after
   the replacement activity reaches idle and never calls `Agent.cancel()`.
5. Given a prompt submitted while maintenance is held, the prompt stays in the
   normal inbox and runs once the hold is released.
6. Given cancellation, executor failure, validation failure, rollback failure,
   or coordinator disposal, every hold settles and no Agent remains parked.
7. Given a changed target/baseline, the executor is never called and the task
   ends with `TARGET_CHANGED`.
8. Given an older Host without drain primitives, the current running-Agent 409
   behavior remains intact.

### Route and API tests

1. Omitted or `reject` busy policy retains the existing status and response.
2. `drain` returns 202 promptly with a boot-scoped operation id.
3. A deferred operation can be queried after the initiating client unmounts.
4. Public API v1 projects drain as `queued` plus `wait.reason = "agents"`.
5. A second mutation remains `OPERATION_BUSY` while one is draining or running.
6. Cancelling a draining operation mutates neither manifest nor `node_modules`.

### Client tests

1. Agent activity produces a waiting task, not a failed task.
2. The task shows count, elapsed time, disclosure, and Cancel.
3. Cards and the Tasks panel converge after navigation away and back.
4. Unsupported Hosts show the current wait/cancel guidance.
5. Boot-id replacement marks the old process-local task cancelled and never
   reports it as completed by the new Host.

### Real runtime acceptance

Run against an isolated real DSH profile, not the user's live profile:

1. Start an Agent with a delayed tool call.
2. Request an install with `busyPolicy: "drain"`.
3. Submit a second prompt while the first turn is still running.
4. Prove no plugin file changes before the first turn becomes idle.
5. Prove the second prompt is admitted but no model request starts during the
   maintenance window.
6. Complete and validate the install.
7. Prove the second prompt wakes exactly once after release.
8. Repeat for update, uninstall, cancellation, and one failed validation with
   rollback.

## Rollout

1. Ship the coordinator and capability flag with `busyPolicy` defaulting to
   `reject` for compatibility.
2. Enable `drain` in the first-party Market UI only on Hosts advertising the
   capability.
3. Verify install, update, and uninstall on isolated Web and Desktop profiles,
   including page close/reopen and Market unload.
4. Keep the old 409 path as the fallback for unsupported Hosts.
5. Consider broader mutation coverage and multi-operation batching only after
   the v1 lifecycle and recovery evidence is stable.

## Rejected alternatives

### Poll until every Agent reports idle

Rejected because a new Agent turn can start between the last poll and pnpm.
The maintenance claim is the admission boundary that polling lacks.

### Cancel Agents, install, then “resume” them

Rejected because cancellation is not suspension. It can interrupt a model
stream or tool after partial side effects, and no general continuation token
exists for the exact execution boundary.

### Hold an `agent/pre-step` listener open

Rejected for v1 because the Agent is already publicly running during a step.
It complicates drain accounting and can park a partially completed turn. The
public idle maintenance boundary is narrower and already defines wake replay.

### Run pnpm concurrently with Agent work and rely on hot reload

Rejected by the mixed-version and missing-file failures reproduced in #186.
Hot activation after mutation does not make the file replacement itself safe.
