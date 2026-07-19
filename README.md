# Simzoner-
Simzoner is a highly distributed, ultra-low-latency multi-agent traffic simulation platform built natively on the Cloudflare Developer Stack to model real-time vehicular behavioral logic and spatial contention. The project leverages an asynchronous, edge-driven architecture to simulate concurrent autonomous profiles (including veichles such the Tesla Cybertruck, BMW M3 Series, and Waymo all driving on HOV Lanes Acrosss Austin, Galveston , Cincanitti , New York and etc.) within constrained runtime environments without reliance on traditional centralized databases or heavy server overhead. By separating high-throughput  execution from resource-intensive cognitive training loops, the platform demonstrates how massive multi-agent simulations can be deployed directly to the network edge, ensuring global data consistency, deterministic state changes, and cost-free execution at scale.

Link to Deployed Project on Cloudflare Workers :  https://simzoner-web.pages.dev/


ns BETWEEN ticks, never inside.   ║
  ║  Cost: CPU                        ║  Cost: NEURONS ← the binding limit   ║
  ╚═══════════════════════════════════╩══════════════════════════════════════╝

                    ratio: 6,000 ticks : 20 decisions  =  300 : 1
                    one policy is executed for ~300 ticks before it is revisited
```

### 5.2 Timing diagram — one race, three cars

Sim-time runs left to right. `█` = a tick. `◆` = an LLM decision point.

```
 segment:      S1                    S2                  S3           S4
 boundary:  ───┬─────────────────────┬───────────────────┬────────────┬──►
               │                     │                   │            │
 cybertruck  ◆ ████████████████████  ◆ ███████████████   ◆ ██████████ ◆
               ↑ policy P1_ct        ↑ P2_ct             ↑ P3_ct
               │ "HOV, 0.95 cap,     │ "hold lane, watch │ "Webster —
               │  aggressive"        │  the M3"          │  merge early"
               │                     │                   │
 m3          ◆ ██████████████████████ ◆ ██████████████    ◆ ███████████ ◆
               ↑ P1_m3               ↑ P2_m3             ↑ P3_m3
               │ "GP lanes — not     │                   │
               │  HOV-eligible"      │                   │
               │                     │                   │
 waymo       ◆ █████████████████████████ ◆ █████████████    ◆ ██████████ ◆
               ↑ P1_wy               ↑ P2_wy               ↑ P3_wy
               │ "HOV if AV_POLICY   │                   │
               │  allows; 0.85 cap"  │                   │

               └──── ~300 ticks ─────┘
                  ZERO LLM calls in here.
                  The engine is alone with the policy.
```

**The cars do not cross boundaries at the same tick.** They're at different positions. Decision
points are **staggered** — and that's correct. Nothing synchronizes them.

### 5.3 The insight that makes staggering free

**Sim time is not wall time.**

When a car crosses a boundary, the DO **suspends the tick loop**, awaits the LLM call, applies the
returned policy, and resumes. The LLM call takes 800 ms of wall time. It costs the race **zero**
sim-time, because the sim clock is a counter, not a clock.

```
  wall time  ├──ticks──┤├── 800ms LLM await ──┤├──ticks──┤├─ LLM ─┤├──ticks──┤
  sim time   ├──ticks──┤                       ├──ticks──┤         ├──ticks──┤
                        ↑
                        sim clock does not advance.
                        No distortion. No race condition. No "the M3 got ahead
                        while the Waymo was thinking."
```

This is why an LLM in the *tick loop* is impossible but an LLM *between* ticks is free. It's also why
the DO must be single-threaded and authoritative: suspend-await-resume is only safe because nothing
else can touch the world while the agent thinks. **The DO's concurrency model is doing real work
here, not just holding state.**

### 5.4 Sequence — one boundary crossing

```
  TickLoop          RaceDO            Cache(SQLite)     BudgetDO      WorkersAI
     │                 │                    │               │              │
     │ car crosses     │                    │               │              │
     │ boundary S2 ───►│                    │               │              │
     │  (loop suspends)│                    │               │              │
     │                 │ build observation  │               │              │
     │                 │ (world snapshot,   │               │              │
     │                 │  rivals, HOV elig, │               │              │
     │                 │  energy, next seg) │               │              │
     │                 │                    │               │              │
     │                 │ quantize ─────────►│               │              │
     │                 │  (gap, closing,    │               │              │
     │                 │   lane, risk,      │               │              │
     │                 │   seg archetype)   │               │              │
     │                 │◄── HIT ────────────┤               │              │
     │                 │    └─► apply policy, resume. 0 neurons. ──────────┤
     │                 │                    │               │              │
     │                 │◄── MISS ───────────┤               │              │
     │                 │                    │               │              │
     │                 │ reserve(1 decision)────────────────►│              │
     │                 │◄── DENIED (budget exhausted) ───────┤              │
     │                 │    └─► FALLBACK policy (§8.1). Race continues.     │
     │                 │        degraded = true. 0 neurons.                │
     │                 │                    │               │              │
     │                 │◄── GRANTED ────────┼───────────────┤              │
     │                 │                    │               │              │
     │                 │ env.AI.run(persona + observation) ─────────────────►│
     │                 │   max_tokens ~64-100, JSON mode (ARCH §4, §6)      │
     │                 │◄──────────────────── {"action":"pass","risk":0.7} ─┤
     │                 │                    │               │              │
     │                 │ parse + VALIDATE + CLAMP           │              │
     │                 │  └─ invalid? → FALLBACK (§8.2)     │              │
     │                 │                    │               │              │
     │                 │ write policy ─────►│               │              │
     │                 │ append decision log│               │              │
     │                 │ checkpoint (§8.3) ►│               │              │
     │                 │ commit(actual) ────────────────────►│              │
     │                 │                    │               │              │
     │◄── policy ──────┤                    │               │              │
     │  (loop resumes) │                    │               │              │
     │ ~300 ticks ─────┤ ← no further contact with anything  │              │
```

**Every path out of that diagram ends in a policy.** Hit, miss, denied, malformed — the tick loop
always gets a `Policy` and always resumes. There is no branch where the race stops. That's §8's
thesis, visible in the sequencing.

### 5.5 Cache-key quantization is where the leverage is

`ARCHITECTURE.md` §4 mitigation #5 calls for bucketing the situation into a discrete key. The design
consequence: **the cache key must derive from the observation, not from raw floats.**
`gap = 43.7218 m` never repeats. `gap_bucket = "30-50m"` repeats constantly.

```
key = hash(persona_id, seg_archetype, gap_bucket, closing_bucket, lane_class,
           hov_eligible, energy_bucket, rival_order)
```

Coarser buckets ⇒ higher hit rate ⇒ fewer neurons ⇒ more races/day — and worse tactical resolution.
**That trade is a tuning dial, and it is [OPEN].** The key must include `persona_id`: the whole point
of three cars is that they answer the same situation differently, so a shared entry across personas
would erase the thing the project demonstrates.

### 5.6 Why a policy and not an action

A per-tick action would be `{ accelerate: 0.8, lane: -1 }` — valid for 100 ms. A **policy** is
`{ targetLane: -1, targetSpeedFraction: 0.95, useHovLane: true, ... }` — an *intent* the engine
executes for ~300 ticks, resolving it against traffic, gaps, cooldowns, and the speed cap every tick.

```
   LLM says:  "run the HOV lane hard until Buda, then settle"
                              │
                              ▼
   Engine does: 300 × { check cap · check gap · check cooldown · check eligibility
                        · integrate · clamp v ≤ limit+10 · commit }
```

**The engine is not obeying the LLM. It is interpreting it under hard constraints.** The policy is a
*request*, and `SIMULATION_RULES.md` §3.1 is explicit that the speed cap is a clamp in the integrator
that no policy can lift — `targetSpeedFraction` is 0..1 *of the cap*, so an illegal speed is
**unrepresentable** rather than rejected. The agent cannot cheat because the type won't let it
express cheating.

This is what makes a bad LLM decision **survivable**: the worst a broken policy can do is pick a slow
lane. It cannot violate physics, exceed the cap, or teleport.

---

## 6. Data flow, end to end

```
 ┌─ BUILD TIME (training plane, §1) ──────────────────────────────────────────┐
 │  FastAPI: generate → train → evaluate → export                             │
 │        └──► model.json + golden_vectors.json ──► committed, bundled        │
 │             (happens a few times ever, not per race)                       │
 └────────────────────────────┬───────────────────────────────────────────────┘
                              │  ═══ PLANE BOUNDARY ═══
 ┌─ SETUP ────────────────────▼───────────────────────────────────────────────┐
 │  user picks matchup + route + start time + seed                            │
 │              │                                                             │
 │              ▼                                                             │
 │  Worker: retrieve                                                          │
 │     ├─► Vectorize ──► vehicle spec docs + route/segment corpus             │
 │     │      (fallback: KV warm copy — §8.4)                                 │
 │     └─► KV ─────────► route config, segment table, persona templates       │
 │              │                                                             │
 │              ├─► PREDICT: sigmoid(Σ wᵢxᵢ + b) from model.json              │
 │              │      └─► p(win) + confidence, labeled synthetic (ML §1.3)   │
 │              │          fails → hide the prediction, race unaffected (§8.7)│
 │              ▼                                                             │
 │  ASSEMBLE PERSONAS  ← ONCE, here, not per call                             │
 │     persona_prompt[car] = template(persona) + retrieved specs + rules       │
 │     └─► frozen for the whole race. See §6.1.                               │
 │              │                                                             │
 │              ▼                                                             │
 │  BudgetDO.reserve(3 cars × ~20 decisions)  ← admission control (§8.1)      │
 │              │                                                             │
 │              ▼                                                             │
 │  RaceDO.init(RaceConfig{ seed, route, vehicles[3], startTime, rules })     │
 └────────────────────────────┬───────────────────────────────────────────────┘
                              │
 ┌─ DECISION LOOP (§5) ───────▼───────────────────────────────────────────────┐
 │    ┌──────────────────────────────────────────────────────┐                │
 │    │  tick × ~300  ──►  boundary?  ──no──┐                │                │
 │    │        ▲                            │                │                │
 │    │        └────────────────────────────┘                │                │
 │    │                    │ yes                             │                │
 │    │                    ▼                                 │                │
 │    │        cache → budget → LLM → validate → policy      │                │
 │    │                    │                                 │                │
 │    │                    ├─► decision log (append-only)    │                │
 │    │                    └─► checkpoint (SQLite)           │                │
 │    └──────────────────────────────────────────────────────┘                │
 │                     repeat until finish line                               │
 │                                                                            │
 │    accumulating: segment splits, penalties, hovDistance_m,                 │
 │                  lane_changes, energyStops  ← AGGREGATES, not per-tick      │
 │                  telemetry. See §6.2.                                      │
 └────────────────────────────┬───────────────────────────────────────────────┘
                              │
 ┌─ PERSIST ──────────────────▼───────────────────────────────────────────────┐
 │   D1  ◄── RaceResult { seed, configHash, engineVersion, finishOrder,        │
 │                        segmentSplits, degraded, MOCK_DATA: true }           │
 │   D1  ◄── decision log (~60 rows) — batched, ONE write (D1: 50 queries      │
 │                                      per invocation, ARCH §3)               │
 │   R2  ◄── replay blob, ONLY if it outgrows the DO/D1 row (§7)               │
 └────────────────────────────┬───────────────────────────────────────────────┘
                              │
 ┌─ REPLAY ───────────────────▼───────────────────────────────────────────────┐
 │   browser GET /race/:id ──► { seed, config, policies[~60] }   ~few KB       │
 │                                    │                                       │
 │                                    ▼                                       │
 │   SAME deterministic engine, compiled into the client bundle               │
 │   replay(seed, config, policies) ⇒ bit-identical race                      │
 │                                    │                                       │
 │   ZERO server CPU · ZERO neurons · instant scrubbing · shareable as a URL   │
 └────────────────────────────────────────────────────────────────────────────┘
