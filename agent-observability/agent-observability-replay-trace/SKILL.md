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
- **A callable seam.** Not the binary "is the app locally runnable?" — ask **what is the innermost callable
  seam corresponding to the trace's root span, and can it be called directly with a JSON input?** A
  deployed-only HTTP/gRPC service often still exposes a plain callable underneath its handler (the common
  ports-and-adapters / hexagonal case) — full local-setup would be overkill, but "already runnable" is also
  wrong. Only when no seam can be called directly — truly deployed-only, or one needing live infra — does the
  skill offer to **set up a local testing flow** first (see `references/local-setup.md`); it still needs you
  for secrets and the stub-vs-real call.
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

**pup response shape (parse this, not the top level):** pup nests results under **`data.spans[]`**, and when
it detects an AI agent it wraps the whole payload in `{status, data, metadata}` (its own `metadata.note`
suggests `--no-agent` for scripts). Parsing the top level yields **zero hits on a trace that is actually
ingested** — the single most dangerous failure here, because a wrong "0 spans" reads as a normal "not found"
(see step 7). Pass `--no-agent` to stabilize the parse path, and always look under `data.spans[]`.

**pup exact usage** (the flags are non-obvious, but this copy has drifted before — treat `pup <cmd> --help`
and pup's own error text as the source of truth, and keep only the genuinely non-obvious bits below):
- `pup llm-obs spans get-trace --trace-id <id> --from 30d` — `--trace-id` is a flag (not positional); the
  default window is **1h**, so pass `--from` as a **bare duration** (`30d`, `7d`, `1h`) — `now-30d` is
  rejected. Returns `total_duration_ms` and a ready `trace_url`. **Carry the `trace_id` forward — not the
  `apm_trace_id`** that also appears in results; feeding `apm_trace_id` back into get-trace 404s.
- `pup llm-obs spans get-content --trace-id <id> --span-id <root-span-id> --field output --from 30d` —
  span-id comes from get-trace; `--field` is required. **get-content has the same 1h default window**, so
  pass `--from` for older traces — a 404 `"span not found: <id>"` there is usually the **window**, not a bad
  span id.
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
(MCP `get_llmobs_span_content` / pup `spans get-content`). Read its `metadata.replay_input` /
`metadata.replay_entrypoint` if present, note its `total_duration_ms` (drives the step 7 timeout) and its
`trace_url` (both backends return one). (In pup mode the default window is 1h — pass `--from 30d` (bare
duration, **not** `now-30d`) for older traces.)

**Locate the baseline field — don't assume it's the root output.** The value the developer is dissatisfied
with is often **not** the root span's output (which may be a summary/counter carrying nothing about the
change) — it can be a tool-call **input** or an intermediate output several levels deep. Find the span
field(s), at whatever depth, that actually express the complaint, and use those as the diff baseline. Also
check whether the app **post-processes** that value between what the model produced and what the span
records (e.g. appending deterministic boilerplate): if two plausible "before" values exist, pick the one the
code change can actually move, or the delta drowns in noise.

### 2.5. Check for fan-out before replaying the whole root
If the root span **fans out into repeated sibling subtrees** (a batch/map that dispatches N parallel
sub-runs), the change under test is usually visible in a **single** representative branch. Replaying the
whole root then costs ~N× the model spend and wall-clock for no extra signal. Surface the fan-out and offer
to replay one representative branch instead of the entire root; **log what you skipped** (consistent with
the skill's "no silent caps" instinct). Only replay the full root if the change is inherently cross-branch.

### 3. Resolve the entrypoint + input
- **Entrypoint:** if `metadata.replay_entrypoint` is present, use it as the dispatch id. If absent, **infer**
  the entrypoint from the root span (name/kind) + code and **ask the user to confirm** before proceeding.
- **Input:** if `metadata.replay_input` is present, use it. If absent, derive a **suggested** input from the
  trace (best-effort — the rendered prompt is lossy, so prefer the code signature) and have the user
  **confirm or edit** it.

### 3.5. Ensure a local run path (find the innermost callable seam)
Replay re-runs the entrypoint **locally**. Resolve this by asking **"what is the innermost callable seam for
this root span, and can I call it directly with JSON?"** — not the binary "is the app runnable?":
- **Seam already directly callable** (an importable function you can call) → **skip this step**, continue.
- **Seam buried under a handler/service** (deployed-only HTTP/gRPC, no local `__main__`/CLI, deps not
  installed, live-infra coupling) → **read `references/local-setup.md` and follow it**: detect the gap,
  **propose** a local testing flow, get one approval, then build it (pausing only for secrets and the
  stub-vs-real decision).

The middle case — a deployed service whose core logic is *already* a plain callable with local-dev seams —
is the **normal** ports-and-adapters shape: extract/call that seam rather than running full local-setup.

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
- **b) The runner** — a small CLI that satisfies the **language-independent runner contract** in
  `references/details.md` (load env → derive `<ml_app>-local` → dispatch one entrypoint on JSON input →
  **flush on every exit path, including errors** → print the `-local` ml_app). For **Python**, copy
  `scripts/replay_runner_template.py` → `replay_runner.py` and fill its **`ENTRYPOINTS` dispatch table** (one
  entry per type, keyed by `replay_entrypoint` → function + sync/async). For **other languages, write the
  runner to the same contract** — do not assume the Python template's API shape carries over (e.g. Go uses
  `tracer.Start(WithLLMObsEnabled/WithLLMObsMLApp/WithLLMObsAgentlessEnabled)`, not `LLMObs.enable(...)`, and
  needs an explicit flush before exit **on the error path too**, or a failed replay leaves no diffable trace;
  the `DD_TAGS` correlation marker does carry across languages). Extend the table/dispatch when new
  entrypoints appear; keep the file.
  - **Run command:** infer it (venv/interpreter/build) from the project and **confirm it** with the user.
    Follow the host repo's **build-file conventions** for the new source file — repos with generated build
    metadata (Bazel/Gazelle, Pants, Buck, or a lockfile/manifest to regenerate) need that step or the runner
    won't build. On a **transient dependency-fetch failure**, retry the build **once** before reporting a
    break. If the entrypoint needs non-serializable live infra, ask how to build it or skip it.
  - **Export mode:** check how the app ships spans before replaying. The Python template defaults to
    agentless (`agentless_enabled=True`), which is what you want locally, but a service may be wired to a
    local Agent sidecar. Prefer **agentless** for the replay. Pre-empt a benign gotcha: with agentless LLM
    Obs on, the APM tracer may still dial `localhost:8126` and log `ERROR: lost N traces … connection
    refused` — **harmless** (LLM Obs spans ship independently and arrive fine); flag it so it isn't mistaken
    for a failed replay (it compounds the false-negative in step 7).

