# Commentary — simple-hardware-controller-service

The **living validation logbook** for this blueprint. A blueprint extracted from one instance is a
**hypothesis**; this file is where it earns correctness (or is corrected) by being built against real
instances. Per the [Charter](../../CHARTER.md) §3, every blueprint claim is tagged `[V]` verified on
a real build · `[D]` documented/researched · `[A]` assumed/untested — and an untested claim is `[A]`,
never fact.

**How to use this doc:** when you build a product against this blueprint, record here — as you go —
where the blueprint **held**, where it **leaked** (a protocol/transport concept reached the core, a
contract was awkward, something was missing), and what you **learned**. A leak is not a failure of the
build; it is the blueprint telling you where it is wrong. Fix the spec, note it here.

---

## Provenance of the blueprint's current claims

The blueprint was **extracted from LiteController's design** (a lighting controller) — itself a
*design*, not yet running code. So even the `[V]`-ish claims are "verified in a worked design,"
awaiting confirmation in a *running* build. Honest current state:

| Claim | Status | Basis |
|---|---|---|
| Agnostic core + two module edges + drivers (hexagonal / NB-SB) | `[D]` | Two independent research traditions converge (§ research artifact). Not yet built. |
| Core owns contracts; arrows point inward | `[D]` | Ports-and-adapters canonical rule. |
| Modules compile-time, in-tree (not dynamically loaded) | `[V]`-design / `[A]`-runtime | LiteController chose this in design; reliability rationale is `[A]` until a build. |
| Core mediates driver↔transport | `[A]` — **counter-example found, see F-1 (2026-07-16)** | Reasoned from the confidence requirement; not yet built. Strongly held, unproven. **DeckLibre achieves the stated property (core observes every outcome) WITHOUT this mechanism** — the §3 rationale over-claims. |
| `{value, confidence, since}`; `confirmed/commanded/unknown/stale` | `[V]`-design | LiteController's core rule; the write-only case is real, the readable case (`confirmed`) is untested here. |
| `DeliveryOutcome = queued|written|failed`, never "success" | `[D]` | The "GATT write success means nothing" finding (LiteController research). |
| Intent ack = "accepted" not "done" (confirm by observation) | `[D]` | WS Control Doctrine, generalized. Untested over REST/gRPC. |
| Async southbound; single-flight ordering | `[A]` — **charter tension, see F-3 (2026-07-16)** | Design decision; not yet built. "Async throughout" is a **runtime-model mandate**, which CHARTER §2 forbids a blueprint from making. |
| Asymmetric failure (SB degrades device confidence; NB client-facing) | `[D]` | IoT/edge research finding. |
| Same confidence enum spans read- and write-oriented devices | `[A]` | **The key litmus test — UNPROVEN.** The light meter (read-oriented) is what validates it. |

---

## Open validation targets

### V-1 — Prove against the Bluetooth light meter (the second instance)
The light meter is the designated proof. It matters because it is **read-oriented** where the light
was **write-oriented** — so it exercises the parts of the blueprint the lighting prototype could not:
- Does `confirmed` (real readback) fall out of the same confidence model as cleanly as `commanded`
  did? (Litmus test for the whole abstraction.)
- Does the **same Bluetooth southbound module** carry the meter's reads and the light's writes
  unchanged — i.e. is the transport genuinely device-agnostic, with only the *driver* differing?
- Does a **read-mostly** device fit the northbound contract (state-heavy, few intents) without
  awkwardness?
- Record every place the blueprint had to bend. `[A]` until done.

### V-2 — Confirm core-mediation in a running build
Core mediation (driver never touches transport) is reasoned, not built. Confirm it actually yields
the honest confidence + single-flight properties claimed, and note the cost/shape of the internal
core↔driver↔transport wiring.

### V-3 — A second northbound protocol
Everything northbound is proven only over WebSocket (in design). Building a REST or gRPC module would
prove the northbound contract is genuinely protocol-agnostic. Not required for V1; a strong
validation if it happens.

---

## Findings log

*(Append dated entries as builds happen. Format: date · what was built · held / leaked / learned ·
what changed in the spec.)*

- **2026-07-15** — Blueprint drafted and extracted from LiteController's design. No running build yet;
  all claims at their initial provenance above. First proof target: the BT light meter (V-1).

- **2026-07-15 — Conformance audit: LiteController spec ⟷ blueprint.** The LiteController spec
  (v0.1.0, written *before* the blueprint) was audited against the blueprint. **Result: the blueprint
  HELD — every divergence was additive or clarifying; no contract had to bend, and no prior
  LiteController decision was reversed.** This is the blueprint's first (design-level) validation, and
  it is meaningful: an abstraction extracted from an instance that then *fits that instance back
  cleanly* is at least self-consistent. Divergences found (all reconciled into LiteController's
  changeset, D-021…D-025):
  - The spec **hardcoded WebSocket as the interface**; the blueprint's **pluggable northbound**
    generalized it (WS = one adapter). → spec now conforms; V1 still ships WS only.
  - The spec **only implied core-mediation**; the blueprint made it a **stated invariant**. → the
    spec was *underspecified*, not wrong — the blueprint surfaced a load-bearing rule the spec needed.
  - The spec treated **transport and driver as peer "traits"**; the blueprint sharpened
    **transport = module, driver = not-a-module + declares its transport**. → clarification.
  - The spec described the southbound seam **behaviourally**; the blueprint gave it an **async
    contract**. → the blueprint was *more specified*.
  - **What this does NOT prove:** the litmus test (V-1) is still open — LiteController is
    **write-oriented**, so the audit did *not* exercise `confirmed`/readback or transport-agnosticism
    across a read device. The audit shows the blueprint is self-consistent with its source; the
    **light meter** is still what proves it generalizes. Two same-shaped write-devices is not yet "two
    is a pattern" across the read/write axis. Status of the read-side claims remains `[A]`.