```

**Note the training plane also *consumes* this loop.** `ML_APPROACH.md` §8 step 2 generates ~50k
races — by driving the **same deterministic TS engine**, headless, with noise injected (§3.1 there)
and **no LLM at all** (fallback policies, §8.1 here, make that free and deterministic). The training
plane doesn't need a parallel physics implementation, and must never have one: two engines would
drift, and the model would learn a simulator that isn't the one being served.

### 6.1 Personas are precomputed, and that's a budget decision

The persona system prompt is assembled **once at setup** and frozen for the race. Never re-derived,
never re-retrieved, never regenerated by a model.

The prompt is the *large* half of every call (~800 input tokens, `ARCHITECTURE.md` §4). Re-deriving
it per call would mean re-running retrieval ~60 times per race and re-paying assembly for a string
that never changes. And per `ARCHITECTURE.md` §5, **there is no training on Cloudflare** — persona
*is* the prompt. It's the only place vehicle character lives, so it must be stable, inspectable, and
cheap.

Frozen personas also make the decision log interpretable later: `(persona_hash, observation) →
policy` reproduces the reasoning context without needing the model.

### 6.2 Telemetry: aggregate, don't store

`TickTelemetry` (`SIMULATION_RULES.md` §5.1) is ~15 fields × 6,000 ticks × 3 cars ≈ **270,000 rows
per race**. Absurd on any tier, unnecessary on this one.

**We don't store telemetry. We store the seed.** The tick loop accumulates only aggregates
(`segmentSplits`, `hovDistance_m`, `penalties_s`, `lane_changes`). Anyone who wants tick 8,412
replays the seed and reaches tic# SimZoner — System Design

**Status:** Draft v0.1 — design only, no implementation
**Scope:** The planes, the boxes, the data flow, the per-car agent architecture, and the sequencing
**Audience:** A new contributor who needs to understand how the thing actually works

---

## 0. What this doc is, and what it isn't

This is the **system design** layer. It answers: *what are the components, how do the three cars
connect as systems, where does Python fit, and in what order does anything happen?*

It deliberately does **not** restate its siblings. Read them for their layers:

| Doc | Layer | Read it for |
|---|---|---|
| `docs/PROBLEM_STATEMENT.md` | Why | What this project is for, and what it doesn't claim |
| `docs/ARCHITECTURE.md` | Platform | Cloudflare components, **verified free-tier limits**, neuron budget, where LangChain fits |
| `docs/SIMULATION_RULES.md` | World | Lanes, speed cap, HOV eligibility, routes, data shapes, determinism contract |
| `docs/VEHICLE_SPECS.md` | Parameters | Per-vehicle physics values and their provenance |
| `docs/ML_APPROACH.md` | Learned model | The pre-race outcome predictor (**not** the driver agents — see §7 there) |
| **This doc** | **System** | **The two planes, component wiring, the three-car agent structure, the decision loop, failure modes** |

**Every Cloudflare platform limit cited here is sourced in `ARCHITECTURE.md` §3.** Numbers are quoted
here only where a design decision is unintelligible without them. If a number in this doc disagrees
with `ARCHITECTURE.md`, `ARCHITECTURE.md` wins.

Claims about **Python Workers** are new research and are tagged `[VERIFIED]` / `[UNVERIFIED]` with
source URLs inline (§2), matching the sibling docs' standard.


### 0.1 The one-sentence version

> **The LLM decides. The engine computes. The seed reproduces.**
> — `SIMULATION_RULES.md` §6.3

And the one this doc adds:

> **The training plane learns offline and ships weights. The serving plane executes at the edge in
> 10ms. The only thing that crosses is a JSON file.**

---

## 1. The two planes

The most important structural fact about this system is that it has **two planes with completely
different constraints**, and exactly one narrow artifact crossing between them.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║  TRAINING PLANE — offline · Python-native · no latency budget                     ║
║                                                                                  ║
║   Constraint profile:  CPU: minutes-to-hours    Latency: irrelevant              ║
║                        Memory: GBs              Availability: not required       ║
║                        Runs: on demand, locally or in CI                         ║
║                                                                                  ║
║   ┌────────────────────────────────────────────────────────────────────┐         ║
║   │  FastAPI service  (uvicorn, local / CI / container)                 │         ║
║   │                                                                    │         ║
║   │   POST /datasets/generate   → drive the TS engine, N seeded races   │         ║
║   │   GET  /datasets/{id}       → races.parquet + manifest              │         ║
║   │   POST /train               → sklearn: split → fit → evaluate       │         ║
║   │   GET  /models/{id}/report  → baselines, AUC, calibration (ML §6.2) │         ║
║   │   POST /models/{id}/export  → model.json + golden vectors           │         ║
║   │   GET  /registry            → what's trained, what's shipped        │         ║
║   │                                                                    │         ║
║   │   owns: numpy · pandas · scikit-learn · joblib · matplotlib         │         ║
║   └────────────────────────────────────────────────────────────────────┘         ║
╚═══════════════════════════════════╤══════════════════════════════════════════════╝
                                    │
                    ┌───────────────▼────────────────┐
                    │   THE BOUNDARY                 │
                    │                                │
                    │   model.json  (a few KB)       │
                    │   ├─ coef_, intercept_         │
                    │   ├─ scaler mean_, scale_      │
                    │   ├─ feature_names (ORDERED)   │
                    │   └─ schema_version            │
                    │                                │
                    │   + golden_vectors.json (tests)│
                    │                                │
                    │   ── BUILD-TIME ARTIFACT ──    │
                    │   Not a request. Not a hop.    │
                    │   Not a runtime dependency.    │
                    │   Committed & bundled.         │
                    └───────────────┬────────────────┘
                                    │
╔═══════════════════════════════════▼══════════════════════════════════════════════╗
║  SERVING PLANE — edge · TypeScript/V8 · latency-critical                          ║
║                                                                                  ║
║   Constraint profile:  CPU: 10ms/invocation      Latency: user-facing            ║
║                        Memory: small             Availability: required          ║
║                        Runs: 100k req/day, globally, always                      ║
║                                                                                  ║
║   Worker · RaceDO · BudgetDO · Workers AI · Vectorize · D1 · KV · R2 · Pages      ║
║   Model inference = sigmoid(Σ wᵢxᵢ + b) — a dot product. Microseconds. (ML §5.1)  ║
║                                                                                  ║
║   ── UNCHANGED by any of this. ──                                                ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

### 1.1 Why this split is correct rather than arbitrary

The two planes are not two ways of doing the same job. **They have no overlapping requirement.**

| | Training plane | Serving plane |
|---|---|---|
| CPU per unit of work | **minutes–hours** | **10ms** |
| Runtime | CPython, native wheels | V8 / WASM |
| Latency budget | none | user-facing |
| Uptime requirement | none — it can be *off* | required |
| Runs how often | a few times per model | 100k/day |
| Language | Python (because sklearn is Python) | TypeScript (because the edge is) |
| Failure impact | a stale model | a broken site |
| Scaling axis | dataset size | request count |

**Nothing in that table is close.** Two systems whose constraint profiles differ by five orders of
magnitude on the primary axis (CPU) should not be one system. That's the whole argument.

`ML_APPROACH.md` §5.3 already establishes that **training happens locally in Python** — Cloudflare
cannot train (`ARCHITECTURE.md` §5: Workers AI is inference-only, LoRA is bring-your-own-adapter).
What this doc adds is that **that work deserves a real home and a real interface** instead of living
in a folder of ad-hoc notebooks. The FastAPI service *is* that home.

### 1.2 What FastAPI is actually for here

**Not** a second web server serving user traffic. It never sees a user request, and it is not in the
path of a race. It is the **training plane's control surface**: a typed, documented, testable
interface over the four things the ML work actually does — generate, train, evaluate, export.

Concretely, it turns `ML_APPROACH.md` §8's 13-step build order from a checklist into an API:

```
   ML_APPROACH §8 step          →  FastAPI endpoint
   ─────────────────────────────────────────────────────────────
   2. generate 50k races        →  POST /datasets/generate
   3. split by seed group       →  POST /datasets/{id}/split
   4. compute naive baselines   →  GET  /datasets/{id}/baselines
   6. fit logistic regression   →  POST /train
   8. fit GBM as ceiling probe  →  POST /train?family=gbm
   9. evaluate on test (ONCE)   →  POST /models/{id}/evaluate   ← auditable: logs that it ran
   10. export weights → JSON    →  POST /models/{id}/export