### 5. (If a change was requested) edit the code, then gate on a selector
Analyze the trace + the request, make the code changes, show the developer the diff of your changes, then
present an `AskUserQuestion` **selector** (not a plain question) — e.g.:
- **Replay now** — proceed to step 6.
- **Adjust the changes first** — the user says what to adjust; edit again and re-present this gate.
- **Cancel** — stop without replaying.
Only replay on the "Replay now" choice.

### 6. Replay
**Before the first replay, run an ambient-environment check and surface it (don't silently correct).** An
ambient var in the shell can quietly reroute the replay so it doesn't match production — a fidelity gap that
is **invisible in the diff**. Check for: a provider API key (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, …) that
could make the SDK **bypass the app's configured model gateway**, and ambient `DD_*` that could send the
trace to a different org/site. The runner's `load_dotenv(override=True)` handles `DD_*`, but provider
credentials change what the diff is *worth*, so **report what you found** and let the user decide.

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
  **Sanity check before you ever report "not found":** re-query with **no tag filter** (just the
  `<ml_app>-local` + time window) to distinguish "nothing ingested" from "my filter/parse is wrong." A
  parse/scope mistake (e.g. reading pup's top level instead of `data.spans[]`, or polling the production
  ml_app) produces a **confident false negative** — telling the user the replay never landed when it did,
  which reads as normal and invites a wasteful re-run. If the unfiltered query returns spans, the problem is
  your filter, not ingestion.
  **Don't hard-fail** on timeout: say it hasn't appeared yet and offer to keep waiting.

### 8. Summarize the diff (with links to both traces)
Fetch the new trace and give a **concise** summary of how the **new output differs from the old** — just the
meaningful output differences, not the full span trees. Note that live-world drift (time, prices, search
results) can differ even with unchanged code. **When the edit targeted model-facing text** (a prompt, system
message, or tool schema) rather than deterministic code, add that **model sampling variance is a separate
confound**: a single replay can't separate "my change worked" from "this sample differed." Say plainly that
n=1 is *suggestive, not conclusive*, and offer to replay 2–3 units (or the same unit twice) to tell them
apart.

**Every diff view must start with clickable links to BOTH traces** so the developer can open either in the
UI. Both backends return a ready `trace_url`, but there's one scoping trap:
- **Old trace:** use its `trace_url` **verbatim** — do NOT hand-construct a `/llm/traces` URL.
- **New (replay) trace:** the `trace_url` carries **no `ml_app`**, so opened as-is it scopes to whatever app
  the user's picker last had (usually production) and renders **empty** — it must carry
  `ml_app=<ml_app>-local`. **But the returned URL is an org-switch wrapper**
  (`…/switch_to_user/<id>?next=<URL-encoded /llm/traces …>&flow=org_switch`) with the real `/llm/traces` path
  encoded inside `next`. So **do NOT append `&ml_app=…` to the outer URL** — that lands after
  `&flow=org_switch` on the `switch_to_user` endpoint, which ignores it and redirects to the decoded `next`
  path (no ml_app). Instead: URL-**decode** the `next` value, add `ml_app=<ml_app>-local` to its
  `/llm/traces` query, then re-encode `next` (keeping `&flow=org_switch`). If a backend ever returns a
  **bare** `/llm/traces?…` URL (no wrapper), just add `&ml_app=<ml_app>-local` directly. *(This scoping can't
  be verified from a coding agent — confirm the replay link opens non-empty in a browser once.)*
```
- [Old trace](<old trace_url — verbatim>)
- [New trace](<new trace_url with ml_app=<ml_app>-local injected into the `next` path>)
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
