---
name: agent-observability-replay-trace
description: >-
  Use when a developer wants to iterate on ONE specific Agent Observability / LLM Obs trace whose output
  they didn't like — re-running that trace against their LOCAL code, seeing a concise diff of the old vs
  new output, and looping (change code → replay → diff) until satisfied. Invoked as
  /agent-observability-replay-trace <trace-id> [changes to test]. Signals: "replay this trace"; "iterate on
  a trace"; "this trace's output is wrong, fix it and re-run"; "re-run trace <id> with <change>"; pasting a
  trace id from the Agent Observability UI with a description of what to fix. It fetches the trace via the
  datadog-llmo MCP (or the pup CLI as a fallback), edits code, re-runs the app to emit a NEW trace, and
  diffs the two — no local server, no browser. For agents traced with ddtrace / LLM Obs (Python first-class), with JSON-serializable entry
  input. Do NOT use for: scored Experiments or the browser "Replay" button (that's
  agent-observability-replay-experiment), building an experiment from a dataset/CSV, writing evaluators,
  root-causing failed traces, or RUM/HTTP session replay.
---

# Replay a trace against local code

A fast **iteration loop** on a single production trace: take a trace whose output a developer didn't like,
optionally change the code to fix it, **re-run that trace against their LOCAL code**, and show a concise
diff of old vs new output — repeating until they're happy. It assumes nothing about the project's layout.

Invoked from the developer's coding agent (they paste a CTA from the Agent Observability UI):
`/agent-observability-replay-trace <trace-id> [<changes to test>]`.

## The loop (what you're building each run)

1. Fetch the trace and read its output (the baseline).
2. If a change was requested, edit the local code to address it — **show the changes and get an OK before replaying**.
3. **Replay**: re-run the entrypoint locally so it emits a **new trace**.
4. Wait for the new trace, then show a **concise diff** of old vs new output.
5. Satisfied → done. Not satisfied → the developer says what's still wrong → back to step 2. Iterate.

With **no** modification (`/agent-observability-replay-trace <trace-id>`): do the replay + diff only (a
reproduce/regression check), then offer to enter the edit loop.

## Interaction model — selector gates, never a hard stop

This is a live loop. At every decision point, present the choices as an **interactive selector** (the
`AskUserQuestion` tool — the same menu style as plan mode), **not** a plain question that ends your turn.
There are two gates: (a) after you **propose code changes**, before replaying; and (b) after **each diff
view**. Keep re-presenting the selector after every replay until the user explicitly chooses to finish —
do not stop mid-loop. Only end when they pick "Looks good — stop here".

The selector always offers a **free-text option**, so when a choice needs detail (what to refine, what to
adjust), the user types it **right in the selector** — you get their description in the same view. Treat
that free-text as the instruction and act on it directly; don't follow up with a separate question.

## Scope — check this first

- **Traced with `ddtrace` / LLM Obs**, with an `ml_app` and a discoverable entrypoint. **Python is
  first-class**; other languages work in principle (the loop is language-neutral) but you must learn that
  language's build/run command and generate the runner in it.
- **Entrypoint input JSON-serializable.** The runner invokes the entrypoint with a JSON input.
- **Locally runnable.** If the app can't be invoked locally with a JSON input — a deployed-only service, an
  HTTP/gRPC handler entrypoint, or one needing live infra — the skill offers to **set up a local testing
  flow** first (see `references/local-setup.md`); it still needs you for secrets and the stub-vs-real call.
- **Needs a trace-access backend** — the `datadog-llmo` MCP (preferred) or the `pup` CLI (step 0).
- **Credentials:** `DD_API_KEY` + `DD_SITE` + the agent's provider key(s). **Not** `DD_APP_KEY` — this
  replays into a plain trace, not an Experiment.
- **Side effects:** replaying re-runs real code (real model spend + any real writes the agent does). See
  step 6 — warn before the first replay.

## Why trace-only (not an Experiment)