```

**The endpoint that earns its keep is `/evaluate`.** `ML_APPROACH.md` §6.3 and §8 step 9 are
emphatic that the test split is touched **once**. A checklist cannot enforce that; a service with a
registry can — it records that the test set was evaluated, for which model, when, and refuses a
silent second look. **Turning a discipline you're supposed to remember into a constraint the system
enforces is the reason to build the interface at all.** That's the honest justification for FastAPI
here, and it's a better one than "the owner wants Python in the diagram."

Pydantic also gives the boundary a schema. `ML_APPROACH.md` §5.3 flags feature ordering as **"the #1
way exported models break"** — a silently reordered vector produces plausible, wrong predictions with
no error. A Pydantic model on the export payload makes that a validation failure at build time
instead of a mystery at the edge (§8.7).

---

## 2. Python Workers — research findings

The owner asked for FastAPI in the architecture. Before drawing the box, here's whether it can run
on Cloudflare. **This determines where the training plane lives, so it's worth the space.**

### 2.1 Verified

| # | Claim | Status | Source |
|---|---|---|---|
| 1 | Python Workers are **in open beta**; require the `python_workers` compatibility flag | **[VERIFIED]** — *"Python Workers are in beta."* / *"You must add the `python_workers` compatibility flag to your Worker, while Python Workers are in open beta."* | [Python Workers docs](https://developers.cloudflare.com/workers/languages/python/) |
| 2 | **FastAPI is explicitly supported** | **[VERIFIED]** — *"Easy to install and fast-booting Packages, including FastAPI, Langchain, Pydantic and more."* | [Python Workers docs](https://developers.cloudflare.com/workers/languages/python/) |
| 3 | Packages bundled at deploy via `pywrangler` / `uv` | **[VERIFIED]** — *"To run a Python Worker locally, install packages, and deploy it to Cloudflare, you use pywrangler, the CLI for Python Workers."* | [Python Workers docs](https://developers.cloudflare.com/workers/languages/python/) |
| 4 | Package support = pure + PyEmscripten PyPI packages, plus what Pyodide ships | **[VERIFIED]** — *"Python Workers support pure and PyEmscripten Python packages on PyPI. Additionally, Python Workers support packages that are included in Pyodide."* | [Packages docs](https://developers.cloudflare.com/workers/languages/python/packages/) |
| 5 | **Python Workers ARE available on the free plan** | **[VERIFIED]** — *"For many use cases, Python Workers are completely free. Our free tier offers 100,000 requests per day and 10ms CPU time per invocation."* | [Python Workers redux](https://blog.cloudflare.com/python-workers-advancements/) |
| 6 | **Imports happen at deploy time, not request time** (memory snapshotting) | **[VERIFIED]** — *"When a Worker is deployed, we execute the Worker's top-level scope and then take a memory snapshot and store it alongside your Worker. Whenever we are starting a new isolate for the Worker, we restore the memory snapshot and the Worker is ready to handle requests, with no need to execute any Python code in preparation."* and *"we perform the expensive work of importing packages at deploy time, rather than at runtime."* | [redux](https://blog.cloudflare.com/python-workers-advancements/) · [Bringing Python to Workers](https://blog.cloudflare.com/python-workers/) |
| 7 | Snapshots cut cold start ~10s → ~1s for a FastAPI-class app | **[VERIFIED]** — *"starting a Worker that imports `fastapi`, `httpx` and `pydantic` without snapshots takes around 10 seconds. With snapshots, it takes 1 second."* | [redux](https://blog.cloudflare.com/python-workers-advancements/) |
| 8 | **Startup time is a SEPARATE limit from per-request CPU**: 1s to parse+execute global scope, on Free and Paid alike | **[VERIFIED]** — *"A Worker must parse and execute its global scope (top-level code outside of handlers) within 1 second."* Enforced at deploy; error `10021`. | [Workers limits](https://developers.cloudflare.com/workers/platform/limits/) |
| 9 | The startup limit was raised 400ms → 1s (2025-10-10) | **[VERIFIED]** — *"You can now upload a Worker that takes up 1 second to parse and execute its global scope. Previously, startup time was limited to 400 ms."* | [Changelog](https://developers.cloudflare.com/changelog/post/2025-10-10-increased-startup-time/) |
| 10 | **numpy is supported** | **[VERIFIED]** — *"FastAPI, Langchain, Numpy and more"*; redux: *"any package supported by Pyodide, including pure Python libraries and many that rely on dynamic C extensions (such as NumPy, Pandas, and Pillow)."* | [Bringing Python to Workers](https://blog.cloudflare.com/python-workers/) · [redux](https://blog.cloudflare.com/python-workers-advancements/) |
| 11 | Free per-request CPU is **10ms**; paid is 30s default, up to 5 min | **[VERIFIED]** (consistent with `ARCHITECTURE.md` §3) | [Workers limits](https://developers.cloudflare.com/workers/platform/limits/) |

### 2.2 Unverified — and honestly so

| # | Question | Finding |
|---|---|---|
| 1 | **Is scikit-learn available on Cloudflare Python Workers?** | **[UNVERIFIED].** Pyodide ships scikit-learn, and Cloudflare says it supports "packages that are included in Pyodide" (#4), which *implies* yes. But **Cloudflare's own package documentation never names scikit-learn** — it names FastAPI, Langchain, Pydantic, numpy, pandas, Pillow. Deriving sklearn support from a general statement is **inference, not verification**, and the docs warn: *"WebAssembly support for Python packages is still in early stages, and some packages may not yet be available as PyEmscripten wheels on PyPI."* **Do not assume sklearn runs in a Worker until someone deploys one.** |
| 2 | **Does snapshot restore / import cost count against the 10ms per-request CPU budget?** | **[UNVERIFIED] in the sense that matters.** The mechanism is documented (#6): imports are executed at *deploy* time and a snapshot is restored at isolate start, explicitly *"with no need to execute any Python code in preparation."* And global-scope execution is policed by the separate 1s **startup** limit (#8), not the per-request CPU limit. So the design intent is clear: **boot is not in your 10ms.** But Cloudflare **never states this explicitly for Python Workers**, and none of the pages that discuss snapshots discuss CPU accounting. Directionally favorable; not documented. |
| 3 | Snapshot cold start (~1s, #7) vs the 1s startup limit (#8) | **[UNVERIFIED] whether these are the same measure.** #7 is a wall-clock cold-start figure; #8 is a CPU limit on global-scope execution. They are **not** the same metric — but a FastAPI-class app measuring ~1s against a 1s-class ceiling is uncomfortably tight, and community reports of *"FastAPI Python Workers Exceed CPU Startup Limit"* exist (thread returned HTTP 403 to automated fetch; **unread, therefore uncited as evidence**). Flagging as a risk to measure, not a fact. |

### 2.3 The conclusion — and it doesn't depend on the unverified parts

**The free-plan question turned out not to be the crux.** Python Workers *are* free-tier eligible
(#5). The crux is what the free tier *is*:

> **The Python Workers free tier is 10ms CPU per invocation — the same ceiling the TypeScript Worker
> lives under (#5, #11).**

```
   sklearn training + 50k race generation   ≈   minutes of CPU
   Workers free tier                        =   10 ms of CPU
                                                ───────────────
                                                off by ~5 orders of magnitude
```

**Training cannot run in a Worker. Not a Python Worker, not a TypeScript Worker, not on the paid
plan** (whose 5-minute ceiling is still short for `ML_APPROACH.md` §8's 50k-race generation plus
fits plus cross-validation plus a GBM ceiling probe). Snapshotting doesn't help: it removes *import*
cost from request time, not *work* cost. sklearn's availability doesn't matter. The beta doesn't
matter. **The CPU arithmetic settles it on its own, using only verified numbers.**

This *confirms and sharpens* `ML_APPROACH.md` §5.2's table, which already lists "Python runtime in
Worker → Not on this architecture." The refinement: a Python runtime in a Worker **now exists and is
free**, and the conclusion is unchanged anyway — because the binding constraint was never the
language, it was the CPU.

> **Therefore: the training plane runs locally / in CI. This is not a compromise, and it does not
> break "all on Cloudflare free."**

### 2.4 Does the local training plane break "everything on Cloudflare free tier"? No.

This deserves a direct answer, because it looks like it should.

**The training plane is a build-time concern, not a hosting concern.** It is in the same category as
`wrangler deploy`, `tsc`, and `npm install` — developer-machine work that produces an artifact.
Nobody says a project "isn't really on Cloudflare" because its TypeScript was compiled on a laptop.

```
   ┌─────────────────────────────────────┐         ┌──────────────────────────────┐
   │  DEVELOPER MACHINE / CI             │         │  CLOUDFLARE FREE TIER        │
   │                                     │         │                              │
   │  tsc            ──► bundle.js ──────┼────────►│  Worker                      │
   │  wrangler                           │         │                              │
   │  FastAPI+sklearn ──► model.json ────┼────────►│  Worker (dot product)        │
   │                                     │         │                              │
   │  ── nobody hosts this. It's a build.│         │  ── this is the product.     │
   └─────────────────────────────────────┘         └──────────────────────────────┘
