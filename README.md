# Firefly

**Firefly coordinates AI agents, robots and people when work in a truck yard goes off plan, and keeps a verified record of every outcome.**

[Live site](https://firefly-yard-c15.netlify.app) · Demo video: _link coming_ · Built in one day at c0mpiled-15 (Penn State, 11 October 2026)

> **Status: hackathon prototype.** Firefly runs on a simulated yard and has not been tested with operators or customers. The problem framing below is our working hypothesis, and we expect it to change as we talk to people who run yards.

---

## The short version

**The problem.** Yards are starting to use autonomous trucks and other robots. When one of them hits something unexpected (a closed dock door, a pallet in the path), it stops itself, which is the safe thing to do. Deciding what happens next usually still falls on a supervisor: find another door, get someone with a forklift over, make sure people are clear, then let the truck move again. That coordination happens over radios and phone calls, and what actually happened is rarely written down.

**What Firefly does.** When a truck reports that it's blocked, an AI agent takes the case and works it through to a verified outcome:

1. It looks at the job and the yard, and finds the options (for example, another compatible door).
2. It acts through **checked actions only**. Every reservation, move request and task offer is validated against the live state of the yard at the moment it runs, and a rejected action comes back to the agent as a real failure it has to handle.
3. When a person is needed, it offers the task to someone qualified, on their phone.
4. The supervisor can tell it things no sensor knows ("F-3 is leaking, keep it out of assignments"). That becomes a site rule that cancels conflicting work straight away and shapes later decisions.
5. Safety-critical steps wait for people. A truck that stopped near workers resumes only after they report clear and the supervisor approves.
6. The case closes only when the truck's own controller confirms it arrived. The agent saying "done" closes nothing.

**Why it could matter.** The supervisor spends time on judgment calls instead of logistics, stalled moves get worked immediately, and every exception leaves a record of what happened, who decided, and how it ended. We think that record is the long-term asset: it's what lets the same loop run across more sites, more machines and more kinds of work.

---

## What you'll see in the demo

Two problems happen at once in a yard with 8 simulated trucks and 8 dock doors.

- **Door 3 closes.** Truck T-2 stops and reports `BLOCKED`. The agent tries Door 4, loses it to another truck that got there first (a real rejection from the reservation check), re-plans to Door 2, and closes the case when the truck confirms arrival. No human input.
- **A pallet blocks the Door 7 approach.** Truck T-5 stops. That job can't switch doors, so the agent needs a person. It offers the task to a worker on forklift F-3, the supervisor types that F-3 is leaking, the offer is withdrawn, and the task goes to a worker on F-4. The worker clears the pallet from their phone. An early attempt to resume is rejected until people are clear and the supervisor approves.
- **Later, another pallet.** The agent skips F-3 on its own, because of the supervisor's rule.

---

## What's real and what's simulated

- **Simulated:** the trucks, their controllers and sensors ("Twin mode"). No real robots or site systems are connected.
- **Real:** the AI. A Claude model chooses every action, and each line in the console maps to a real model call, action result, controller message or human input.
- **Staged:** the demo problems themselves (the closed door, the pallet, the truck that takes Door 4). How the agent responds is not scripted.
- **Self-reported:** a worker's "people clear" is their report, not independent sensing. The resume procedure is a demo procedure, not a certified safety system.
- **No invented numbers.** Anything shown on screen is counted from the events of the run itself. Message names are modeled on VDA 5050; Firefly is not certified against it.

---

## What we don't know yet

These are the assumptions we most want challenged:

1. **Who owns a blocked move today**, inside a shipper, carrier, 3PL or autonomous-truck operator, and how it's handled on a real shift.
2. **How often exceptions happen and what they cost**, in stalled trucks, labor and missed appointments.
3. **Whether mixed fleets are coming**, meaning autonomous and human-driven trucks working the same yard, and how soon.
4. **Where Firefly fits** next to yard management systems and the software robot vendors ship with their own machines.

## Directions we're exploring (subject to change)

- **Exception handling for conventional yards.** The same loop works with human drivers and yard jockeys, which doesn't depend on autonomy arriving first.
- **Remote assistance for autonomous trucking hubs.** A console for the teams who monitor autonomous trucks and step in when one needs help.
- **The verified work record as the product.** An auditable history of every exception and how it ended, useful for operations reviews, safety cases and vendor accountability.

If you work in or around yards, hubs or terminals and any of this matches (or doesn't match) what you see, we'd like to hear from you. Open an issue or contact us on GitHub.

---

## How it works (for technical readers)

```text
truck controller ──BLOCKED──▶ incident ──▶ agent (Claude, tool loop)
                                              │
                         checked actions ◀────┤──▶ reads: job, doors, workers,
            (reserve door, request move,      │          site history, lane history
             offer task, save rule, resume)   │
                                              ▼
      people on phones  ◀── task / approval ──┘      every step ──▶ event log
      supervisor console ── facts, approvals ──▶                  (and MongoDB)
                                                                       │
                     controller confirms arrival ──▶ verify_outcome ──▶ case closed
```

- **Server:** Node + Hono. The yard is held in memory on the server and changed only through an append-only event log. Screens subscribe over Server-Sent Events.
- **Agent:** Anthropic TypeScript SDK with `claude-opus-5-5` in a hand-written tool loop, one run per incident; two run at once. 17 tools: reads such as `list_compatible_doors` and `list_eligible_workers`, and checked actions such as `reserve_door`, `offer_task`, `save_correction`, `request_resume` and `verify_outcome`.
- **Safety by construction:** every action validates the current state itself, inside one synchronous step, so two agents can't double-book a door or a worker. A supervisor's correction cancels conflicting pending work immediately.
- **Screens:** Vite, React and react-three-fiber. The 3D yard only moves a truck after the matching controller event.
- **Memory:** "memory" means retrieval, not training. Agents read recorded outcomes (`get_site_history`), and with MongoDB connected they can also query lane statistics computed from a public trucking dataset (`get_lane_history`, the Delhivery dataset on Kaggle, which is not stored in this repo and is not data from this yard).
- **Decisions:** the reasoning behind the design is in [`docs/adr/`](docs/adr/README.md). Contributor and agent rules are in [`AGENTS.md`](AGENTS.md).

---

## Run it locally

Needs Node 22.12+ and pnpm.

```bash
pnpm install
cp .env.example .env   # add your ANTHROPIC_API_KEY (and optionally MONGODB_URI)
pnpm dev               # server on :8787, site on http://localhost:5173
```

- Landing story: `http://localhost:5173` · console: `http://localhost:5173/console.html` · worker phone: `http://<laptop-ip>:5173/worker.html`
- In the console, shift the lever into **D** to run the demo scenario. Use **Cause a problem** to close a door, drop a pallet, take a forklift out of service or send a worker home.
- `pnpm test` · `pnpm typecheck` · `pnpm build`

The hosted [live site](https://firefly-yard-c15.netlify.app) serves the landing story only. The console needs the Firefly server running, so run it locally to try the live yard.

### Flight recorder (MongoDB, optional)

Set `MONGODB_URI` in `.env` and every event of every shift is also written to MongoDB. Without it, Firefly runs the same, in memory only.

- **Replay any past shift**, event for event: `GET /api/shifts` lists them, then open `http://localhost:5173/console.html?replay=<shift>`. A replay is a recording, with long waits shortened; nothing in it is happening live.
- **Memory across shifts:** `get_site_history` also returns counts and averages from earlier shifts, computed by aggregation pipelines in MongoDB.
- **Rules survive a restart,** and any past shift's reliability panel is at `GET /api/reliability?shift=<shift>`.
- **Lane history:** import the Delhivery CSV with `pnpm tsx scripts/import-delhivery.ts <path-to-csv>` to enable `get_lane_history`.

The live yard never waits on the database. Events are written in the background, and if MongoDB is unreachable the yard carries on and logs the failure.

---

## Team

Built by [Muhammad Usman Siddiqui](https://github.com/usmannsiddiqui) (screens, 3D yard, design) and [Rayyan](https://github.com/Rayyan2235) (server, simulator, agent loop) at c0mpiled-15, for YC's Fall 2026 Request for Startups, *New Operating Systems for the Physical World*.
