# Plan: Public Transportation Simulation

A 2D simulation of a small metro area served by an LLM/agent-controlled fleet of autonomous vehicles.

## Deliverable

A single `index.html` file (HTML + Canvas + vanilla JS, no build step). Double-click to run in any modern browser.

## Tech Decisions

| Question | Choice |
|----------|--------|
| Stack | Single-file HTML + Canvas + JS |
| Dispatcher | Deterministic Dijkstra with congestion-weighted edges, labeled "the agent". K-shortest is used only for Interactive-mode candidate visualization |
| LLM use | Claude Sonnet 4.6, narrates *interactive mode* decisions only (API key pasted in UI, in-memory) |
| Visual style | Clean geometric — labeled colored blocks, smooth car motion, HUD |
| Population | 100 simulated humans |
| Initial scope | Larger build (~1000–1400 lines) — schedules, pooling, obstacles, gridlock metric |
| As-built size | ~2500 lines after iterations (Driver POV, spectate, auto-mode obstacles, surge events, traffic infrastructure) |

## City Model

- 12×12 intersection / 11×11 block grid; streets are the lines *between* blocks
- Building types placed across the grid:
  - 4 employers
  - 2 grocery stores
  - 2 schools
  - 1 train station (drop-off for transfer out of the metro area)
  - 8 residential clusters
  - remainder empty / streets
- Street graph: nodes = intersections, edges = segments. Each edge tracks live claims from cars whose routes traverse it. Edge weight for routing = `1 + CONGESTION_W × claims.size`
- Traffic infrastructure (randomized at init): 2 one-way edges, 4 stoplights (cycling H-green → yellow → V-green → yellow), 4 stop signs

## Population (100 humans)

Each human has:
- a home block
- a profile: worker (→ employer), student (→ school), commuter (→ train station), errand-runner (→ grocery)
- a daily schedule that triggers ride requests at the appropriate simulated time-of-day

## Dispatcher ("S")

Deterministic, called "the agent" in narration.

On each dispatch tick (every tick in Auto mode):
1. Group queued requests by destination
2. For each destination, pool 2+ "fresh" (< 30 sim-min old) requests into a shuttle if one is idle; otherwise assign each remaining request to an idle sedan (or shuttle if no sedans are free)
3. For each assignment, run **Dijkstra** on the street graph using edge weight `1 + CONGESTION_W × edges[k].claims.size` (the routing one-way restrictions are also enforced inside the Dijkstra loop)
4. Claim the route's edges so subsequent dispatch decisions account for the new traffic
5. Maintain live grid-state for safety + collision-avoidance invariants