```

**Nothing in the training plane is deployed, and nothing at runtime calls it.** There is no server to
pay for, no uptime to maintain, no secret to manage, no network hop in any user's request. If the
training plane is switched off forever, **every race still runs and every prediction still works**,
because the weights are already in the bundle.

`PROBLEM_STATEMENT.md` §7 claims the constraints made the project better. This is another instance:
because the free tier can't host a training service, the model became **a JSON file in a bundle**
instead of **a service to operate** — which is simpler, faster, free, and has strictly fewer failure
modes. The constraint picked the right architecture.

### 2.5 Where the FastAPI service should actually run — the recommendation

| Option | Free-tier honest? | Can train? | Verdict |
|---|---|---|---|
| **A. Local / CI (uvicorn, container, GitHub Actions)** | ✅ Build-time, not hosted (§2.4) | ✅ Full CPython, full CPU, real sklearn | ✅ **RECOMMENDED** |
| **B. Python Worker on Cloudflare** | ✅ Free-tier eligible **[VERIFIED]** (#5) | ❌ **10ms CPU. Not remotely.** | ❌ Not for training. Possible later for a read-only registry API (§2.6). |
| **C. External host (Fly/Render/Railway/EC2)** | ❌ **Breaks it. Say so plainly.** | ✅ | ❌ Not now. Honest only if the training plane must be always-on and multi-user — which it mustn't (§2.4). |

**Recommendation: (A).** The FastAPI service runs on the developer's machine and in CI. It is real
software — typed, tested, containerized, `docker compose up` — it just isn't *hosted*, because
nothing needs it to be.

**On (C), stated plainly as asked:** if this project ever needs the training plane reachable over the
internet — a team retraining on demand, a UI that kicks off a fit — **then an external host is the
honest answer and "everything on Cloudflare free" becomes false.** That's a real trade, and the
moment to make it is when someone actually needs it. For a portfolio project whose primary user is
the person building it (`PROBLEM_STATEMENT.md` §2), **that moment does not exist**, and pretending
otherwise would buy a monthly bill and a second thing to operate in exchange for nothing.

### 2.6 The one place a Python Worker could earn a spot later

Not training — **a read-only model registry / metrics API**: serving `ML_APPROACH.md` §6.2's report
(baselines, AUC, calibration curve, confusion matrix, coefficient table) to the UI. That handler is
trivial work over precomputed JSON, which is exactly the shape the 10ms budget and snapshotting
suit.

**Recommend against it for v1 anyway**, for reasons that have nothing to do with Python:

1. It's a **second runtime and a second deploy pipeline** for a job the existing TS Worker + D1 + KV
   already do (`ARCHITECTURE.md` §2).
2. **`python_workers` is a beta flag** (#1). `ARCHITECTURE.md` §5 already sets the precedent — it
   declines to architect a dependency on Cloudflare's LoRA beta because betas are explicitly
   temporary. Same reasoning, same answer.
3. **sklearn-in-Worker is [UNVERIFIED]** (§2.2 #1), so a Python Worker couldn't do anything the TS
   Worker can't — the served artifact is a static report either way.

**Revisit if** a genuine Python-only serving need appears (e.g. a GBM that must run server-side per
`ML_APPROACH.md` §5.2). Note even then the export-trees-to-JSON path may beat it.

---

## 3. System context — the serving plane

Everything below this line is the serving plane. **The training plane appears exactly once more, as a
build-time arrow.**

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  BROWSER                                                          │
   │  race setup · replay viewer · scrubber · telemetry charts          │
   │  ── VIEW STATE ONLY. Never authoritative. ──                       │
   │  Replays a race locally from (seed + config + decision log).        │
   └────────────────────────────┬─────────────────────────────────────┘
                                │ HTTPS
   ┌────────────────────────────▼─────────────────────────────────────┐
   │  PAGES — static assets, replay viewer bundle                       │
   └────────────────────────────┬─────────────────────────────────────┘
                                │ fetch /api/*
   ┌────────────────────────────▼─────────────────────────────────────┐
   │  WORKER — API / orchestrator                    ┌───────────────┐ │
   │  Routes. Assembles personas at setup.           │  model.json   │ │
   │  Delegates. Runs the dot product. (ML §5.1)     │  ◄── BUILD    │ │
   │  ⚠ 10ms CPU ⇒ NEVER runs physics. ARCH §8.      │  TIME (§1)    │ │
   └──┬──────────────┬──────────────┬──────────────┬─└───────────────┘─┘
      │              │              │              │
      │ setup        │ per race     │ results      │ replay blobs
      │ (once)       │              │              │
 ┌────▼──────┐  ┌────▼─────────┐  ┌─▼──────┐  ┌────▼───┐
 │ VECTORIZE │  │  RaceDO      │  │   D1   │  │   R2   │
 │ spec +    │  │  1 per race  │  │results │  │ replay │
 │ route     │  │              │  │catalog │  │ blobs  │
 │ corpus    │  │  ┌────────┐  │  │leader- │  │(opt'l) │
 └───────────┘  │  │ 3 cars │  │  │ board  │  └────────┘
      │         │  │ 1 world│  │  └────────┘
      │ warm    │  └───┬────┘  │
      │ copy    │      │       │
 ┌────▼──────┐  │  ┌───▼─────┐ │       ┌──────────────────┐
 │    KV     │  │  │ hot     │ │       │   BudgetDO       │
 │ specs,    │◄─┼──┤ decision│ │       │  singleton       │
 │ route     │  │  │ cache   │ │       │  account-wide    │
 │ config    │  │  │(SQLite) │ │       │  neuron ledger   │
 │ read-only │  │  └─────────┘ │       │  §6, §10 — NEW   │
 │ in loop   │  └──────┬───────┘       └────────▲─────────┘
 └───────────┘         │                        │
                       │ env.AI.run()           │ reserve / commit
                       │ ~60 calls/race         │
                  ┌────▼────────────────────────┴─────┐
                  │  WORKERS AI                        │
                  │  driver agents · embeddings        │
                  │  ⚠ 10k neurons/day, ACCOUNT-WIDE   │
                  └────────────────────────────────────┘
```

**The load-bearing shapes:**

1. **The Worker never computes.** 10ms CPU. It routes, assembles, persists — and runs a dot product,
   which is the only "ML" the edge ever does (`ML_APPROACH.md` §5.1). (`ARCHITECTURE.md` §8)
2. **Vectorize is touched once per race, at setup.** Never in the loop. (`ARCHITECTURE.md` §2)
3. **KV is read-only in the hot path.** Its 1k writes/day makes it unusable as a cache.
   (`ARCHITECTURE.md` §4.1)
4. **The hot decision cache is inside the RaceDO, in SQLite.** Not KV. The most counterintuitive
   placement in the stack; `ARCHITECTURE.md` §4.1 is the argument.
5. **`model.json` enters at build time.** No arrow points from the Worker back to Python. Ever.
6. **BudgetDO is a singleton** — because neurons are account-wide, not per-race. **This is an
   addition to `ARCHITECTURE.md` §2's component map, not a restatement of it.** See §10.

> **[OPEN]** `ARCHITECTURE.md` §9.1 flags that the **DO CPU limit on the free plan is unverified**.
> The "physics in the DO" box depends on it. If DOs are capped at 10ms like plain Workers, the
> fallback is chunked ticking across alarms — the design survives, with more alarm hops. **Verify
> empirically.**

---

## 4. The centerpiece — how the three cars connect

This is the section the rest of the doc exists to support.

### 4.1 The shape

Three cars. Three **independent** driver agents. **One** shared physics world. **One** DO.

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  RaceDO — one per race, single-threaded, authoritative                         ║
║                                                                               ║
║   ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐         ║
║   │  DriverAgent      │  │  DriverAgent      │  │  DriverAgent      │         ║
║   │  "cybertruck"     │  │  "m3"             │  │  "waymo"          │         ║
║   ├───────────────────┤  ├───────────────────┤  ├───────────────────┤         ║
║   │ persona           │  │ persona           │  │ persona           │         ║
║   │  brash / high     │  │  precise / mid    │  │  conservative /   │         ║
║   │  risk, trusts     │  │  risk, trusts     │  │  low risk, obeys  │         ║
║   │  torque           │  │  late braking     │  │  everything       │         ║
║   │  ↳ system prompt  │  │  ↳ system prompt  │  │  ↳ system prompt  │         ║
║   │    PRECOMPUTED    │  │    PRECOMPUTED    │  │    PRECOMPUTED    │         ║
║   ├───────────────────┤  ├───────────────────┤  ├───────────────────┤         ║
║   │ spec (frozen)     │  │ spec (frozen)     │  │ spec (frozen)     │         ║
║   │  3084 kg          │  │  1855 kg          │  │  2140 kg          │         ║
║   │  Cd 0.335         │  │  Cd ~0.34 [MOCK]  │  │  Cd 0.29          │         ║
║   │  600 hp           │  │  503 hp           │  │  395 hp           │         ║
║   │  battery          │  │  fuel tank        │  │  90 kWh           │         ║
║   ├───────────────────┤  ├───────────────────┤  ├───────────────────┤         ║
║   │ policy (mutable)  │  │ policy (mutable)  │  │ policy (mutable)  │         ║
║   │  ← last LLM call  │  │  ← last LLM call  │  │  ← last LLM call  │         ║
║   │  lane, speed frac │  │  lane, speed frac │  │  lane, speed frac │         ║
║   │  hov, risk, branch│  │  hov, risk, branch│  │  hov, risk, branch│         ║
║   ├───────────────────┤  ├───────────────────┤  ├───────────────────┤         ║
║   │ hovEligibility    │  │ hovEligibility    │  │ hovEligibility    │         ║
║   │  occupancy 1..N   │  │  occupancy 1      │  │  occupancy 0      │         ║
║   │  ↳ RECOMPUTED per │  │  ↳ RECOMPUTED per │  │  ↳ AV_POLICY [OPEN]│        ║
║   │    (seg, time)    │  │    (seg, time)    │  │    decides this   │         ║
║   └─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘         ║
║             │                      │                      │                   ║
║             │   ── IDENTICAL INTERFACE, DIFFERENT DATA ──                     ║
║             │                      │                      │                   ║
║             └──────────┬───────────┴──────────┬───────────┘                   ║
║                        │                      │                               ║
║                    decide()              VehicleState                          ║
║                    ↓ Policy              ↑ (read)                              ║
║   ┌────────────────────────────────────────────────────────────────┐          ║
║   │  WORLD — ONE instance. The only place the cars meet.            │          ║
║   │                                                                │          ║
║   │   route: HighwaySegment[]        traffic field (PRNG)          │          ║
║   │   simClock                       lane occupancy grid            │          ║
║   │   vehicles: VehicleState[3]      enforcement rolls              │          ║
║   │                                                                │          ║
║   │   ┌──────────────────────────────────────────────────┐         │          ║
║   │   │  HOV CORRIDOR — 3 lanes, SHARED (SIM_RULES §1)    │         │          ║
║   │   │  ┌────┐ ┌────┐ ┌────┐   ← nine lane-slots do not │         │          ║
║   │   │  │ -1 │ │ -2 │ │ -3 │      exist. Three do.      │         │          ║
║   │   │  └────┘ └────┘ └────┘      They contend.         │         │          ║
║   │   └──────────────────────────────────────────────────┘         │          ║
║   │   ┌──────────────────────────────────────────────────┐         │          ║
║   │   │  GENERAL-PURPOSE LANES 0..n                       │         │          ║
║   │   └──────────────────────────────────────────────────┘         │          ║
║   └────────────────────────────────────────────────────────────────┘          ║
║                        │                                                       ║
║   ┌────────────────────▼───────────────────────────────────────────┐          ║
║   │  TICK LOOP — 10 Hz · pure · deterministic · NO LLM              │          ║
║   │  integrate(world, policies[3], prng) → world'                   │          ║
║   └────────────────────┬───────────────────────────────────────────┘          ║
║                        │                                                       ║
║   ┌────────────────────▼───────────────────────────────────────────┐          ║
║   │  DO SQLite — checkpoints · decision log · hot decision cache    │          ║
║   └────────────────────────────────────────────────────────────────┘          ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

