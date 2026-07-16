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
| Core mediates driver↔transport | `[A]` | Reasoned from the confidence requirement; not yet built. Strongly held, unproven. |
| `{value, confidence, since}`; `confirmed/commanded/unknown/stale` | `[V]`-design | LiteController's core rule; the write-only case is real, the readable case (`confirmed`) is untested here. |
| `DeliveryOutcome = queued|written|failed`, never "success" | `[D]` | The "GATT write success means nothing" finding (LiteController research). |
| Intent ack = "accepted" not "done" (confirm by observation) | `[D]` | WS Control Doctrine, generalized. Untested over REST/gRPC. |
| Async southbound; single-flight ordering | `[A]` | Design decision; not yet built. |
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
