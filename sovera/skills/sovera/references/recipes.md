# Sovera recipes

Every call below is a plain HTTPS request. Rules for all of them:

- Pass the key only as `-H "Authorization: Bearer $SOVERA_API_KEY"`. Never print it.
- Build JSON bodies with `jq -n --arg ...` so user text is quoted safely. If `jq` is
  missing, use `python3 -c 'import json,sys; print(json.dumps(...))'` instead.
- Requests accept snake_case field names. Responses use camelCase (`runId`,
  `nodeId`), omit empty and false fields, and encode 64-bit numbers as strings
  (`"durationMs": "3413"`).
- An error is a non-2xx status with a body like
  `{"code":"not_found","message":"run not found"}`. Look the code up in
  `errors.md`. The async start endpoint is the exception: its errors are plain text.

## Reachability check

`GetVersion` needs no key. Run it first to prove `SOVERA_URL` reaches a Sovera
server.

```bash
curl -sS -X POST "$SOVERA_URL/orchestrator.v1.OrchestratorService/GetVersion" \
  -H "Content-Type: application/json" -d '{}'
# {"version":"..."}
```

Any JSON reply means the server is reachable. Do not compare its `version` with
the plugin's: the server does not report the release it runs. If the user asks
whether the plugin matches their instance, tell them to ask their operator for the
release tag (like `master-<sha8>-<epoch>`) and pin it in Claude Code:

```
/plugin marketplace add wearenotch/sovera-claude-plugins#<tag>
```

## Short task

`Complete` runs a prompt and blocks until the answer is ready. Use it for tasks
that finish within about a minute and need no approval: it cannot wait on a gate.

```bash
jq -n --arg p "Summarise the attached notes in three bullets: ..." --arg s "my-session" \
  '{prompt: $p, session_id: $s}' |
curl -sS -X POST "$SOVERA_URL/orchestrator.v1.OrchestratorService/Complete" \
  -H "Authorization: Bearer $SOVERA_API_KEY" -H "Content-Type: application/json" -d @-
```

The answer is in `output`. `status` is `success` on success; `agentRole` names the
agent that produced it. `session_id` is optional; reuse one to keep a conversation
going (see `concepts.md`, Sessions).

To skip planning and run one blueprint directly, prefix the prompt with its ID:
`"[summarizer] Summarise ..."`. An unknown or disallowed ID is not an error: Sovera
silently plans the whole prompt instead. Check `planning.mode` on the run
(`direct` vs `generated`) to see which happened.

## Long task

Use this for anything slow, or any task that may stop at a gate. Start the run,
keep the IDs, then poll.

1. Start it. The endpoint is plain HTTP, not an RPC:

```bash
jq -n --arg p "Research X and draft a report" --arg s "my-session" \
  '{prompt: $p, session_id: $s}' |
curl -sS -X POST "$SOVERA_URL/api/async/workflows" \
  -H "Authorization: Bearer $SOVERA_API_KEY" -H "Content-Type: application/json" -d @-
# 202 {"run_id":"b75e6dc1-...","workflow_id":"req-70e5...","request_id":"req-70e5...",
#      "session_id":"my-session","status":"accepted"}
```

Keep `run_id` (for `GetRun`, gates) and `request_id` (for the final output, step 3).
Optional body field `callback_url`: a public `https` URL Sovera POSTs the result
to when the run ends. Errors here are plain text, e.g. `400 prompt cannot be empty`,
`401 unauthenticated: ...`.

2. Poll `GetRun` every 5 seconds, slowing to every 15 seconds after a minute,
until `run.status` is `completed` or `failed`:

```bash
jq -n --arg r "$RUN_ID" '{run_id: $r}' |
curl -sS -X POST "$SOVERA_URL/run.v1.RunService/GetRun" \
  -H "Authorization: Bearer $SOVERA_API_KEY" -H "Content-Type: application/json" -d @-
```

- `not_found` right after starting is normal: the run is recorded once planning
  finishes, which can take 10 seconds or more. Keep polling. If it is still
  `not_found` after two minutes, check step 3: planning may have failed.
- While `running`, report progress from `nodes[].state` (e.g. "2 of 3 steps done").
  A node that is still working reads `pending`; only finished states are recorded.
- If a node of kind `gate` is `pending` and the steps before it are `done`, the
  run is probably waiting for a signal: go to **Gates**.
- If `failed`, explain `run.failure` and the failed nodes' `failure` (see `errors.md`).
- `completed` can still contain failed nodes or `synthetic` outputs (a fallback
  replaced a failure). Mention them; the answer may be degraded.

3. Read the answer. Either take the `output` of the nodes in the last entry of
`run.layers`, or ask for the bundled result by `request_id`:

```bash
jq -n --arg r "$REQUEST_ID" '{run_id: $r}' |
curl -sS -X POST "$SOVERA_URL/orchestrator.v1.OrchestratorService/GetRunStatus" \
  -H "Authorization: Bearer $SOVERA_API_KEY" -H "Content-Type: application/json" -d @-
# {"runId":"req-70e5...","status":"completed","output":"{\"final_output\":\"...\",
#   \"node_outputs\":{...}, ...}"}
```