### 4.2 The symmetry is the point

**All three cars implement the same interface.** There is no `if (vehicle.id === 'waymo')` anywhere
in the engine. A car is:

```
Car = Persona (a string) + Spec (numbers) + Policy (mutable state) + Eligibility (computed)
```

Every difference between the Cybertruck, the M3, and the Waymo is **data**, not code:

| Axis | Cybertruck | M3 | Waymo | Where it lives |
|---|---|---|---|---|
| Behavioral difference | brash persona | precise persona | conservative persona | **system prompt string** |
| Physical difference | 3084 kg, Cd 0.335 | 1855 kg, Cd ~0.34 | 2140 kg, Cd 0.29 | **spec record** |
| Structural difference | battery | fuel tank | battery | **energy spec (a variant)** |
| Rules difference | occupancy 1+ | occupancy 1 | occupancy 0 → `AV_POLICY` | **eligibility function input** |

**Why this matters beyond tidiness:** it's what makes a fourth car a config change, and it's what
makes the project's central experiment valid. `SIMULATION_RULES.md` §6.2 wants to run the same seed
under `av_hov_policy: 'strict'` vs `'transit_equivalent'` and attribute the delta to *the rule*. That
attribution is only sound if the rule is the **only** thing that differs. Special-casing a vehicle in
engine code would silently invalidate the experiment.

> **The Waymo is not a special vehicle. It is an ordinary vehicle with `occupancy: 0` and
> `is_autonomous: true`, hitting a rules function that has an [OPEN] question in it.**
> (`SIMULATION_RULES.md` §2.3)

### 4.3 Independent vs coupled — the exact line

```
   INDEPENDENT                     │  COUPLED (through World, never directly)
   ───────────────────────────────┼──────────────────────────────────────────
   ✓ Persona / system prompt       │  ✗ Position & lane occupancy
   ✓ Spec parameters               │  ✗ Traffic density encountered
   ✓ The LLM call itself           │  ✗ HOV lane contention (3 lanes, 3 cars)
   ✓ Policy selection              │  ✗ Following gaps / blocking
   ✓ Energy state                  │  ✗ Drafting, if modeled [OPEN]
   ✓ PRNG substream (see §4.4)     │  ✗ Merge yielding at ramps
                                   │  ✗ The HOV cliff at Webster (hits all 3)
```

**Agents never talk to each other.** No negotiation, no shared scratchpad, no agent-to-agent message
bus. Car A learns about Car B **exactly the way a real driver does: by looking at the world.** Rival
positions arrive in the observation passed to `decide()`, and only at segment boundaries.

This is a deliberate scope cap. A multi-agent negotiation layer would be a different (and much more
expensive) project, and per `ARCHITECTURE.md` §4 the neuron budget has no room for conversation on
top of decisions.

**Coupling is entirely mediated by `World`.** Three consequences:

1. **Lane contention is real.** Three cars, three HOV lanes. That's the *only* reason interpretation
   (A) in `SIMULATION_RULES.md` §1.1 makes a race. Under (B) — nine lanes — this column empties out
   and the three agents become three independent time trials.
2. **Coupling is read-then-write, never read-during-write.** Within a tick: all three cars observe
   the *same* frozen world snapshot, then all three integrate, then the world commits. Car 1 must not
   see Car 2's already-updated position — that would make results depend on iteration order,
   violating `SIMULATION_RULES.md` §6.1 requirement 4.
3. **Drafting is [OPEN].** If modeled, it's a coupling term in the drag calculation
   (`Cd_effective = Cd × f(gap_to_leader)`) — pure physics, in the World, no agent involvement. Not
   required for v1, and it interacts with the `d_min = 15 m` gap floor (`SIMULATION_RULES.md` §3.4)
   in ways nobody has thought through yet.

### 4.4 One seed, three cars — the PRNG substream trap

A determinism trap not covered in any sibling doc. Worth designing away now.

**Naive:** one global PRNG, cars draw from it in order. **This is broken.** Car 3's noise now depends
on how many draws cars 1 and 2 happened to make. Add a car, change a rule, reorder a loop — every
car's numbers move. The A/B experiment in `SIMULATION_RULES.md` §6.2 becomes unattributable, because
changing `av_hov_policy` shifts draw counts and therefore shifts *everyone's* traffic. **The rule
change would appear to alter cars it never touched.**

**Design: derive independent substreams from the one seed.**

```
prng_world   = PRNG(hash(seed, "world"))      // traffic field, enforcement rolls
prng_car[id] = PRNG(hash(seed, "car", id))    // per-vehicle noise
```

Now the seed contract from `SIMULATION_RULES.md` §6.1 holds, **and** a change to one car's rules
cannot perturb another car's draws. Same seed still ⇒ same race. One seed in, deterministic fan-out
to four streams.

**This also matters to `ML_APPROACH.md`.** §3.1 injects noise to generate the training set and §3.3
splits by seed group. Both assume a seed identifies a race cleanly. Order-dependent PRNG draws would
leak variance across cars *within* a seed group and quietly corrupt the split's independence
assumption. **The substream design protects the dataset, not just the sim.**

---

## 5. The decision loop — the crux

> **The single most important design decision in the project.** `ARCHITECTURE.md` §4 has the
> arithmetic that forces it; `SIMULATION_RULES.md` §6.3 has the determinism argument. This section
> shows the **sequencing**.

### 5.1 Two clocks, two frequencies

```
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║  PHYSICS                          ║  LANGUAGE                            ║
  ║  10 Hz — every 100 ms sim-time    ║  ~20× per car per race               ║
  ║  ~6,000 ticks per 10-min race     ║  ~60 calls per 3-car race            ║
  ║  Deterministic. Pure. Local.      ║  Non-deterministic. Impure. Network. ║
  ║  NO network. NO LLM. NO clock.    ║  Ruk 8,412 exactly — in the browser, for free.

> **Determinism is a storage strategy.** A whole race is a seed, a config, and ~60 policies —
> kilobytes, not gigabytes. `SIMULATION_RULES.md` §6.2 calls replay a hosting decision, not just a
> code virtue.

### 6.3 The seed-replay property

> **`same seed + same config + same engineVersion + same policy log ⇒ bit-identical result`**

Note the fourth term. `SIMULATION_RULES.md` §6.1 states the contract for the *engine*; the system
adds this: **replay does not call the LLM.** It replays the *recorded* policies — which is what makes
the property hold at all, since the LLM is non-deterministic even at `temperature: 0`.

```
   RACE (once):     seed + config + LLM ──► policies[60] ──► result
                                    ↑
                                    └── non-deterministic, network, costs neurons

   REPLAY (∞):      seed + config + policies[60] ─────────► result   (identical)
                                    ↑
                                    └── deterministic, local, free
