# AutoCon5 · WS:C2 — From Alert to Action
### Hands-on labs: build an agentic AI NOC, one lab at a time

> **The one-sentence story:** you fire a real network fault, watch 5 agents solve it, approve the
> fix as the human gate — then *teach the system a brand-new fault of your own.*
>
> Everything ladders up to one idea: **Institutional Memory as Code (IMaC)** — your 2 a.m. expertise,
> written down once as reviewable code and run 24/7.

**The mantra you'll hear at every step:**
> **Model proposes · evidence grounds · twin validates · human disposes.**

This repo is the **laptop-local lab** — a strip-down of the production "Dream Team" stack that runs on
your machine with `docker compose up`. No cloud, no GPU, no API keys, no real LLM. **Mock devices +
a stub LLM** so the whole agent reasoning chain is reproducible offline. Everything *else* — the
agent-to-agent protocol, the device-evidence step, the routing, the human-approval gate — is the
**real architecture**.

---

## The 6-lab journey

| Lab | Movement | You will… | Where |
|---|---|---|---|
| **[1 · OBSERVE](labs/lab1_trigger_an_incident/)** | Watch | Fire a fault and watch 5 agents solve it; approve as the gate | Browser |
| **[2 · UNDERSTAND](labs/lab2_understand/)** | Watch | Go under the hood — see the agents talk on the bus, verify the evidence by hand | Terminal |
| **[3 · BUILD](labs/lab3_wire_your_own_specialist/)** | Build | Write your own specialist agent (2 small functions), make 4 tests pass, fire YOUR card | Terminal |
| **[4 · GRAPH](labs/lab4_build_a_kg/)** | Build | Turn a 3-router network into a live Neo4j knowledge graph the agents read | Terminal + browser |
| **[5 · TWIN](labs/lab5_validate_in_twin/)** | Build | Dry-run a fix on the graph — it catches the dangerous one *before* a human sees it | Terminal |
| **[6 · CATALOGUE](labs/lab6_extend_the_catalogue/)** | Build | Add a brand-new fault by writing ONE YAML file — no code, no restart | Terminal |

Each lab folder has its own step-by-step `README.md` with a plain-English "the idea" intro, a NOC
analogy, and a "what you learn" payoff. Start with Lab 1.

---

## Quick start (5 minutes, laptop-local)

```bash
# 1) clone, then from the repo root:
docker compose up -d          # starts 6 small containers on a private network
docker compose ps             # all up? (mcp, redis, team-leader, stability, ui, your-agent)

# 2) open the dashboard
open http://localhost:8501    # mac   (linux: xdg-open / just paste in a browser)
```

You should see a calm, healthy board. Now do **[Lab 1](labs/lab1_trigger_an_incident/)**.

Prefer paste-safe shortcuts? The `Makefile` wraps the common commands:
```bash
make up            # docker compose up -d
make trigger-bgp   # fire a BGP fault from the terminal
make logs          # watch the agents talk
make down          # stop everything
```

---

## What's behind the dashboard — the 5 layers

When you fire a fault, this is the journey it takes (the same *shape* as the production system):

```
        ┌─────────────────────────── you click "fire a fault" ───────────────────────────┐
        ▼                                                                                   │
  UI (ws-c2-ui :8501)                                                          approval desk (UI)
        │  the dashboard + the approval desk                                         ▲
        ▼                                                                            │
  Team Leader (ws-c2-team-leader)  ── deterministic triage: WHICH specialist? ──►  Specialist
        │                                                                            │ (stability / your-agent)
        │  A2A delegation over the Redis bus (ws-c2-redis = the "post office")       │
        ▼                                                                            ▼
  Specialist  ── step 1/3: read the device (MCP) FIRST ──►  MCP (ws-c2-mcp :8001) ──► mock devices
              ── step 2/3: LLM diagnosis (the ONLY model call) ──►  (stub LLM here; fine-tuned 7B in prod)
              ── step 3/3: reply ──►  Team Leader ──►  approval_queue ──►  UI ──►  YOU approve
```

| Container | Plain-English role | Port |
|---|---|---|
| `ws-c2-ui` | the dashboard you see + the approval desk | 8501 |
| `ws-c2-team-leader` | the **dispatcher** — routes each alarm to the right specialist | 8002 |
| `ws-c2-stability` | a specialist (OSPF / BGP / EVPN) | — |
| `ws-c2-your-agent` | a specialist (QoS) — **you build this in Lab 3** | — |
| `ws-c2-mcp` | the **"hands"** — reads device facts (read-only) | 8001 |
| `ws-c2-redis` | the **"post office"** — the agent-to-agent message bus | — |

**Where the LLM actually fits:** the specialist has 3 steps; the LLM is *exactly one* of them (step 2,
"diagnosis"). Detect, route, read devices, validate, approve — all deterministic or human. *The
intelligence is the architecture, not the model* — swap the model and the system still works.

---

## Two honest simplifications (and why they're the right call)

| | This lab | Production |
|---|---|---|
| **Devices** | mock — canned, realistic facts | real Cisco / Arista over the network |
| **The LLM** | a stub — small canned brain per scenario | fine-tuned 7B per specialist (GPU) |

Why fake exactly these two? Mocks make faults **instant, repeatable, and collision-free** (a whole room
can fire at once, offline, on conference wifi, touching no real router). The stub makes the reasoning
**deterministic** so the teaching is reproducible. **Everything else is the real shape** — a real router
just feeds live numbers into the exact same pipeline.

> This is **not** the production system. The fine-tuned models, the 72-scenario catalogue, the
> multi-vendor adapters, the full validation pipeline and compliance mapping are proprietary and not in
> this repo. See [`NOTICE`](./NOTICE) for the pilot path.

---

## Repo layout

```
.
├── docker-compose.yml        # the 6-container stack
├── Makefile                  # paste-safe shortcuts (make up / trigger-bgp / logs / down)
├── services/                 # the 5 agent services (mcp, team-leader, stability, your-agent, ui) + shared
├── lab_scenarios/            # the "brain" — YAML fault catalogue (Lab 6 extends this)
├── labs/                     # the 6 hands-on labs (start at lab1)
├── scripts/                  # preflight + helpers
├── CHEAT-SHEET.md            # one-page command + troubleshooting reference
└── TROUBLESHOOTING.md
```

---

## If it wobbles
- `make up` then `docker compose ps` — are 6 containers up?
- A command "hangs" after `docker compose up` (no `-d`) → press **`d`** to detach (never Ctrl+C).
- Logs empty? The `--since` window aged out — fire fresh, then read.
- More fixes in [`CHEAT-SHEET.md`](./CHEAT-SHEET.md) and [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md).

---

*Built for AutoCon5 Munich · WS:C2 · vExpertAI. The human is never removed from the loop.*
