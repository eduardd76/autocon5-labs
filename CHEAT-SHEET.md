# Cheat Sheet — one page

## Access & constants
| | |
|---|---|
| Dashboard (UI) | http://localhost:8501 |
| MCP API key | `dev-key-123` |
| Ports | UI 8501 · Team-Leader 8002 · MCP 8001 · Redis 6379 |
| Neo4j (Lab 4) | http://localhost:7474 · login `neo4j` / `ospf12345` |
| Scenarios | `bgp_session_idle`, `ospf_neighbor_down`, `evpn_route_missing`, `qos_interface_congested` |

## The commands you actually use
```bash
make up                 # docker compose up -d  (start the 6 containers)
docker compose ps       # are all 6 up?
make trigger-bgp        # fire a BGP fault from the terminal
make logs               # watch the agents talk (timestamp-ordered)
make down               # stop everything

# fire any scenario by hand (ONE line):
curl -s -X POST localhost:8001/incident/trigger -H 'X-MCP-API-Key: dev-key-123' \
  -H 'Content-Type: application/json' -d '{"scenario_id":"bgp_session_idle","device_id":"router-1"}'

# read a device tool directly — prove the card grounded (ONE line):
curl -s -X POST localhost:8001/tools/bgp_summary -H 'X-MCP-API-Key: dev-key-123' \
  -H 'Content-Type: application/json' -d '{"device_id":"router-1"}' | python3 -m json.tool

# see pending approval cards:
curl -s localhost:8002/acp/approvals | python3 -m json.tool
```

## Per-lab key command
| Lab | Command |
|---|---|
| 1 · OBSERVE | (browser) click a sidebar fault button → Approve |
| 2 · UNDERSTAND | `make trigger-bgp` then `make logs` |
| 3 · BUILD | `make lab3-test` → `make lab3-solve` → `docker compose up -d --build your-agent` |
| 4 · GRAPH | `cd labs/lab4_build_a_kg && bash start-neo4j.sh && python3 load.py && python3 load_rich.py` |
| 5 · TWIN | `cd labs/lab5_validate_in_twin && python3 twin_validate.py` |
| 6 · CATALOGUE | edit a YAML in `lab_scenarios/mine/` → `make fire-mine SCN=<id>` |

## If it wobbles
| Symptom | Fix |
|---|---|
| `up` hangs / no prompt | you ran `docker compose up` without `-d` → press **`d`** (not Ctrl+C) |
| logs empty | `--since` window aged out → fire fresh, then read |
| logs out of order | add `--timestamps … \| sort -k3` |
| `command not found` after paste | the command got split — paste each as **one line** |
| Lab 4 `localhost:7474` won't load | give Neo4j ~30s after `start-neo4j.sh`; check `docker ps \| grep neo4j` |
| a container is `Restarting`/`Exited` | `docker compose logs <service>` — see `TROUBLESHOOTING.md` |

**The mantra:** Model proposes · evidence grounds · twin validates · human disposes.
