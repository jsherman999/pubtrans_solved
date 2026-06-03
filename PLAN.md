# Plan: Public Transportation Simulation

A 2D simulation of a small metro area served by an LLM/agent-controlled fleet of autonomous vehicles.

## Deliverable

A single `index.html` file (HTML + Canvas + vanilla JS, no build step). Double-click to run in any modern browser.

## Tech Decisions

| Question | Choice |
|----------|--------|
| Stack | Single-file HTML + Canvas + JS |
| Dispatcher | Deterministic k-shortest-paths + congestion scoring, labeled "the agent" |
| LLM use | Claude Sonnet 4.6, narrates *interactive mode* decisions only (API key pasted in UI, in-memory) |
| Visual style | Clean geometric — labeled colored blocks, smooth car motion, HUD |
| Population | 100 simulated humans |
| Scope | Larger build (~1000–1400 lines) — schedules, pooling, obstacles, gridlock metric |

## City Model

- 20×20 block grid; streets are the lines *between* blocks
- Building types placed across the grid:
  - 4 employers
  - 2 grocery stores
  - 2 schools
  - 1 train station (drop-off for transfer out of the metro area)
  - ~8 residential clusters
  - remainder empty / streets
- Street graph: nodes = intersections, edges = segments with `length` + live `congestion` weight

## Population (100 humans)

Each human has:
- a home block
- a profile: worker (→ employer), student (→ school), commuter (→ train station), errand-runner (→ grocery)
- a daily schedule that triggers ride requests at the appropriate simulated time-of-day

## Dispatcher ("S")

Deterministic, called "the agent" in narration.

On each ride request:
1. Compute *k* shortest paths from origin to destination on the street graph
2. Score each candidate: `base_length + congestion_penalty(edges committed by other cars in overlapping time windows)`
3. Pick lowest-cost route
4. Assign vehicle:
   - **Shuttle (cap 8)** if 2+ riders share destination in same time window — pool them
   - **Sedan (cap 4)** otherwise
5. Maintain live grid-state for safety + collision-avoidance invariants

Priority order: (1) safety, (2) fastest route for this rider, (3) avoid system gridlock.

## Vehicles ("C")

- Sedans: capacity 4, teal
- Shuttles: capacity 8, orange
- Tick-based motion along assigned route edges
- Per tick:
  - advance along current edge
  - report position to S
  - *(interactive mode only)* run sensor-cone check for scripted obstacles → slow / stop / replan

## Two Modes

### Auto mode
- Speed slider, time-of-day clock, runs a full simulated day
- HUD: active rides, completed rides, avg trip time, **gridlock score** (mean edge congestion)
- City visibly hums with sedans and shuttles satisfying schedule-driven demand

### Interactive mode
- User clicks source on map, then destination
- Side panel walks through the agent's decisions:
  1. Candidate routes drawn on the map (color-coded)
  2. Conflict analysis per route — count of other cars on overlapping edges, projected time
  3. Selected route + rationale (LLM-narrated if API key present, templated otherwise)
  4. **Drive** button starts the simulated trip — scripted obstacles fire mid-route (jaywalker, parked truck, manual-driven car) to demonstrate sensor avoidance and on-the-fly replan

## LLM Narration

- Settings panel with Anthropic API key input (in-memory only, never persisted)
- Model: `claude-sonnet-4-6`
- Direct browser call with `anthropic-dangerous-direct-browser-access: true`
- Input: structured decision data (candidate routes, conflict counts, choice)
- Output: 3–5 sentences of plain-English reasoning
- Fallback when no key: templated narration ("Route B has 2 conflicting cars vs Route A's 5; chose B despite being 80m longer")

## Debug Window

Optional separate browser window (popup) streaming every behind-the-scenes decision in real time. Triggered by an "Open debug window" button in the sidebar. Categories logged:

- `REQUEST` — new ride request enqueued (rider profile, source → destination)
- `DISPATCH` — sedan assignment, with pickup + dropoff route lengths and costs
- `POOL` — shuttle pooling formation, riders grouped, segment count and total blocks
- `PICKUP` / `DROPOFF` — per-car events with rider counts and trip times
- `PLAN` — interactive-mode plan: candidate routes with blocks/conflicts/scores and chosen route
- `OBSTACLE` — sensor cone detection event during interactive drive
- `REPLAN` — detour computation result (new route length and cost, or "no detour available")

Pause / Clear controls in the debug window header. Buffer caps at 500 entries.

## Build Steps (each verifiable)

1. City render + street graph → **verify**: open file, labeled grid city renders correctly
2. Population (100) + schedules + request queue → **verify**: HUD shows requests appearing at expected times-of-day
3. Routing + car movement (auto mode) → **verify**: cars visibly travel origin → destination, drop off, return to idle
4. Conflict-aware dispatching + shuttle pooling → **verify**: shuttles carry 2+ riders to same destination; gridlock score reacts to fleet density
5. Interactive mode: click-to-pick source/dest, candidate routes drawn, side-panel decision walkthrough → **verify**: clicking shows routes + scored analysis
6. Sensor cone + scripted obstacles + replan during interactive drive → **verify**: jaywalker appears, car slows/stops/replans
7. LLM narration with templated fallback → **verify**: works with and without API key
8. Polish: mode switcher, time/speed controls, HUD, legend → **verify**: both modes usable end-to-end