`GetRunStatus` takes the `request_id` here, not the `run_id` (the `run_id` reads
`not_found`). Its `status` is `active`, `completed` or `failed`. `output` is
usually a JSON string whose `final_output` is the answer. The operator can set a
different output format for the instance, so if `output` does not parse as JSON
or has no `final_output`, present `output` itself as the answer.

Do not use `SubmitTask`, `ResumeProgress` or `StreamRun`: they are streaming calls
that plain `curl` cannot speak (a JSON request gets `415`). The async start above
is the curl-friendly way to run long tasks, and a run started this way keeps going
when your connection drops.

## Gates

A gate pauses a run until a signal arrives. List what the run is waiting for:

```bash
jq -n --arg r "$RUN_ID" '{run_id: $r}' |
curl -sS -X POST "$SOVERA_URL/orchestrator.v1.OrchestratorService/GetPendingSignals" \
  -H "Authorization: Bearer $SOVERA_API_KEY" -H "Content-Type: application/json" -d @-
# {"signals":[{"signalId":"bdf4...","nodeId":"approval","runId":"3e86...",
#   "signalSchema":"{\"type\":\"object\",\"required\":[\"approved\"],...}",
#   "timeoutAt":"2026-10-07T08:26:24Z","description":"Approve the draft summary?",
#   "context":"<output of earlier steps>"}]}
```

An empty response (`{}`) means nothing is waiting. `not_found` means the run is
unknown or not this tenant's; right after an async start it can also mean the run
is not recorded yet, so retry for a minute before giving up. With durability off,
gates can be read and answered only while the run is executing. For each signal, show the user
the `description`, the `context` and the deadline `timeoutAt`, and ask for their
decision. Build a payload that matches `signalSchema` (a JSON Schema, given as a
string). Then send it:

```bash
PAYLOAD='{"approved": true}'
jq -n --arg r "$RUN_ID" --arg n "$NODE_ID" --arg s "$SIGNAL_ID" --arg p "$PAYLOAD" \
  --arg a "user:$USER" \
  '{run_id: $r, node_id: $n, signal_id: $s, payload: $p, actor: $a}' |
curl -sS -X POST "$SOVERA_URL/orchestrator.v1.OrchestratorService/SendSignal" \
  -H "Authorization: Bearer $SOVERA_API_KEY" -H "Content-Type: application/json" -d @-
```

- `{"accepted":true,"reason":"ok"}` - delivered; keep polling `GetRun`.
- `{"reason":"unknown_signal"}` (no `accepted` means false) - wrong IDs, or the
  signal was already used. Re-read `GetPendingSignals`.
- `{"reason":"signal_expired"}` / `{"reason":"already_resolved"}` - too late, or
  someone else answered first.
- HTTP 400 `invalid_argument` "signal payload does not match gate schema" - fix
  the payload against `signalSchema` and resend.

`payload` is a JSON **string**, not an object. `actor` says who decided
(`user:<name>`), and is recorded with the run.

## Browse runs

List runs, newest first:

```bash
jq -n '{status: "failed", prompt_search: "report", page: 1, page_size: 10}' |
curl -sS -X POST "$SOVERA_URL/run.v1.RunService/ListRuns" \
  -H "Authorization: Bearer $SOVERA_API_KEY" -H "Content-Type: application/json" -d @-
```

All filters are optional: `status` (`running`, `completed`, `failed`),
`prompt_search` (case-insensitive substring of the prompt), `created_from` /
`created_to` (RFC 3339 timestamps), `page` (from 1), `page_size` (default 25, max
100). Each row has `runId`, `requestId`, `status`, `title` (start of the prompt),
`nodeDone` / `nodeTotal` (`nodeTotal` 0 means unknown) and timestamps; `total`
counts all matches. Do not pass `tenant_id`; the key already scopes the list.

Runs started by `Complete` are listed too. Open one with `GetRun` (see Long task).

**Durability off.** If `ListRuns` returns `"durabilityDisabled": true`, this
instance does not record runs: the list is empty and `GetRun` fails with
`failed_precondition`. Use **Short task** instead. A run you already started with
the async endpoint can still be followed with `GetRunStatus` by `request_id`
(step 3 of Long task); its nodes cannot be read, and its gates only while it runs.

## Blueprints

A tenant key cannot list the instance's blueprints; that list is operator-only.
To find what the user can use:

1. Ask their Sovera operator for the blueprint IDs enabled for their tenant.
2. See which blueprints past runs used: `GetRun` and read `nodes[].blueprint.id`
   (or `nodes[].taskType`).
3. Preview which blueprints Sovera would pick for a prompt, without running
   anything:

```bash
jq -n --arg p "Summarise a document and translate it to French" '{prompt: $p}' |
curl -sS -X POST "$SOVERA_URL/orchestrator.v1.OrchestratorService/GenerateWorkflowTemplate" \
  -H "Authorization: Bearer $SOVERA_API_KEY" -H "Content-Type: application/json" -d @-
# {"nodes":[{"id":"step_0_summarization","kind":"task","taskType":"summarizer"},
#   {"id":"step_1_translation","kind":"task","taskType":"translator",...}], ...}
```

Each task node's `taskType` is a blueprint ID the tenant may use. This calls the
planner, so it takes a few seconds; it only drafts a plan.
