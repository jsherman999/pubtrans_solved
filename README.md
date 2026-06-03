# pubtrans_solved

A 2D simulation of a small metro area served by an LLM/agent-controlled fleet of autonomous vehicles (sedans + shuttles).

## Quick start

Open `index.html` in any modern browser. No install, no build step.

## What it does

- Renders a 20×20-block city with 4 employers, 2 groceries, 2 schools, 1 train station, and residential clusters
- Simulates 100 humans with daily schedules that generate ride requests
- Dispatches sedans (4-passenger) and shuttles (8-passenger, pooled) along agent-chosen routes that balance speed against system gridlock

## Two modes

**Auto** — a sped-up simulated day. The city hums with cars satisfying schedule-driven demand. HUD shows active rides, completed rides, average trip time, and a live gridlock score.

**Interactive** — click a source then destination. The agent's decision is walked through step-by-step: candidate routes, conflict analysis, chosen route, then a simulated trip with scripted obstacles (jaywalker, parked truck, manual-driven car) demonstrating sensor avoidance and on-the-fly replan.

**Driver POV** — same source/destination flow as Interactive, but during the drive the canvas splits into a small top-down mini-map (with the focal car highlighted and its view cone shown) and a wireframe first-person view from inside the car. Roads, lane stripes, building faces, other cars in the fleet (at their real positions), and scripted jaywalkers are projected from the car's perspective. When the sensor cone detects an obstacle, the FPV shows a red BRAKE overlay; the route then replans and the camera swings onto the detour.

## LLM narration (optional)

In interactive mode, paste an Anthropic API key in the Settings panel to have Claude Sonnet 4.6 narrate the agent's reasoning in plain English. Without a key, a templated explanation is shown instead. The key is held in memory only — never persisted.

## Debug window

Click **Open debug window** in the sidebar to open a separate browser window that streams every behind-the-scenes decision — new ride requests, dispatch assignments, shuttle pooling, pickups/dropoffs, route planning in interactive mode, sensor detections, and replans. The debug window has its own Pause and Clear controls.

## See also

- [`Prompt`](./Prompt) — original problem statement
- [`PLAN.md`](./PLAN.md) — architecture and build steps