```

**The LLM is a compile-time input to the race, not a runtime dependency** (`SIMULATION_RULES.md`
§6.3). A race recorded today replays in five years on a model that no longer exists.

---

## 7. State ownership

One rule: **every piece of state has exactly one owner, and the owner is chosen by write pattern, not
by convenience.**

| State | Owner | Plane | Write pattern | Why here |
|---|---|---|---|---|
| Authoritative race state (positions, speeds, sim clock, energy) | **RaceDO (memory + SQLite checkpoints)** | Serving | Very hot, single-writer | Single-threaded ⇒ no locking. Only place suspend-await-resume (§5.3) is safe. |
| **Hot LLM decision cache** | **RaceDO SQLite** | Serving | Write-heavy on miss (~1k/day at 1B scale) | **100k SQL writes/day vs KV's 1k.** `ARCHITECTURE.md` §4.1 — read it before moving this. |
| Decision log (~60/race) | **RaceDO SQLite** → batched to D1 at finish | Serving | Append-only, then immutable | Written next to the state that produced it; flushed once. |
| Checkpoints (~20/race) | **RaceDO SQLite** | Serving | ~20 writes/race | Durability across eviction (§8.3). Boundaries, not ticks. |
| Neuron ledger | **BudgetDO (singleton)** | Serving | Hot, contended, account-wide | Neurons are an **account** resource. One serialization point. §10. |
| Race results, leaderboards, catalog | **D1** | Serving | Once per race, then read-many | Relational, queryable, exact-match. |
| Vehicle specs, route/segment config, persona templates | **KV** | Serving | Written once at deploy, read constantly | Exactly KV's shape: 100k reads/day, read-mostly. |
| Spec + route corpus (embedded) | **Vectorize** | Serving | Indexed once (~184 neurons, `ARCH` §7) | Semantic retrieval at setup only. Never in the loop. |
| Replay blobs *(optional)* | **R2** | Serving | Immutable, large, rare | 10 GB, free egress. Only if replays outgrow a D1 row — they probably won't (§6.2). |
| **Model weights (`model.json`)** | **The repo / the bundle** | **Boundary** | Written by training, read at edge | **A build artifact, not a database.** Versioned by git, shipped in the bundle. No runtime fetch, no hop, no failure mode. |
| **Training datasets (`races.parquet`)** | **Training plane, local/CI** | **Training** | Large, regenerable | Never leaves the training plane. **Regenerable from seeds** — so it needn't be durable at all (§7.2). |
| **Model registry, eval reports, joblib artifacts** | **Training plane, local/CI** | **Training** | Rare | `ML_APPROACH.md` §5.3: joblib stays local; only JSON crosses. |
| View state (camera, scrubber, chart zoom) | **Browser** | Serving | Ephemeral | **Never authoritative.** A refresh loses nothing that matters. |

### 7.1 The two placements that look wrong and aren't

**1. The cache is in a database, not a cache.** DO SQLite isn't what anyone reaches for when they want
a cache — KV is. But KV's free tier is 100k reads/day against **1,000 writes/day**, a 100:1 ratio,
and a decision cache writes on every miss. `ARCHITECTURE.md` §4.1 works the arithmetic and finds the
trap: switching to Llama 3.2 1B (the mitigation that buys ~7× more races/day) multiplies cache writes
by the same factor and lands flush against KV's ceiling. **The fix for the neuron budget is the thing
that breaks the KV budget.** DO SQLite has 100× the write headroom and already owns the state. Also:
KV throttles at 1 write/sec to the same key — a hot key under concurrent races (§10) would serialize.

**2. The cache is per-race, not global.** Two concurrent races can't share hits, and a race's cache
dies with it. Accepted deliberately: a **global** cache must live somewhere shared — KV
(write-limited) or a cache DO (a single-threaded bottleneck on every decision across every race).
Cross-race hit rates are speculative anyway, since the key includes `persona_id` and segment
archetype. **Intra-race repetition is where the hits are** — `ARCHITECTURE.md` §4 mitigation #5 says
situations repeat heavily *within* a race. Revisit only with measured cross-race data. AI Gateway
(mitigation #6) already dedupes identical prompts globally, for free, covering part of this.

### 7.2 Datasets are regenerable, so don't back them up

`races.parquet` may be hundreds of MB. It is also **fully reconstructible from a list of seeds plus
an engine version** (`SIMULATION_RULES.md` §6.1) — which is a few KB.

**Store the seeds and the engine version in the dataset manifest; treat the parquet as a cache.** If
it's lost, regenerate it. This is the same insight as §6.2 (store the seed, not the telemetry),
applied one plane over — and it's another dividend of the determinism contract that isn't obvious
until you go looking for it.

**One caveat:** the manifest must pin `engineVersion`. Determinism is a promise *within* a version
(`SIMULATION_RULES.md` §6.1 req. 6), so a dataset regenerated under a bumped engine is **a different
dataset**, and a model trained on the old one is silently mismatched to the new sim. **Pin it, assert
it at train time, and record it in `model.json`** (§8.7).

---

## 8. Interface sketches

**Illustrative only — this is a design doc, not an implementation.** Core world types (`Vehicle`,
`HighwaySegment`, `RaceConfig`, `RaceResult`, `TickTelemetry`, `DriverPolicy`, `HovEligibility`) are
specified in `SIMULATION_RULES.md` §5.1 and §6.3 and are **not** repeated. These are the *system*
seams.

```ts
/** THE central abstraction. All three cars implement exactly this.
 *  No vehicle-specific branch exists anywhere in the engine. (§4.2) */
interface DriverAgent {
  readonly id: VehicleId;                 // 'cybertruck' | 'm3' | 'waymo'
  readonly persona: PersonaProfile;       // precomputed at setup, frozen (§6.1)
  readonly spec: Vehicle;                 // SIMULATION_RULES §5.1 + VEHICLE_SPECS.md
  policy: DriverPolicy;                   // mutable: replaced at each boundary (§5.6)

  /** Called ONLY at a SegmentBoundary. Never from the tick loop. (§5)
   *  Impure: may hit cache, budget, network. Always returns a policy — never throws. (§9) */
  decide(obs: Observation): Promise<PolicyDecision>;
}

interface PersonaProfile {
  id: string;                    // 'brash' | 'precise' | 'conservative' — [MOCK], all of it
  systemPrompt: string;          // ASSEMBLED ONCE at setup. ~800 tokens. (§6.1)
  riskProfile: number;           // 0..1 — also drives the deterministic fallback table (§9.1)
  fallbackPolicies: Record<SegmentArchetype, DriverPolicy>;  // ← the degradation path
}

/** What an agent may see. Deliberately narrow.
 *  Note what's ABSENT: no rival personas, no rival policies, no future traffic,
 *  no other agent's reasoning. Cars observe the world, not each other's minds. (§4.3) */
interface Observation {
  self: VehicleState;
  rivals: PublicVehicleState[];         // position, lane, speed — what a driver could SEE
  segment: HighwaySegment;              // current
  upcoming: HighwaySegment;             // next — the thing being decided about
  hovEligibility: HovEligibility;       // COMPUTED per (vehicle, segment, time) — SIM_RULES §2.3
  simTime: SimTime;                     // sim clock. NEVER Date.now(). SIM_RULES §6.1
  branchOptions?: string[];             // e.g. ['i35', 'tx130'] — SIM_RULES §4.1
}

interface VehicleState {               // authoritative, owned by World (§7)
  vehicleId: VehicleId;
  position_m: number;                  // offset along route
  lane: number;                        // negative = HOV lanes
  speed_mps: number;
  energy_remaining: number;
  segmentIndex: number;
  lastLaneChange_s: number;            // enforces the 4s cooldown — SIM_RULES §3.4
  penalties_s: number;
  hovDistance_m: number;               // accumulator: how much of the win was lane access
}

/** The ONE instance the three cars share. This is the coupling surface. (§4.1) */
interface WorldState {
  simTick: number;
  simTime_s: number;
  route: HighwaySegment[];
  vehicles: VehicleState[];            // length 3 — STABLE ORDER, SIM_RULES §6.1 req.4
  laneOccupancy: LaneGrid;             // the contention structure
  prngWorld: PRNG;                     // substream — §4.4
  prngCar: Record<VehicleId, PRNG>;    // substreams — §4.4
}

type SegmentArchetype =                // what the cache key and fallback table index on (§5.5)
  | 'open_highway' | 'congestion' | 'hov_entry' | 'hov_cliff'
  | 'branch_point' | 'bridge_funnel' | 'limit_step_down';

interface SegmentBoundary {
  vehicleId: VehicleId;                // boundaries are PER-VEHICLE. They stagger. (§5.2)
  atTick: number;
  fromSegment: string;
  toSegment: string;
  archetype: SegmentArchetype;
  trigger: 'segment_boundary' | 'rival_within_threshold' | 'energy_low' | 'hov_ending';
}

/** Wraps the policy with its provenance. Provenance isn't decoration —
 *  it's how a degraded race labels itself honestly. (§9.6) */
interface PolicyDecision {
  policy: DriverPolicy;                // SIM_RULES §6.3
  source: 'llm' | 'cache' | 'fallback_budget' | 'fallback_malformed' | 'fallback_timeout';
  neuronsSpent: number;                // 0 for cache and every fallback
  boundary: SegmentBoundary;
}

interface RaceDO {
  init(config: RaceConfig): Promise<void>;
  start(): Promise<void>;              // schedules the first alarm
  alarm(): Promise<void>;              // ticks a chunk; MUST be idempotent (§9.3)
  status(): Promise<RaceStatus>;       // polled by the browser during a live race
  result(): Promise<RaceResult>;       // once finished; carries MOCK_DATA: true
}

interface BudgetDO {                   // singleton — the account-wide ledger (§10)
  reserve(n: number): Promise<{ granted: boolean; remaining: number }>;
  commit(actual: number): Promise<void>;   // reconcile reservation vs actual spend
  release(n: number): Promise<void>;       // unused reservation returns to the pool
}

/** ═══ THE PLANE BOUNDARY (§1) ═══
 *  The ONLY artifact crossing from training to serving. A file, not a service. */