This is deliberately **not** the Experiments path (that's `agent-observability-replay-experiment`). Re-running
the app just emits a normal new trace; the comparison is an LLM diff of the two traces' outputs. This keeps
it lightweight, drops the `DD_APP_KEY` requirement, and isn't limited to Python's Experiments SDK. Details in
`references/details.md` — read it before generating the runner.

## Workflow

### 0. Ensure a trace-access backend (MCP preferred, pup fallback)
The skill reads traces through a **trace-access backend** — the `datadog-llmo` MCP (preferred, richest) or
the **pup CLI** as a fallback. Later steps say "fetch the trace / read span content / poll for spans"
without caring which; the two backends map like this:

| operation | MCP backend | pup backend |
|---|---|---|
| fetch a trace | `get_llmobs_trace` | `pup llm-obs spans get-trace --trace-id <id>` |
| read span output | `get_llmobs_span_content` | `pup llm-obs spans get-content` |
| poll for the replay | `search_llmobs_spans` | `pup llm-obs spans search` |

**pup exact usage** (verified against pup 1.8.0 — the flags are non-obvious):
- `pup llm-obs spans get-trace --trace-id <id> --from 30d` — `--trace-id` is a flag (not positional); the
  default window is **1h**, so pass `--from` as a **bare duration** (`30d`, `7d`, `1h`) — `now-30d` is
  rejected. Returns `total_duration_ms` and a ready `trace_url`.
- `pup llm-obs spans get-content --trace-id <id> --span-id <root-span-id> --field output` — span-id comes
  from get-trace.
- `pup llm-obs spans search --ml-app <app> --root-spans-only --from <t0> --query "replay_run_id:<id>"` — the
  tag filter is a plain `key:value` in `--query` (the MCP-style `@replay_run_id:` matches nothing). Full
  results include each span's `output`, `tags`, and `trace_url`.
- Add `--org <name>` (or ensure `pup auth` picked the app's org) — a mismatched org returns a bare
  `404 "no spans found"`, not an auth error.

Pick a backend, in order:
1. If `mcp__datadog-llmo-mcp__*` tools are present → use the **MCP**.
2. Else if `pup` is installed and `pup auth` points at the app's org → use **pup**.
3. Else **guide the user to install the MCP** — it's the one-command option (easier than pup's
   brew-install + `pup auth login`):
   `claude mcp add --scope user --transport http "datadog-llmo-mcp" 'https://mcp.datadoghq.com/api/unstable/mcp-server/mcp?toolsets=llmobs'`
   (pup stays a "use it if already present" fallback — don't send users to install it.) Resume once a
   backend is available; do not proceed without one.

### 1. Parse the command
`<trace-id>` (required) and an optional free-text modification (everything after the id). No modification →
reproduce/diff-only mode. Determine the `ml_app` from the project (`LLMObs.enable(ml_app=…)` /
`DD_LLMOBS_ML_APP`) or the trace; confirm if ambiguous.

### 2. Fetch the trace
Fetch it via the backend (MCP `get_llmobs_trace` / pup `spans get-trace`), reading span content as needed
(MCP `get_llmobs_span_content` / pup `spans get-content`). Read the **root span's output** — this is the
baseline for the diff — and its `metadata.replay_input` / `metadata.replay_entrypoint` if present. Also note
its `total_duration_ms` (drives the step 7 timeout) and its `trace_url` (both backends return one). (In pup
mode the default window is 1h — pass `--from 30d` (bare duration, **not** `now-30d`) for older traces.)

### 3. Resolve the entrypoint + input
- **Entrypoint:** if `metadata.replay_entrypoint` is present, use it as the dispatch id. If absent, **infer**
  the entrypoint from the root span (name/kind) + code and **ask the user to confirm** before proceeding.
- **Input:** if `metadata.replay_input` is present, use it. If absent, derive a **suggested** input from the
  trace (best-effort — the rendered prompt is lossy, so prefer the code signature) and have the user
  **confirm or edit** it.

### 3.5. Ensure a local run path (only if the app isn't locally runnable)
Replay re-runs the entrypoint **locally**. If the app can't be invoked locally with a JSON input — a
deployed-only service, an HTTP/gRPC handler entrypoint, no local `__main__`/CLI, deps not installed, or it
needs live infra — **read `references/local-setup.md` and follow it**: detect the gap, **propose** a local
testing flow, get one approval, then build it (pausing only for secrets and the stub-vs-real decision).
**Skip this step entirely** when a local run path already exists (e.g. the entrypoint is an importable
function you can call). Only then continue.

### 4. Ensure the two persistent artifacts (one-time setup, reused every iteration)
- **a) In-entrypoint annotation** — so future traces self-describe. If the entrypoint doesn't already
  annotate its root span, add it (best-effort, non-destructive):
  ```python
  LLMObs.annotate(span=span, metadata={
      "replay_entrypoint": "<stable id for this type>",
      "replay_input": <input extractor>,   # e.g. {"tickers": tickers}
  })
  ```
  (No `replay_output` — the original trace is the baseline; the diff reads outputs from the traces.)
- **b) The runner** — copy `scripts/replay_runner_template.py` → `replay_runner.py` and fill its
  **`ENTRYPOINTS` dispatch table** (one entry per type, keyed by `replay_entrypoint` → its function +
  sync/async). Extend the table when new entrypoints appear; keep the file. **Infer the run command**
  (venv/interpreter/build) from the project and **confirm it** with the user. If the entrypoint needs
  non-serializable live infra, ask how to build it or skip it.