For Interactive mode the planner additionally produces **k-shortest** alternates (Yen-style edge-banning of the primary's edges) so the user can see candidate routes ranked by score.

Priority order: (1) safety, (2) fastest route for this rider, (3) avoid system gridlock.

## Vehicles ("C")

- 28 sedans (capacity 4, teal) + 14 shuttles (capacity 8, orange) — total fleet of 42
- Tick-based motion along assigned route edges
- Per tick:
  - advance along current edge (slowed to 40% when in a manual-driven-car follow window)
  - on reaching an intersection, sense for obstacles (any kind) and traffic controls; enter the appropriate wait state if triggered
  - report position to S
  - kind-specific obstacle responses: jaywalker → wait then resume same route; parked truck → detour; manual-driven car → safe-follow at 40% speed

## Three Modes

### Auto mode
- Speed slider, time-of-day clock, runs a full simulated day
- HUD: active rides, completed rides, avg trip time, **gridlock score** (mean edge congestion)
- City visibly hums with sedans and shuttles satisfying schedule-driven demand
- **Random obstacles spawn periodically** (~every 4–12s) on the street grid: jaywalkers, parked trucks, manual-driven cars — all cars in the fleet sense and react with the same kind-specific behaviors as in Driver POV mode (jaywalker brake-and-wait, truck detour, manual car slow-follow)
- **Scheduled surge events** inject bursts of pickup requests at simulated school-day starts/ends and train arrivals/departures. A teal banner announces each event in both the top-down and FPV views; the dispatcher pools concurrent same-destination requests into shuttles, illustrating the resulting traffic pattern. Configured times in `eventSchedule`; the day reset re-arms each event.
- **Traffic infrastructure** placed at init time: 2 random one-way streets (yellow arrows in top-down + ground arrows in FPV; Dijkstra respects direction), 4 random stoplights (cycling H-green → yellow → V-green → yellow with ~80/12-tick phases; cars wait until their direction is green), and 4 random stop signs (brief 10-tick halt). All depicted in both top-down and FPV; HUD overlay in FPV when stopped at a light or sign.
- **Spectate**: click any active car on the grid → POV opens for that car alongside a mini-map (speed auto-drops to 1×); auto continues running, you watch traffic / pickups / dropoffs / obstacle reactions from inside the chosen car. Click another car to switch; **Exit POV** returns to full-map view. If the spectated car finishes its trip, the FPV stays parked at the drop-off until you pick a new car.

### Interactive mode
- User clicks source on map, then destination
- Side panel walks through the agent's decisions:
  1. Candidate routes drawn on the map (color-coded)
  2. Conflict analysis per route — count of other cars on overlapping edges, projected time
  3. Selected route + rationale (LLM-narrated if API key present, templated otherwise)
  4. **Drive** button starts the simulated trip — scripted obstacles fire mid-route (jaywalker, parked truck, manual-driven car) to demonstrate sensor avoidance and on-the-fly replan

### Driver POV mode
- Same source/destination/Plan/Drive flow as Interactive
- Canvas splits: small top-down mini-map (with the focal car highlighted by a yellow ring and its FPV view cone overlaid) + larger wireframe first-person view rendered from inside the car
- FPV pipeline (Canvas 2D + hand-rolled perspective):
  - Camera at car position with eye height; heading interpolated for smooth turns
  - World points → camera-space via 2D rotation; near-plane polygon clipping; perspective divide
  - Roads with dashed centerlines, intersection plates, lane-correct other cars (right-of-heading with direction-aware head/taillights), building faces shaded by distance, all three obstacle kinds (jaywalker wireframe figure with walking gait, parked truck, manual-driven car), and traffic infrastructure (hanging stoplights, pole-mounted stop signs, ground-plane one-way arrows)
  - HUD: car id, state, route progress; full-screen overlay for BRAKE (obstacle), RED LIGHT, STOP SIGN states
- The FPV is purely a visualization — routing, obstacle detection, and replanning still run in the same deterministic dispatcher used in Auto and Interactive modes

## LLM Narration

- Settings panel with Anthropic API key input (in-memory only, never persisted)
- Model: `claude-sonnet-4-6`
- Direct browser call with `anthropic-dangerous-direct-browser-access: true`
- Input: structured decision data (candidate routes, conflict counts, choice)
- Output: 3–5 sentences of plain-English reasoning
- Fallback when no key: templated narration ("Route B has 2 conflicting cars vs Route A's 5; chose B despite being 80m longer")

## Debug Window

Optional separate browser window (popup) streaming every behind-the-scenes decision in real time. Triggered by an "Open debug window" button in the sidebar. Categories logged:

- `REQUEST` — new ride request enqueued (from a scheduled human or a surge event)
- `DISPATCH` — sedan assignment, with pickup + dropoff route lengths and costs
- `POOL` — shuttle pooling formation, riders grouped, segment count and total blocks
- `PICKUP` / `DROPOFF` — per-car events with rider counts and trip times
- `PLAN` — interactive-mode plan: candidate routes with blocks/conflicts/scores and chosen route
- `OBSTACLE` — auto-spawn event or sensor cone detection (jaywalker / parked truck / manual-driven car)
- `REPLAN` — detour computation result, jaywalker cleared, or slow-follow resumed
- `TRAFFIC` — car halts at or proceeds through a stoplight or stop sign
- `EVENT` — a scheduled surge event fires (school start/end, train arrival/departure)
- `SYSTEM` — spectate start/exit, debug window open

Pause / Clear controls in the debug window header. Buffer caps at 500 entries.

## Build Steps (original plan, each verifiable)

1. City render + street graph → **verify**: open file, labeled grid city renders correctly
2. Population (100) + schedules + request queue → **verify**: HUD shows requests appearing at expected times-of-day
3. Routing + car movement (auto mode) → **verify**: cars visibly travel origin → destination, drop off, return to idle
4. Conflict-aware dispatching + shuttle pooling → **verify**: shuttles carry 2+ riders to same destination; gridlock score reacts to fleet density
5. Interactive mode: click-to-pick source/dest, candidate routes drawn, side-panel decision walkthrough → **verify**: clicking shows routes + scored analysis
6. Sensor cone + scripted obstacles + replan during interactive drive → **verify**: jaywalker appears, car slows/stops/replans
7. LLM narration with templated fallback → **verify**: works with and without API key
8. Polish: mode switcher, time/speed controls, HUD, legend → **verify**: both modes usable end-to-end

## Iterations beyond the initial plan

The above 8 steps were the original delivery. The simulation grew through several follow-up iterations:

9. **Fleet scaling** — bumped sedans from 14 → 28, shuttles from 6 → 14, schedule jitter ±15 → ±30, dispatch every tick (not every other) — keeps the queue under 10 even during the morning commute spike
10. **Differentiated obstacle behaviors** — jaywalker (brake then resume same route, with animated crossing), parked truck (Dijkstra detour with banned edge), manual-driven car (40% speed follow window) — kind-specific reasoning in narration and debug log
11. **Driver POV mode** — third mode with a wireframe FPV renderer (hand-rolled perspective, near-plane polygon clipping, painter's-algorithm sorting). Split layout: small top-down mini-map + larger FPV canvas. Smooth heading interpolation, right-lane car positioning with direction-aware head/taillights, distance-shaded building faces
12. **Auto-mode obstacles** — same spawn pool now seeds Auto mode (~every 4–12s on a random unoccupied edge); the unified `checkObstacle` matches by `edgeKey` so every car in the fleet reacts the same way
13. **Spectate** — click any active car in Auto mode → POV opens for that car, sim continues at 1×, click another to switch, **Exit POV** to return. Stay-parked behavior when the spectated car finishes its trip
14. **Surge events** — `eventSchedule` injects bursts of pickup requests at school day starts/ends and train arrivals/departures; announces via a teal banner in both the top-down and FPV views
15. **Traffic infrastructure** — 2 random one-way streets (Dijkstra-enforced), 4 random stoplights (cycling phases), 4 random stop signs. All depicted in both top-down and FPV; HUD overlay in FPV when stopped at a light or sign
16. **Documentation pass** — install/run howto, fact-check pass to match the as-built code
