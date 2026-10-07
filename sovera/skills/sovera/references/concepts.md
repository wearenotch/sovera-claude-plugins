# Sovera concepts, from a tenant key's point of view

## Tenants and API keys

A Sovera instance serves several **tenants** (customers or teams). An API key
(`vtg_...`) belongs to exactly one tenant, and everything it does is scoped to that
tenant:

- It sees only its own tenant's runs and session memory. `ListRuns` lists only
  them; `GetRun`, `GetPendingSignals` and `SendSignal` check the run's owner and
  answer another tenant's run ID with `not_found`, exactly like an unknown one.
  `GetRunStatus` does the same for a run started with the async endpoint.
- It can only use the blueprints the operator enabled for the tenant.
- The operator sets the tenant's limits (requests per minute, agents, memory). Going
  over one returns `resource_exhausted`.
- Keys are created, rotated and revoked by the operator, not through this API. A key
  that stops working (`unauthenticated`) may have been revoked or expired; a
  `permission_denied` on every call usually means the tenant is suspended.

Some fields are filled in only for operator credentials and are always empty for a
tenant key: an error's raw `detail`, the raw text of a tool error, and where a
blueprint was loaded from. Their absence is expected, not a bug.

## Blueprints

A **blueprint** defines one kind of agent: its role, skills, model and the tools it
may call. Each step of a run is executed by an agent built from a blueprint, and the
step's `taskType` names it. Blueprint IDs are snake_case, e.g. `summarizer`.

## Execution modes

How Sovera turns a prompt into a graph of steps, reported as `planning.mode` on the
run:

| Mode | When | What happens |
|---|---|---|
| `generated` | default | A planner reads the prompt and the tenant's blueprints and composes the graph. Planning takes seconds. |
| `direct` | the prompt starts with `[blueprint_id]` | One step, run by that blueprint, no planning. An unknown or disallowed ID silently falls back to `generated`. |
| `template` | the operator pinned a fixed workflow for the tenant | Every prompt runs the same graph; the `[blueprint_id]` prefix is ignored. |

A graph can also change while it runs: a `planner` node may add or remove steps, and
a failed step may trigger a replan. Each change is recorded (see Planning).

## Runs, nodes and statuses

A **run** is one execution of a graph. It is identified by `runId`; `requestId` is
the ID of the call that started it.

- Run status: `running` while it executes (and also when it was interrupted by a
  server restart and can be resumed), then `completed` or `failed`. A run cancelled
  by its caller is `failed` with code `CANCELLED`.
- Node kinds: `task` (one agent), `gate` (waits for a signal), `map` (one task per
  item of a list), `reduce` (combines parallel results), `planner` (changes the graph).
- Node states: `pending`, `done`, `failed`, `skipped`. Only finished states are
  recorded, so a node that is currently running still reads `pending`.
- `run.layers` orders the graph: layer 0 has no dependencies, each later layer
  depends only on earlier ones. The last layer holds the final step(s).
- A node's `input` is what it was given (with references to earlier outputs filled
  in), `output` is the agent's answer, `tokens` its LLM usage, and `toolCalls` the
  tool calls it made (arguments and results are stored as the tool saw them).
- `synthetic: true` on a done node means its output is a fallback produced by error
  recovery, not an agent's answer; its `failure` says what went wrong originally.
  A run can therefore be `completed` with degraded content.

Runs are recorded only when the instance has durability on. With it off, `ListRuns`
reports `durabilityDisabled: true` and runs cannot be read back.

## Signal gates

A **gate** node pauses the run until an external **signal** arrives - typically a
human approval. When a gate is reached it exposes a pending signal with:

- `signalId` - a single-use token; the signal must quote it exactly.
- `description` - what is being asked; `context` - output of earlier steps to decide on.
- `signalSchema` - a JSON Schema the signal's payload must satisfy.
- `timeoutAt` - the deadline. What happens after it is set on the gate: the run may
  fail, continue, skip the steps after the gate, or use a default payload.

`GetPendingSignals` and `SendSignal` accept only the run's own tenant. The server
takes the owner from the run's record; when durability is off there is no record,
so these calls work only while the run is executing on the server that answers,
and read `not_found` otherwise (also on an instance with several servers, when the
call reaches one that is not running it).

After the signal is accepted, later steps can use its payload. The gate node becomes
`done` and its output records the payload, who sent it (`actor`) and when.

## Sessions and memory

`session_id` groups prompts into a conversation. Prompts and answers in the same
session are remembered and given to later runs in that session as context, so a
follow-up like "now shorten it" works. Use a new or omitted `session_id` for an
unrelated task; Sovera then creates one and returns it. Session memory belongs to
the tenant; other tenants cannot read it.

## Planning (why the graph looks like this)

`GetRun` returns `run.planning`:

- `mode` - see Execution modes; `unknown` when only runtime changes were recorded.
- `rationale` - a short plain-language explanation of the graph's shape.
- `durationMs` - how long planning took.
- `history` - every runtime change a planner proposed, oldest first: `trigger`
  (`planner_node` or `replan_on_failure`), `outcome` (`applied` or `rejected`),
  `reason`, and the node IDs it `added`, `removed` or `relinked`. A rejected change
  carries `rejection` (code `PLAN_REJECTED`) and left the graph unchanged.

Each node also has `reason`: one sentence on why it is in the graph. Use these to
answer "why did Sovera do it this way?".

## Blueprint provenance (which agent version ran)

Each task node carries `blueprint`, captured when the step was dispatched:

- `id` - the blueprint that ran.
- `contentHash` - a hash of the blueprint as it ran. Equal hashes mean identical
  blueprints, so comparing it across runs tells whether the agent changed between them.
- `sourceType` - `directory`, `git`, `http`, `generated` (created by Sovera for this
  run) or `unknown`. For `git`, `git.commit` and `git.tags` name the version;
  `generated` may carry `generated.runId`.
- `overridden` - environment-specific settings were applied on top of the file.
- `changedDuringRun` - the step's attempts ran different blueprint versions.

Where exactly the blueprint was loaded from (repository, branch, path, URL) is
operator-only and empty for a tenant key.