interface ExportedModel {
  schema_version: string;
  model_family: 'logistic' | 'linear';   // GBM does NOT export here — ML_APPROACH §5.2
  feature_names: string[];               // ORDERED. Asserted at the edge. ML_APPROACH §5.3
  coef: number[];
  intercept: number;
  scaler: { mean: number[]; scale: number[] };   // StandardScaler — ML_APPROACH §5.1
  engine_version: string;                // which sim generated the training data (§7.2)
  trained_at: string;
  dataset_manifest_hash: string;         // provenance: which seeds produced this
  metrics: { test_accuracy: number; test_auc: number; baseline_accuracy: number };
                                         // baseline travels WITH the model — ML_APPROACH §6.1
  MOCK_DATA: true;                       // synthetic benchmark. ML_APPROACH §1.3
}
```

**Two things to notice.**

`DriverAgent` has no vehicle-specific method. Swap the `PersonaProfile` and the `Vehicle` spec and
you have a different car. That is the entire mechanism by which three cars differ. (§4.2)

`ExportedModel` carries `baseline_accuracy` **alongside** `test_accuracy`. `ML_APPROACH.md` §6.1 is
emphatic that a raw accuracy without its baseline is misleading "including to yourself." Shipping
them in the same object makes it **structurally awkward to display one without the other** — the same
trick as `MOCK_DATA: true` (`SIMULATION_RULES.md` §5.1). Honesty enforced by the type, not by
remembering.

---

## 9. Failure modes and degradation

> **The invariant: every failure degrades to something deterministic. No failure produces a broken
> race, a stuck race, or a race that quietly lies about what it is.**

A race that finishes on fallback policies is a valid race with a *labeled* decision source. A race
that hangs at segment 12 because Workers AI returned a 429 is a bug.

| # | Failure | Detection | Degradation | Race finishes? | Labeled? |
|---|---|---|---|---|---|
| 1 | Neuron budget exhausted mid-race | `BudgetDO.reserve()` denied | Persona default policy table | ✅ | `degraded: true` |
| 2 | LLM returns malformed/unparseable JSON | Schema validation fails | Clamp if salvageable, else fallback | ✅ | `source: 'fallback_malformed'` |
| 3 | LLM times out / 5xx / 429 | Await rejects or exceeds deadline | Fallback immediately, no retry storm | ✅ | `source: 'fallback_timeout'` |
| 4 | DO evicted / restarted mid-race | Alarm re-fires on cold object | Resume from last checkpoint, re-tick | ✅ | invisible — result identical |
| 5 | Vectorize unavailable at setup | Query throws | KV warm copy of specs | ✅ | logged, result unaffected |
| 6 | Cold cache day (0% hit rate) | — | Nothing. Budget already assumes it. | ✅ | n/a |
| 7 | DO CPU ceiling lower than hoped (`ARCH` §9.1) | Load test | Smaller tick chunks per alarm | ✅ | slower wall-clock only |
| 8 | D1 write fails at finish | Write rejects | Result stays in DO SQLite; retry | ✅ | race not lost |
| 9 | **`model.json` missing / stale / schema-mismatched** | Schema + feature-order assert at boot | **Hide the prediction. Race is unaffected.** | ✅ | prediction absent, not wrong |
| 10 | **Training plane unavailable** | — | **Nothing. It's build-time.** (§2.4) | ✅ | n/a |

### 9.1 Neuron exhaustion — the one that actually matters

10,000 neurons/day, **account-wide** (§10). This *will* happen — it's the binding constraint on the
project (`ARCHITECTURE.md` §3).

**Two layers of defense.**

**Layer 1 — admission control.** Don't start a race you can't finish. At setup, `BudgetDO.reserve()`
the *whole* race (3 × ~20 × neurons/decision). Denied ⇒ the race never starts, and the user gets an
honest "daily AI budget exhausted — try a deterministic race, or come back tomorrow." **Refusing to
start is a better failure than dying at segment 14.**

**Layer 2 — the persona default policy table.** Reservation isn't a guarantee: estimates can be
wrong, trigger events can fire extra decisions, concurrent races contend (§10). If the budget runs
dry mid-race anyway:

```
   fallbackPolicies: Record<SegmentArchetype, DriverPolicy>
   ────────────────────────────────────────────────────────
   'brash'        × 'congestion'   → { targetSpeedFraction: 1.0, useHovLane: true,
                                       followingAggressiveness: 0.9 }
   'conservative' × 'congestion'   → { targetSpeedFraction: 0.85, useHovLane: true,
                                       followingAggressiveness: 0.3 }
   'precise'      × 'branch_point' → { takeBranch: 'tx130', ... }
   ...
```

A static table, authored once per persona, indexed by `(persona, segment_archetype)`. **[MOCK]** —
hand-authored constants, like every tuning value in `SIMULATION_RULES.md` §3.4.

Three properties make this the right fallback:

1. **It's still in character.** A degraded Cybertruck drives brash; a degraded Waymo drives
   conservative. The race stays a contest between three *distinguishable* cars, because persona was
   never only in the LLM — it's in the risk profile too.
2. **It's a pure function.** Zero neurons, zero network, zero latency.
3. **It makes the race *more* deterministic, not less.** A fully-degraded race is reproducible
   without a decision log at all — `(seed, config)` alone regenerates it, because the fallback table
   is part of the config. Strictly stronger than a normal race.

**What's lost:** situational judgment. The fallback can't notice the M3 is three car-lengths back and
closing. It plays the archetype, not the moment. **That's exactly the value the LLM adds, and a
degraded race is the measurement of it.** Which suggests a real experiment: run the same seed
fully-LLM and fully-degraded, and diff. If the finish order doesn't change, the LLM layer isn't
earning its neurons — a finding worth having, not hiding from.

**And property 3 is why the training plane gets its data for free** (§6): 50k races at zero neurons,
because fully-degraded generation is exactly what `ML_APPROACH.md` §8 step 2 needs. **The failure
mode is also the bulk-generation mode.** That is not a coincidence — both want "deterministic physics,
no network," and the fallback table is the only thing that provides it.

### 9.2 Malformed LLM output

Free-tier models on ~64-100 `max_tokens` will produce truncated JSON, fenced JSON, prose-wrapped
JSON, and confident nonsense. Assume it.

**Ladder, cheapest rung first:**

```
   1. JSON mode          env.AI.run(model, { messages, response_format })  (ARCH §6)
                         ↓ still malformed?
   2. Lenient parse      strip fences, extract first {...}
                         ↓ still unparseable?
   3. Schema validate    unknown fields dropped; missing → persona defaults
                         ↓ present but out of range?
   4. CLAMP              targetSpeedFraction → [0,1]; targetLane → legal lanes for this
                         segment; useHovLane ∧ ¬eligible → violation risk path (SIM_RULES §2.3)
                         ↓ nothing salvageable?
   5. FALLBACK           persona default. source: 'fallback_malformed'.
```

**Clamp, don't retry.** A retry costs a full second decision's neurons — the most expensive possible
response to the cheapest possible problem, and it doubles the worst case on exactly the day the
budget is tight. At most **one** retry, only for a total parse failure at a `branch_point` (genuinely
high-stakes). Everywhere else: clamp or fall back.

**Clamping is safe because the action space is bounded at construction** (`SIMULATION_RULES.md`
§3.1). No value of `targetSpeedFraction` breaks the speed cap, because the field is a fraction *of*
the cap. **A hostile model cannot produce an illegal race.** The worst it can do is drive badly.

### 9.3 DO eviction mid-race — is state durable?

**In-memory world state is not durable. DO SQLite is.** A DO can be evicted between alarms; the alarm
re-fires on a cold object with empty memory.

**Design: checkpoint at segment boundaries, resume by re-ticking.**

```
   ┌── checkpoint at S1 ──┐  ~300 ticks (in memory only)  ┌── checkpoint at S2 ──┐
   │  world snapshot      │  ░░░░░░░░░░░░░░░░░░░░░░░░░░░  │  world snapshot      │
   │  + policies[3]       │           ▲                   │  + policies[3]       │
   └──────────────────────┘           │                   └──────────────────────┘
                                   EVICTED here
                                      │
                                      ▼
   restart ─► load checkpoint S1 ─► re-tick deterministically ─► identical state
              (~20 SQLite reads)     (recorded policies, same PRNG substreams)
```

**Determinism gives us crash recovery for free.** Because `(checkpoint, policies, seed) ⇒ the next
300 ticks, exactly`, re-ticking isn't approximate recovery — it reproduces the lost work bit-for-bit.
Nothing is estimated, nothing drifts.

**Checkpoint at boundaries, not ticks.** ~20 writes/race against 100k SQL writes/day
(`ARCHITECTURE.md` §3) is free. Per-tick checkpointing would be 6,000 writes/race — 16 races/day
against that limit, and pointless, since re-ticking is cheap and exact.

**The alarm handler must be idempotent.** Cloudflare retries failed alarms. Resume-from-checkpoint
gives idempotency by construction: re-running a chunk produces the same state and appends nothing new
(decision log keys on `(vehicleId, boundaryTick)` — an upsert, not an append).

**Cost:** at most one segment's ticks are recomputed. Nothing user-visible.

### 9.4 Vectorize unavailable at setup

**Fallback: KV warm copy.** The route/spec corpus is written to KV at deploy time — written once,
read constantly, precisely KV's shape (`ARCHITECTURE.md` §4.1) and nowhere near its 1k write/day
limit.

**Why this fallback is complete rather than partial:** for **three known vehicles on four known
routes**, semantic retrieval isn't load-bearing. Vectorize earns its place as the corpus grows —
narrative grounding, more vehicles, `VEHICLE_SPECS.md` provenance snippets, spec Q&A. But the
race-critical path needs `spec[vehicleId]` and `route[routeId]`: an **exact-match lookup on a key we
already have.** KV does that better than Vectorize.

Stated plainly: **Vectorize is not on the critical path for a race between three cars we've already
enumerated.** If it's down, races run at full fidelity from KV. Log it, don't fail.

### 9.5 Cold cache day

**Nothing happens, by design.** §11's budget table assumes 0% hit rate. Cache hits are headroom, not
capacity. `ARCHITECTURE.md` §4.1 flags the cold-start day as the scenario that breaks a
cache-dependent budget — so we don't build one.

### 9.6 Degradation must be visible

Every fallback is recorded in `PolicyDecision.source` (§8), aggregated onto the result, and shown in
the UI:

```ts
interface RaceResult {
  // ... SIMULATION_RULES §5.1
  degraded: boolean;                   // any decision came from a fallback
  decisionSources: Record<PolicyDecision['source'], number>;
                                       // e.g. { llm: 42, cache: 15, fallback_budget: 3 }
  MOCK_DATA: true;                     // SIMULATION_RULES §5.1 — literal, non-optional
}
```

This follows the precedent `SIMULATION_RULES.md` §5.1 sets with `MOCK_DATA: true` and
`speed_limit_verified: false`: **a result that escapes into a screenshot should carry its own
caveats.** A race where the Cybertruck won on fallback policies is a different claim than one where
it won on 60 live LLM decisions, and the result object should never let those be confused.

### 9.7 The model boundary fails safe

The plane boundary (§1) has exactly one failure mode, and it's mild.

```
   Worker boot ─► load model.json ─► assert schema_version
                                  ─► assert feature_names order   ← ML_APPROACH §5.3:
                                  ─► assert engine_version match     "the #1 way exported
                                                                      models break"
        any assert fails
              │
              ▼
   ┌──────────────────────────────────────────────────────┐
   │  Serve WITHOUT the prediction.                        │
   │  The race is completely unaffected — the model was    │
   │  never an input to the physics (ML_APPROACH §7).      │
   │  UI simply omits the pre-race prediction card.        │
   └──────────────────────────────────────────────────────┘