### 5. (If a change was requested) edit the code, then gate on a selector
Analyze the trace + the request, make the code changes, show the developer the diff of your changes, then
present an `AskUserQuestion` **selector** (not a plain question) — e.g.:
- **Replay now** — proceed to step 6.
- **Adjust the changes first** — the user says what to adjust; edit again and re-present this gate.
- **Cancel** — stop without replaying.
Only replay on the "Replay now" choice.

### 6. Replay
**Before the first replay, warn:** re-running executes the agent for real — model calls cost tokens and any
external writes (DB/email/billing/queues) happen again. On confirmation, record the launch time `t0`, then
invoke the runner with the entrypoint id + input (as a JSON file), passing a unique correlation marker as a
span tag via the environment:
```
DD_TAGS=replay_run_id:<unique-id> <python> replay_runner.py --entrypoint <id> --input-file <path>
```
The runner runs the entrypoint **directly — no wrapper span — so the replay trace looks identical to a
normal run**, and the marker rides along as a tag on the emitted spans.

**Local replays emit under `<ml_app>-local`** (the app's ml_app + `-local`) so they never pollute the
production ml_app — the original trace stays under the real ml_app; the new one lands under `-local`. The
runner does this automatically and prints the `-local` ml_app; poll for the new trace **under that name**.

### 7. Wait for the new trace
Two waits, keyed off the original trace's duration (`total_duration_ms`, read in step 2):
- **Runner run:** give the runner subprocess a timeout of `max(120s, ~3 × total_duration_ms)` — the replay
  runs the same code, so it takes roughly the original duration; 3× catches a hung/stuck run without
  tripping on a normal one.
- **Ingest:** once the runner returns, tell the user **"waiting for the new trace to appear in Datadog…"**
  and poll the backend **every ~5s for up to ~2 min** for the `replay_run_id` tag (from ≈ `t0`), **under the
  `<ml_app>-local` name** the runner printed (not the production ml_app): MCP `search_llmobs_spans`
  (`ml_app: <ml_app>-local`, `tags: {replay_run_id: <id>}`) or pup `spans search --ml-app <ml_app>-local
  --root-spans-only --from <t0> --query "replay_run_id:<id>"` (plain `key:value`, not `@`). If that tag
  isn't queryable, fall back to the **newest root span** under `<ml_app>-local` for this entrypoint created
  after `t0`. Ingest lag is seconds-to-~2 min and does **not** scale with duration.
  **Don't hard-fail** on timeout: say it hasn't appeared yet and offer to keep waiting.

### 8. Summarize the diff (with links to both traces)
Fetch the new trace and give a **concise** summary of how the **new output differs from the old** — just the
meaningful output differences, not the full span trees. Note that live-world drift (time, prices, search
results) can differ even with unchanged code.

**Every diff view must start with clickable links to BOTH traces** so the developer can open either in the
UI. **Both backends return a ready `trace_url`** for each trace (MCP `get_llmobs_trace`; pup
`spans get-trace`/`search`) — use it **verbatim**, do NOT hand-construct a `/llm/traces` URL:
```
- [Old trace](<old trace_url from get_llmobs_trace>)
- [New trace](<new trace_url from get_llmobs_trace>)
```
The link text is just "Old trace" / "New trace". Then the diff summary.

### 9. Iterate — gate on a selector (never a hard stop)
After the diff, present an `AskUserQuestion` **selector** with two options (the tool also offers a free-text
"Other"):
- **Looks good — stop here** — finish; leave the code changes in the working tree for the user to review.
- **Make more changes** — the user describes what to change **inline in the selector** (free-text); use
  that description and go to step 5 (edit → gate → replay → diff). In diff-only mode, this is where the
  first change is made.
Re-present this gate after every replay until the user picks "stop here". Do not end your turn between iterations.

## Reference
- `scripts/replay_runner_template.py` — the runner to copy + fill. Read it first.
- `references/details.md` — the trace-access backend (MCP or pup), the annotation + runner contract,
  correlation-marker/polling, concise-diff guidance, and scope/limitations. Read before generating the runner.
- `references/local-setup.md` — how to set up a local testing flow when the app isn't locally runnable
  (step 3.5). Read only when that gap is detected.
