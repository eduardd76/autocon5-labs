# Lab 2 — UNDERSTAND: look under the hood

## The idea — read this first (1 min)

Lab 1 was the dashboard — the front of the restaurant. This lab is the **kitchen.** You'll prove that
the "AI magic" is really just **agents passing tracked messages on a bus**, each doing one small job —
and you'll **verify the evidence by hand** so you trust the card.

Watch for:
- **Every incident carries a tracking number** (`task-id`) that threads across all agents — that's how
  you audit any single incident.
- **The agent reads the device BEFORE it forms an opinion** — step 1 (evidence) is structurally before
  step 2 (diagnosis). Grounding, enforced.
- **It ends `queued`, not `applied`** — the AI is fenced; a human approves.

**NOC analogy:** you stop reading the polished ticket and go watch the on-call engineer actually work —
pull the device stats, reason, write the fix — and you re-run one command yourself to confirm they
didn't make anything up.

---

**Time:** ~15 minutes · **Goal:** see the agents talk *in order*, then verify the evidence yourself.

---

## Step 1 — Confirm the stack is up

From the repo root:
```bash
docker compose ps
```
All 6 containers should be `Up`. If not: `docker compose up -d` (note the **`-d`** — it runs them in
the background instead of attaching to a log firehose that never returns your prompt).

> ⚠️ If you ever run `docker compose up` *without* `-d` and the terminal "hangs", press **`d`** to
> detach. Do **not** press Ctrl+C — that stops the agents.

## Step 2 — Fire a fault from the terminal

In Lab 1 you clicked a button. Now fire the same alarm by hand. Easiest (paste-safe):
```bash
make trigger-bgp
```
…or the raw call it wraps (all one line):
```bash
curl -s -X POST localhost:8001/incident/trigger -H 'X-MCP-API-Key: dev-key-123' -H 'Content-Type: application/json' -d '{"scenario_id":"bgp_session_idle","device_id":"router-1"}'
```
It returns `{"status":"queued", ...}` — **`queued` = your alarm was accepted onto the work queue**,
identical to clicking the button.

## Step 3 — Watch the agents talk (in order)

The reliable habit: **fire, then immediately read the conversation.**
```bash
make logs
```
…or directly:
```bash
docker compose logs --since=1m --timestamps team-leader stability-agent \
  | grep -Ei 'incident|delegat|a2a|step|confidence' | sort -k3
```
You'll see one incident, start to finish, in milliseconds:
```
team-leader   received incident — recognised the type
team-leader   delegating to stability_specialist  (stamps task-xxxx)
[A2A]         team_leader → stability_specialist: task_delegation
stability     step 1/3: read the device (MCP) FIRST
stability     step 2/3: diagnosis · step 3/3: confidence 0.87
[A2A]         stability_specialist → team_leader: task_response
team-leader   ✓ approval queued → lands on YOUR desk
```

**Point at three things:** the `task-id` threads every line · step 1 (read) is before step 2
(diagnose) · it ends **`queued`, not `applied`**.

> Two gotchas everyone hits: logs print **grouped per-container** (add `--timestamps … | sort -k3` to
> interleave by time), and `--since` is a **sliding window** — if the output is empty, just fire fresh
> and read again.

## Step 4 — Prove it grounded (see the raw evidence)

Call the device "hands" (MCP) by hand — exactly what the specialist did in step 1/3 (one line):
```bash
curl -s -X POST localhost:8001/tools/bgp_summary -H 'X-MCP-API-Key: dev-key-123' -H 'Content-Type: application/json' -d '{"device_id":"router-1"}' | python3 -m json.tool
```
The raw read (`neighbor 10.0.0.2`, `state: "Idle"`, `msg_rcvd: 0`) **matches the card's evidence line
exactly.** The LLM does the *reasoning* (Idle + 0 msgs → "TCP/179 blocked"); the **facts come from the
device, not the model.**

---

## The role of the LLM — one gated step

The specialist has **three** steps, and the LLM is **exactly one** of them:

| step 1/3 | step 2/3 | step 3/3 |
|---|---|---|
| gather MCP evidence | **LLM diagnosis** | send response |
| MCP, *not* the LLM | the only model call | packaging, *not* the LLM |

Everything else — detect (triage rules), route (Team Leader), read devices (MCP), validate, approve —
is deterministic or human. **Swap the model and the system still works. The intelligence is the
architecture, not the model.**

*(On this laptop the LLM is **stubbed** — the catalogue supplies the diagnosis. In production it's a
fine-tuned 7B per specialist. Still just this one gated step.)*

---

## What you proved
1. Every incident carries a `task-id` threaded across all agents → **auditability**.
2. The agent **reads the device before it diagnoses** → grounding, enforced.
3. You **verified the evidence by hand** → the card's claims trace to a tool call you can re-run.
4. It ends at a human → **`queued`, not `applied`.** The AI is fenced.

Move to **[Lab 3 — BUILD](../lab3_wire_your_own_specialist/)** when ready.