- **2026-07-16 — Review against DeckLibre** (a control-surface deck controller, `~/repositories/DeckLibre`;
  approved spec 0.4.0 + a settled daemon-architecture draft, realtime core still a stub). DeckLibre was
  **not** built against this blueprint; it was reviewed against it, and its owner **declined** one
  load-bearing claim on the evidence below. This is the blueprint's **first contact with an instance it did
  not come from** — the value is entirely in where it leaked. Findings are **design-level** (DeckLibre's
  core is a stub; LiteController is itself a design), so nothing here is `[V]`-on-a-running-build.
  Supporting artifact: `DeckLibre/docs/_working/research/2026-07-16-family-repo-pattern-adoption.md`;
  DeckLibre decisions `D-033`/`D-034`.

  **F-1 — LEAK: §3 fuses a PROPERTY with a MECHANISM, and the mechanism is not necessary.** `[D]`
  §3 argues *"if the driver called the transport directly, the core would be blind to outcomes and could
  only guess at state — which would break the one property the whole shape exists to guarantee."* Those are
  two separable claims:
  - **The property** — the core observes every delivery outcome + connection event, so it can be honest.
  - **The mechanism** — the core drives a pluggable async transport module; the driver holds no connection.

  DeckLibre has the **property without the mechanism**: its driver owns its `hidapi` handle, but the core
  calls the driver **synchronously and the driver returns to the core**, so *observation is the return
  value*, and `connection_state()` is polled every cycle. The core cannot be blind. Arguably this is a
  **stronger** guarantee than the blueprint's own — a synchronous return is unavoidable, whereas an async
  `delivery_signal(handle, …)` callback is something a core can drop on the floor.
  → **Recommended spec change** (for this repo's owner, not applied here): keep the property as the
  invariant; demote core-mediation-via-transport-module to **one mechanism that satisfies it**, and name
  sync-call-and-return as another. The conformance checklist should test *"the core observes every delivery
  outcome and connection event,"* not *"the driver never touches a transport."* **V-2 should be re-scoped**
  accordingly: it is validating a mechanism, and the property is what needs validating.

  **F-2 — SCOPE GAP: the shape assumes hardware is CONTROLLED, not that hardware DISPATCHES.** `[D]`
  DeckLibre clears every §1.1 "simple" exclusion (its own architecture states **soft** realtime, tens of ms
  — not hard-realtime, no media pipeline, not distributed, not multi-node), so *complexity* is not what
  excludes it. The mismatch is **control-flow inversion**: this blueprint's model is `Intent{device,
  property, value}` flowing **down** from northbound surfaces to hardware — and its §2 diagram lists **"deck"
  as a northbound surface**. A deck controller inverts that: the deck is **southbound hardware acting as a
  surface**, a press travels **up**, and it dispatches an action **elsewhere** (an app, a plugin). Its
  product — bindings, profiles, resolver, dispatch — has **no counterpart in this blueprint**. DeckLibre is
  therefore a **partial instance**: the hardware-ownership half (fleet, state, confidence, connection health)
  fits cleanly; the dispatch half is out of shape. It mines the blueprint and **does not claim conformance**.
  → **Recommended:** state the assumption in §1.1 — *the hardware is controlled by surfaces; hardware that
  is itself an input surface dispatching elsewhere is a different shape.* Per CHARTER §5, do **not** try to
  generalize a second shape from this one instance; if an input-surface-dispatch blueprint is ever wanted,
  it needs its own two instances.

  **F-3 — CHARTER TENSION: "async throughout" is a stack mandate.** `[D]`
  CHARTER §2: a blueprint is *"Not a stack mandate. It never requires a language, runtime, or packaging."*
  `spec.md` §6 mandates **"Async throughout"** for the southbound. That is a concurrency/runtime-model
  requirement, and it is exactly what made this blueprint unadoptable wholesale for a soft-realtime product:
  DeckLibre's keystone is a dedicated **synchronous** realtime thread walled off from an async subsystem
  world, chosen so nothing a plugin or web UI does can stall input→face. Adopting §6 would drag `.await`
  machinery onto the one path that design exists to keep clean.
  → **Recommended spec change:** state the southbound seam **behaviourally** (outcomes are observed;
  delivery is not assumed) and let async-vs-sync be the realization's choice. Note the irony worth keeping:
  the LiteController audit above logged *"the spec described the southbound seam **behaviourally**; the
  blueprint gave it an **async contract** → the blueprint was more specified."* **F-3 says that was the
  blueprint over-reaching, not improving** — the spec's behavioural description was the charter-compliant
  one.

  **What HELD** `[D]`: the honesty vocabulary — **`DeliveryOutcome = queued|written|failed`, never
  "success"**, and **`{value, confidence, since}`, never a bare value** — landed cleanly and was **adopted by
  DeckLibre** (`D-033`). It caught a real defect: DeckLibre's `push_face() -> Result<(), DriverError>` is
  precisely the success/fail boolean §6 warns about, and the repo had **no runtime confidence model at all**.
  This is the blueprint's first demonstrated *external* value, and notably it is the part that survived the
  instance that rejected the mechanism around it — evidence the **vocabulary is the portable core** and the
  plumbing is the local choice.