```

**Assert loudly, degrade quietly, never guess.** A missing prediction is a small hole in the UI. A
*silently wrong* prediction from a reordered feature vector is the failure `ML_APPROACH.md` §5.3
warns about — plausible, wrong, and undetectable. **Given the choice, show nothing.**

The `engine_version` assert is the subtle one (§7.2): a model trained on data from engine v1.2 makes
statements about a simulator that no longer exists once physics changes. It isn't wrong so much as
**about a different world** — so bump, retrain, or hide.

**Note the blast radius.** The learned model is a **read-only decoration on the setup screen**
(`ML_APPROACH.md` §7: two separate systems, don't let them merge). It never touches physics, never
feeds an agent, never changes a result. **That's why the whole training plane can be missing (#10)
and every race still runs.** Loose coupling wasn't an accident here — it fell out of the plane split.

---

## 10. Concurrency — many races, one budget

**Multiple simultaneous races = multiple RaceDOs.** That scales cleanly: DOs are isolated by ID, each
single-threaded, each with its own state, cache, and checkpoints. Two races never touch.

**The daily budgets do not scale. They're ACCOUNT-WIDE.**

```
        race A          race B          race C          ← isolated. No contention.
       ┌───────┐       ┌───────┐       ┌───────┐
       │RaceDO │       │RaceDO │       │RaceDO │
       │ 3 cars│       │ 3 cars│       │ 3 cars│
       └───┬───┘       └───┬───┘       └───┬───┘
           │               │               │
           └───────────────┼───────────────┘
                           ▼
              ╔════════════════════════════╗
              ║   ONE 10k neuron/day pool  ║   ← CONTENDED. Account-wide.
              ║   ONE 1k KV write/day pool ║   ← CONTENDED. Account-wide.
              ║   ONE 100k D1 write pool   ║
              ╚════════════════════════════╝
                           ▲
              ┌────────────┴────────────┐
              │  BudgetDO (singleton)   │  ← the serialization point
              │  reserve / commit /     │     NEW — not in ARCH §2's map
              │  release                │
              └─────────────────────────┘
```

**This is the real design consideration, and it's easy to miss.** Three concurrent races are not "3×
the work" — they're **three claimants on one pool sized for ~5 races/day total** (§11, 8B model).
Race C can be starved by races A and B, mid-flight, through no fault of its own. **Isolation of
*state* does not give isolation of *budget*.**

**Design responses:**

1. **BudgetDO is a singleton.** Every race reserves against one DO. Its single-threaded model is
   exactly the needed property: reservations serialize, no lost-update race on the counter. A correct
   distributed counter, for free.
2. **Reserve whole races, not individual decisions.** Reserving up front (§9.1) makes starvation
   deterministic and *early*: race C is refused at setup, when refusing is cheap and honest, rather
   than degrading at segment 14. **Fail at admission, not in flight.**
3. **Release unused reservations.** A race finishing under budget (cache hits, fewer triggers than
   estimated) returns the difference. Without `release()`, reservations are a slow leak that makes the
   pool look emptier than it is and starves later races for nothing.
4. **Cap concurrent races.** A hard, low limit (start with **2**), derived from §11's races/day. It's
   a portfolio project (`PROBLEM_STATEMENT.md` §2) — the honest concurrency requirement is roughly
   one. Queue or refuse beyond it.
5. **Replay is unlimited and uncontended.** The pressure valve, and why §6.3 matters more than it
   first appears: replays consume **zero neurons and zero server CPU**. The expensive path is running
   a *new* race. Sharing, scrubbing, re-watching, embedding a result in a portfolio page — all free,
   all infinitely concurrent. **A link to a seed is a race anyone can watch without spending
   anything.**

**BudgetDO's own limits:** one DO taking a handful of requests per race — trivially inside 100k DO
req/day. But it *is* a single-threaded global bottleneck, so it must stay small: a counter and a date
check. **No inference behind that lock, ever.**

> **[OPEN]** Neuron accounting is *our* estimate (`ARCHITECTURE.md` §4's per-decision figures), not a
> number Cloudflare returns per call. BudgetDO tracks what we *think* we spent. Reconcile against
> Workers AI analytics (via AI Gateway, `ARCHITECTURE.md` §4 mitigation #6) before trusting the
> ledger's absolute accuracy. Keep a margin — reserve at ~1.3× estimate.

---

## 11. The budget, for a three-car field

`ARCHITECTURE.md` §4's headline — **~3 races/day on an 8B model** — is computed for a **6-car** field
(6 × 20 = 120 decisions). The owner's ask is **3 cars**. Same per-decision figures from
`ARCHITECTURE.md` §4, halved decision count:

| Model | Neurons/decision (`ARCH` §4) | 3-car race (60 decisions) | Races/day @ 10k neurons |
|---|---|---|---|
| Llama 3.2 1B | ~4.2 | ~252 | **~39** |
| Llama 3.1 8B | ~29.5 | ~1,770 | **~5.6** |
| Llama 3.3 70B | ~46 | ~2,760 | ~3.6 |

**Derived from `ARCHITECTURE.md` §4's per-decision figures — not independently measured.** Two honest
caveats:

1. **Budget for a 0% cache hit rate.** The table assumes *no* cache. Treat every hit as headroom,
   never as capacity already spent. A design that only fits *with* a 60% hit rate fails on its first
   cold-start day — exactly the scenario `ARCHITECTURE.md` §4.1 warns about. **If it fits cold, it
   always fits.**
2. **The 1B row re-opens the KV trap at 3 cars too.** ~39 races/day × 60 decisions = ~2,340
   decisions/day. At a 60% hit rate that's ~936 cache writes — flush against KV's 1,000/day. This is
   `ARCHITECTURE.md` §4.1's arithmetic reproducing itself at a different field size, which is a good
   sign the conclusion (cache in DO SQLite) is **structural**, not a coincidence of one configuration.

**Recommendation:** Llama 3.2 1B for routine boundaries; escalate to 8B only for high-stakes
archetypes (`branch_point` — the TX-130 decision; `hov_cliff` — Webster). Per `ARCHITECTURE.md` §4
mitigation #4. A mixed field of ~55 cheap + ~5 expensive decisions lands near ~380 neurons/race →
**~25 races/day**, which is comfortable.

**And note what costs nothing:** replays (§10.5) and training-set generation (§9.1). **The only
expensive thing in this system is a new race with live agents.** Everything else is deterministic
arithmetic, and deterministic arithmetic is free.

---

## 12. What this design does not decide

Carried forward, not resolved here. **These belong to their owning docs — this section records only
what the system design is blocked on or sensitive to.**

| # | Question | Owner doc | Impact on this design |
|---|---|---|---|
| 1 | **HOV interpretation (A) — one shared 3-lane corridor?** | `SIM_RULES` §1.1 | **Structural.** Under (B), §4.3's coupling column empties and the three cars stop being a system. |
| 2 | **`AV_POLICY`** — can a zero-occupant Waymo use HOV? | `SIM_RULES` §2.3 | None structurally — it's config, by design (§4.2). Changes *outcomes*, not *architecture*. That it's a knob and not a branch is the point. |
| 3 | **DO CPU ceiling on the free plan** | `ARCH` §9.1 | **Load-bearing.** If DOs get 10ms, §5.3's suspend-resume needs more alarm hops. Design survives; wall-clock suffers. **Verify empirically.** |
| 4 | **scikit-learn on Python Workers** | this doc §2.2 | **None — and that's the point.** Training never runs in a Worker (§2.3), so the answer doesn't matter. Only revisit if §2.6 is ever built. |
| 5 | **Snapshot restore vs the 10ms CPU budget** | this doc §2.2 | None for v1 (no Python Worker deployed). Blocking *only* for §2.6. |
| 6 | Cache bucket granularity | this doc §5.5 | Tuning dial: hit rate vs tactical resolution. Needs measurement. |
| 7 | Drafting as a coupling term | this doc §4.3 | Additive. Not v1. |
| 8 | Cross-race cache sharing | this doc §7.1 | Deferred pending measured cross-race hit rates. |
| 9 | Trigger events beyond segment boundaries | `ARCH` §4 | Each trigger type raises decisions/race above ~20 and re-prices §11. Budget before adding. |
| 10 | Model escalation policy (1B vs 8B per archetype) | this doc §11 | Recommendation only. Needs a real quality comparison. |

---

## 13. Reading this as a new contributor

The six things to take away:

1. **Two planes, one artifact.** Training is Python, offline, unlimited CPU. Serving is TypeScript,
   edge, 10ms. The only thing crossing is `model.json`. If you find yourself wanting the Worker to
   call Python, you've merged the planes — stop. (§1)
2. **The LLM is not in the loop.** It runs ~20×/car at segment boundaries and returns a *policy*.
   Physics runs 6,000×/race and never touches a network. If you're adding an `await` inside the tick
   loop, stop. (§5)
3. **Three cars, one world.** Identical interface; different persona + spec + policy + eligibility. No
   vehicle-specific code paths. Ever. (§4)
4. **The seed is the race.** Don't store telemetry — store the seed and the ~60 policies. Replay
   regenerates everything, in the browser, for free. (§6.2, §6.3)
5. **Every failure ends in a policy.** Budget gone, JSON broken, DO evicted — the race finishes,
   deterministically, and says so in `degraded`. (§9)
6. **The budget is account-wide.** Concurrent races contend for one 10k-neuron pool. Reserve up
   front, fail at admission, release what you don't use. (§10)

---

*SimZoner produces synthetic results from a mock world. It is not a prediction of real vehicle
performance and must never be presented as one. See `SIMULATION_RULES.md` §0.*
