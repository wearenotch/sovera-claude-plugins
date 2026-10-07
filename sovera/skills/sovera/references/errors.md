# Sovera errors: what they mean and what to do

There are two layers:

1. **Call errors** - the HTTP call itself failed. Non-2xx status, body
   `{"code":"<code>","message":"..."}`.
2. **Run failures** - the call worked but the run, or one of its steps, failed.
   Recorded on the run as structured `failure` objects.

## Call errors (ConnectRPC codes)

| HTTP | `code` | Meaning for you | What to do |
|---|---|---|---|
| 400 | `invalid_argument` | A request field is missing or malformed (empty prompt, missing `run_id`, gate payload not matching its schema). From `Complete`: the run's input was rejected. | Fix the request; do not retry unchanged. |
| 401 | `unauthenticated` | Key missing, unknown, revoked or expired. | Ask the user to check `SOVERA_API_KEY`; a new key comes from their operator. |
| 403 | `permission_denied` | The tenant is suspended or disabled, or the call is not allowed for a tenant key (e.g. filtering another tenant's runs). | Do not retry. Ask the operator. |
| 404 | `not_found` | Unknown ID, or another tenant's. Right after an async start: the run is still being planned. From the gate calls with durability off: the run is not executing on the server that answered. From `Complete`: a blueprint the plan named does not exist. | Check the ID; for a just-started run, keep polling. |
| 400 | `failed_precondition` | From `GetRun`: this instance does not record runs (durability off). From `Complete`: the LLM provider rejected its key, or an agent asked for a tool it may not use. | Durability: use `Complete`. Otherwise: tell the operator. |
| 429 | `resource_exhausted` | Rate limit or tenant quota reached, or the LLM provider throttled. | Wait and retry with backoff (e.g. 5 s, 15 s, 45 s). |
| 499 | `canceled` | The run was cancelled, e.g. the connection closed. | Start it again; prefer the async start for long work. |
| 504 | `deadline_exceeded` | A step, the run, or a gate ran out of time. | Retry once; if it repeats, split the task. |
| 503 | `unavailable` | A dependency was unreachable: LLM provider, tool, agent, or the server restarted. | Retry with backoff. |
| 409 | `aborted` | The run failed for a reason without a closer code. | Read the message; retry once. |
| 500 | `internal` | Unexpected server error. | Retry once with backoff; if it repeats, report the time and run ID to the operator. |

The async start endpoint (`/api/async/workflows`) answers with plain text, not JSON:
`400` bad body or `callback_url`, `401` bad key, `409` the `X-Request-Id` you sent
is already used, `500` the tenant's callback signing is misconfigured (operator).

When `Complete` fails, `message` is a plain-language sentence. For the structured
cause, find the run with `ListRuns` (newest first) and read it with `GetRun`.

## Run failures (`failure` objects)

`GetRun` reports why things failed in `run.failure` (the primary cause, set only
when the run `failed`) and in each node's `failure`. A failure has:

- `code` - one of the codes below; `category` - its broad class.
- `message` - a plain-language sentence naming the step, blueprint, tool or limit.
  Quote it to the user. `hint` - what can be changed. Pass it on.
- `source` - `llm`, `tool`, `agent`, `planner`, `gate` or `orchestrator`.
- `retryable` - Sovera already retries these while attempts remain; `attempts` is
  how many it made.
- `nodeId` - the failing step; `tool` - the failing tool, if any.

There is no `detail` field for a tenant key: the raw technical error is kept for the
operator. If the user needs it, give the operator the `runId` and `nodeId`.
`nodes[].error` is an older copy of `failure.message`; prefer `failure`.

A node can carry a `failure` without failing the run: on a `done` node with
`synthetic: true`, error recovery replaced the failure with a fallback output. Say
so when presenting the result.

| `code` | Meaning | What the user can do |
|---|---|---|
| `LLM_AUTH` | The LLM provider rejected the API key. | Operator issue: the model's credentials. |
| `LLM_RATE_LIMITED` | The LLM provider throttled requests. | Run it again later. |
| `LLM_OUTPUT_TRUNCATED` | The model hit its token limit before answering. | Ask for a shorter answer or split the task; the operator can raise the blueprint's limit. |
| `LLM_UNAVAILABLE` | The LLM provider failed or gave no usable answer. | Run it again later. |
| `LLM_REQUEST_REJECTED` | The provider refused the request (input too large, unknown model). | Shorten the input; otherwise operator issue. |
| `TOOL_FAILED` | A tool call returned an error. | Check the inputs the tool got (`toolCalls`); tool credentials are configured per tenant by the operator. |
| `TOOL_UNAVAILABLE` | A tool or its gateway could not be reached. | Run it again later; operator issue if it persists. |
| `TOOL_NOT_ALLOWED` | The agent asked for a tool its blueprint does not allow. | Rephrase the task, or ask the operator. |
| `TOOL_LOOP_LIMIT` | The agent kept calling tools without answering. | Make the task more specific. |
| `TIMEOUT` | A step, or the run, exceeded its time limit. | Split the task, or run it again. |
| `CANCELLED` | The caller cancelled the run (request cancelled, connection closed). | Start it again with the async start. |
| `INTERRUPTED` | The server shut down while the step ran. | The run may be resumed by the operator; otherwise start it again. |
| `INVALID_INPUT` | A step's input could not be built (an unresolved reference to an earlier step). | Usually a planning or template problem; run it again, or tell the operator if it is a fixed template. |
| `INVALID_OUTPUT` | An agent's answer was not in the expected format. | Run it again; make the expected format explicit in the prompt. |
| `BLUEPRINT_NOT_FOUND` | The plan named a blueprint that does not exist. | Use a blueprint ID the tenant has (see recipes, Blueprints). |
| `PLAN_REJECTED` | A runtime change to the graph was rejected; the graph stayed as it was. | Usually harmless; see `planning.history`. |
| `QUOTA_EXCEEDED` | A tenant or budget limit was reached. | Wait, or ask the operator for a higher limit. |
| `GATE_TIMEOUT` | A gate's deadline passed without a signal. | Answer gates before `timeoutAt`; start the run again if needed. |
| `GATE_NOTIFY_FAILED` | A gate's notification could not be delivered. | Answer the gate directly with `SendSignal`. |
| `AGENT_CRASHED` | The agent crashed or disconnected mid-step. | Run it again. |
| `AGENT_UNAVAILABLE` | No agent could be started (often the tenant's agent limit). | Wait and run it again; ask the operator if it persists. |
| `INTERNAL` | Anything else, and failures recorded before structured errors existed. | Run it again; report the `runId` to the operator if it repeats. |

## Explaining a failed run

1. `GetRun` the run.
2. Lead with `run.failure.message` (or, for a `completed` run, the failed and
   `synthetic` nodes).
3. For each failed node, give its `reason` (why the step existed), `failure.message`
   and `failure.hint`, and the `tool` if one failed.
4. Say what the user can change, using the table above. Say plainly when only the
   operator can fix it.
