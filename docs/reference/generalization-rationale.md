# Design Rationale — "Generalize This Whole Effort"

**What this is.** The narrative record of how the `blueprints` repo and the
`simple-hardware-controller-service` blueprint were designed — the reasoning, the forks, the
owner's corrections, and the *why* behind each decision. The [spec](../../blueprints/simple-hardware-controller-service/spec.md)
is the *what*; the [decision log](../reference/decision-log.md) is the terse *what + status*;
**this doc is the *why and how-we-got-here*.** It exists so a future reader (or a future you)
understands the *reasoning*, not just the outcome — because an unrecorded rationale gets re-derived,
usually wrongly.

**Status.** Owner-directed into the approved tree (`docs/reference/`). It is a rationale record, not
a spec — it does not define contracts; it explains them.

---

## 1. The seed idea

The prompt: LiteController (a lighting controller) is **one instance of a recurring product shape** —
a headless service that owns some hardware, exposes it to other surfaces, and is honest about state.
Many more like it will follow. They should all *look/act/feel the same*; the only real differences
are **the target hardware and its specific commands.** The goal: **factor out the infrastructure so
building the next one is "focus on the hardware specifics," not "re-build the whole daemon."**

This is a generalization effort, and the discipline throughout was: **extract from what we have, but
never over-abstract from a single instance.** (The Device-Model Doctrine's own warning — "one is a
coincidence, two is a pattern" — applied to the act of generalizing itself.)

---

## 2. The forks, in order, and how each resolved

The design was reached by a sequence of corrections — several of them the owner catching the agent
mid-assumption. Recording them because the *corrections* are where the real design lives.

### 2.1 Document vs. framework → a repo of blueprints, expressed language-agnostically
The first question was whether "generalize" meant *doctrine* (write the shape down) or *code* (a
shared library). The answer landed as **neither, exactly**: a **`blueprints` repo** holding
**language-agnostic descriptions of product shapes** — architecture + contracts + conformance,
expressed in **prose and pseudo-code**, not a library anyone links against.

**Correction that mattered:** the agent initially said "foundation crate," baking Rust into the
*definition* of a blueprint. The owner corrected: **do not limit blueprints to crates/Rust.** A
blueprint describes a shape realizable in *any* stack. Pseudo-code is normative *because* a concrete
language snippet would quietly privilege one stack. Real code appears only as labeled reference.

### 2.2 One blueprint vs. many → the repo is plural; this is blueprint #1
The agent first treated "the blueprint" as singular. The owner corrected: **this is but one blueprint
of (hopefully) many.** That forced a **meta-layer** — a definition of what a blueprint *is* and how
the repo is organized — distinct from any single blueprint. (§2.6.)

### 2.3 The name of the shape
Settled by the owner: **`simple-hardware-controller-service`.** "Simple" is load-bearing — it
deliberately **excludes** the high-complexity class (multi-device orchestration, media pipelines,
distributed control). CameraConductor is explicitly *out of scope* as a validator precisely because
it is that complex, different beast; the validation target is a **light meter** — a simple peer of
the lights.

### 2.4 The southbound transport is pluggable, and "BT" must not be required
The agent understood "surface all connection methods." The owner sharpened it into the load-bearing
rule: **the hardware defines what it is required to connect to it.** BT (and other transports) *appear*
in the blueprint — named, discussed, given contracts and pseudo-code, because a blueprint that only
says "implement an abstract seam, good luck" is useless — but **none is required.** The blueprint
provides the *catalog + the seam*; the hardware picks.

### 2.5 The northbound protocol is ALSO pluggable — and the WS doctrine is mostly protocol-agnostic
The pivotal insight: **WebSocket falls in the same boat as BT.** A product of this shape might expose
itself over **REST or gRPC**, not WS. So the northbound protocol is a second pluggable edge. And the
valuable corollary the owner named: the existing WebSocket Control Doctrine **is immensely valuable to
a REST project** — because ~80% of it (source of truth, asymmetric shape, confirm-by-observation,
headless-clean, identity handshake) is **protocol-independent**; only the frame mechanics are
WS-specific. So WS is *one northbound adapter*, and the protocol-agnostic principles belong to the
*contract*, not the adapter.

**This revealed the true structure:** if *both* edges are pluggable, the reusable asset is the
**core in the middle** — the stuff true regardless of which protocol or transport you pick.

### 2.6 The meta-layer → index + thin charter (option B)
Three options were weighed: (A) pure index, (B) index + a thin charter defining a blueprint's
required *anatomy*, (C) charter + shared cross-blueprint conventions. **B was chosen.** A pure index
(A) enforces nothing, so blueprint #2 resembles #1 only by luck — failing the "all look/feel the
same" goal. (C) over-reaches: asserting "all blueprints compose CommonMind / are
language-agnostic" is a guess about blueprints not yet met. **B** defines only the *container*
(anatomy + folder convention + compose-don't-duplicate) and stays silent on future blueprints'
domains — consistency without over-claim. The charter lives **in the repo** (self-describing), not in
CommonMind.

### 2.7 "Plugins" vs. "modules", and the loading mechanism → compile-time, in-tree modules
The big architectural fork. The owner initially reached for **plugins** in the OBS/nginx/Audacity
sense (in-process dynamic library loading — `.so`/`.dll`/`.dylib`), then reconsidered. The analysis:

- **Naming:** "plugin" over-promises a third-party, dynamically-loaded extension model. **"Module"**
  names the architectural role — a swappable unit behind a contract — independent of loading
  mechanism. Chosen: **modules.**
- **Mechanism, weighed against the North Stars (Reliability #1):** in-process dynamic loading trades
  away crash isolation — *a crashing module takes down the whole service* (the owner's own OBS
  example). For a **reliability-first** service, that directly attacks the top value. Dynamic loading
  buys "no recompile" at the cost of NS #1 — a bad trade.
- **The owner's decisive reframe:** *the value was never "avoid recompiling" — it's "don't touch the
  service code."* A recompile is cheap and safe; editing the core to add hardware is what's expensive
  and dangerous. **Compile-time, in-tree modules** deliver the whole payoff ("focus on the hardware,
  don't touch the core") while keeping every reliability benefit of one statically-verified binary.
- **Recorded non-decision:** if third-party/cross-app module distribution ever becomes real, the
  honest path for *this* family is **out-of-process** (preserves crash isolation), **not** in-process
  dynamic loading. But that is a future decision, not this shape.

### 2.8 The critical correction — modules are NOT "add hardware"; drivers are
The agent had conflated two seams. The owner drew the line sharply: **modules are purely northbound
and southbound *connection providers* — standard, device-agnostic protocol/transport plumbing.**
Adding **controlled hardware** is a *driver* (device-specific command semantics) + a descriptor —
that is **core design, not a module.** The data always originates in the **core + drivers**; modules
only *carry* it.

**The owner's analogy (imperfect but close):** the core is like a web server's application/HTTP logic
— it cares about the meaningful layer and *could not care less how the bytes physically arrived*. The
north/south modules are the connection providers that care about the lower layers (the physical
"how the bytes move"). **Modules are dumb pipes; the core + drivers are the brains.**

This is *the* clarifying distinction of the whole effort. It made the module contracts radically
simpler (a conduit with a byte/message contract, zero device knowledge) and revealed the real
payoff: **the connection modules are shared, reusable plumbing; the drivers are the small per-hardware
thing you write.**

### 2.9 The core↔module boundary → researched, not invented
The owner directed: **flesh out the boundary properly; do research on best practices and pitfalls.**
A thin-then-deep web sweep found the architecture is a **textbook instance of two independently-
developed patterns that converge** — **Hexagonal / Ports & Adapters** (Cockburn) and the **IoT
Gateway northbound/southbound** pattern. Convergence from unrelated traditions = strong evidence the
shape is right. The research (see [the artifact](../research/2026-07-15-core-module-boundary-research.md))
settled several rules:

- **The core owns and DEFINES the contracts; modules/drivers depend on the core, never the reverse.**
  The dependency arrow points *inward.* This is what keeps the core protocol/transport-agnostic.
- **Driving ports** (northbound — surfaces invoke the core) vs. **driven ports** (southbound — the
  core invokes hardware).
- **Adapter ≠ driver:** the IoT tradition independently confirms "southbound protocol adaptors manage
  protocol *drivers*" — the transport (module) and the device driver (not a module) are separate.
- **Asymmetric failure:** southbound failures have *physical* consequences (device state uncertain →
  confidence degrades); northbound failures are *client-facing only* (never touch device truth).
- **Pitfalls to guard:** protocol leakage into the core (the #1 failure), over-abstraction (too many
  granular ports), adapter maintenance overhead, contract versioning.

### 2.10 Async, and core-mediation ("there is a reason it is the core")
Two decisions the owner made on the boundary shape:

- **Async** throughout the southbound seam (matches the reconnect-heavy, notification-driven reality).
- **The core mediates driver↔transport — the driver never touches the transport directly.** The
  owner's reasoning: *"there is a reason it is the core."* Because everything flows through the core,
  the core **observes every delivery outcome and connection event** — which is the *only* way it can
  maintain honest confidence (`confirmed`/`commanded`/`unknown`/`stale`) and single-flight ordering.
  If the driver called the transport directly, the core would be blind to outcomes and could only
  *guess* — breaking the one property the whole shape exists to guarantee. Core-mediation is *why it
  is the core.*

---

## 3. The design that resulted (in one picture)

```
surfaces ─▶ NORTHBOUND MODULE (WS/REST/gRPC — dumb protocol pipe)
                 │  (core's uniform model)
              CORE  ── owns state, confidence, adoption, ordering; protocol- & transport-agnostic;
                 │     the thing you DO NOT touch. Defines all contracts. Mediates everything.
              DRIVER (device-specific translator: intent→bytes, bytes→state. NOT a module.)
                 │  (core hands bytes down, observes every outcome)
              SOUTHBOUND MODULE (BT/USB/net — dumb transport pipe)
                 │
              hardware
```

**Three roles:** the **core** (brains + source of truth), the **driver** (device semantics, the
small thing you write per hardware, not a module), and the **modules** (dumb connection pipes, one
per protocol/transport, compile-time and in-tree, none required).

**Three contracts, all core-owned:** the **Core Model** (the interior vocabulary both edges speak —
`Device`, `Value{value,confidence,since}`, the confidence enum, `Intent`, `Fleet`, `Event`), the
**Northbound** contract (driving), the **Southbound** contract (driven).

**The load-bearing invariants:** every value is `{value, confidence, since}` (never bare);
`DeliveryOutcome` is `queued|written|failed` (never "success"); an intent ack is "accepted," not
"done" (confirm by observation); failures are asymmetric.

---

## 4. Why the abstraction is honest — the litmus test

The whole abstraction rests on one claim: **the same confidence model spans read-oriented and
write-oriented hardware.** Lights are write-only (`commanded`, never `confirmed`); a light meter is
read-oriented (`confirmed` from real readback). If the *same* enum and the *same* contracts serve
both cleanly, the abstraction is real; if the light meter forces the contracts to bend, it is
lights-shaped and wrong.

This is why the **light meter is the designated second instance** — it is deliberately *unlike* the
lights on the axis that matters (read vs. write), so it tests the parts the lighting prototype could
not. The blueprint is, until that build, a **hypothesis extracted from one instance.** The
[`commentary.md`](../../blueprints/simple-hardware-controller-service/commentary.md) records exactly
which claims are proven vs. assumed, and the light-meter build is what will confirm or correct them.

---

## 5. What was deliberately NOT done (recorded so it isn't re-litigated)

- **No dynamic module loading** (in-process `.so`/`.dll`/`.dylib`). Reliability cost (crash-coupling)
  outweighs the "no recompile" benefit; compile-time in-tree gives the real payoff. Out-of-process is
  the future path *if* third-party distribution becomes real.
- **No cross-blueprint conventions asserted** in the charter. With one blueprint, "all blueprints are
  X" is a guess. The charter constrains only the container; genuine cross-blueprint patterns wait for
  a second, different blueprint.
- **CameraConductor was not used as a validator.** Too complex; a different shape. It may *conform* to
  the same underlying doctrines at a high level, but it is not what this "simple" blueprint is proven
  against.
- **LiteController was not refactored** to consume the blueprint. The blueprint was extracted *from
  its design*, not by touching it. A later refactor of LiteController to consume the blueprint is a
  possible step — flagged, not assumed.

---

## 6. Doctrine produced along the way

"Generalize this whole effort" also surfaced several **portable rules** that were promoted to the
CommonMind family (not kept inside this blueprint, per compose-don't-duplicate):

- **WebSocket Control Doctrine** — generalized from CameraConductor's control server; the northbound
  interaction principles behind Contract B.
- **Service Foundations Doctrine** — the daemon plumbing every family service shares (config
  resolution, bind/port, graceful shutdown + durable-state flush, single-instance, headless-clean).
- **Conventions Doctrine** — follow the platform/ecosystem's conventions; deviate only with an
  articulable constraint.
- **Data Format Doctrine** — strict JSON for what we define; markers as fields, not comments.

Plus the top-level **Precedence** directive (when two sources of truth conflict, surface it — never
silently reconcile). The blueprint *composes* all of these rather than restating them.
