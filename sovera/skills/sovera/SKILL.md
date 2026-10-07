---
name: sovera
description: Use when the user wants to work with a Sovera instance - run a task or prompt on Sovera, follow or list Sovera runs, approve or answer a gate (signal) a run is waiting on, explain why a Sovera run or node failed, or find out which Sovera blueprints (agents) they can use. Calls the Sovera HTTP API with curl and a tenant API key.
---

# Sovera

Sovera runs multi-agent workflows. You send a prompt; Sovera plans it as a graph of
steps (nodes), runs each step with an agent built from a **blueprint**, and records
the run so it can be read back. This skill drives Sovera with `curl` only.

## Setup

Two environment variables, set by the user in their shell:

- `SOVERA_URL` - the instance base URL, e.g. `https://sovera.example.com` (no trailing slash).
- `SOVERA_API_KEY` - the tenant API key (starts with `vtg_`).

Check them without revealing the key:

```bash
[ -n "$SOVERA_URL" ] && echo "SOVERA_URL=$SOVERA_URL" || echo "SOVERA_URL is not set"
[ -n "$SOVERA_API_KEY" ] && echo "SOVERA_API_KEY is set" || echo "SOVERA_API_KEY is not set"
case "$SOVERA_URL" in
  https://*) ;;
  http://*) case "${SOVERA_URL#http://}" in
      localhost|localhost[:/]*|127.0.0.1|127.0.0.1[:/]*|'[::1]'|'[::1]'[:/]*) echo "SOVERA_URL is plain http to this machine (loopback)" ;;
      *) echo "WARNING: SOVERA_URL is not https; the API key would be sent in cleartext" ;;
    esac ;;
  *) echo "WARNING: SOVERA_URL does not start with https://" ;;
esac
```

The key travels in a header on every call, so on a WARNING tell the user and make
no authenticated call until they switch `SOVERA_URL` to `https://` or explicitly
accept the risk. Plain `http` is fine only for an instance on their own machine.

**Never print, echo, log or paste the key.** Always pass it as
`-H "Authorization: Bearer $SOVERA_API_KEY"` so the shell expands it; never `set -x`,
never `curl -v`, never write it into a file. If it is missing, ask the user to export
it themselves.

## Mental model

- The key belongs to one **tenant**. `ListRuns` lists only that tenant's runs, and
  `GetRun`, `GetPendingSignals` and `SendSignal` answer another tenant's run ID with
  `not_found`, exactly as for an unknown one.
- A **run** is one workflow execution: `running`, then `completed` or `failed`.
  Its **nodes** are `pending`, `done`, `failed` or `skipped`.
- A **gate** is a node that pauses the run until someone sends a **signal**.
- Every call is `POST $SOVERA_URL/<package>.<Service>/<Method>` with a JSON body.

## What to read

Read only the file the task needs.

| The user wants to... | Read |
|---|---|
| check the setup or that the server is reachable | `references/recipes.md` - Reachability check |
| run a short task and get the answer | `references/recipes.md` - Short task |
| run a long task, or one that may wait for approval | `references/recipes.md` - Long task |
| answer or approve a waiting gate | `references/recipes.md` - Gates |
| list or inspect runs | `references/recipes.md` - Browse runs |
| know which blueprints they can use | `references/recipes.md` - Blueprints |
| understand an error or a failed run | `references/errors.md` |
| understand tenants, modes, statuses, planning, provenance, sessions | `references/concepts.md` |

Start a session with the reachability check. Before sending a gate signal, show the user
what the gate asks and get their decision; never approve on their behalf.
