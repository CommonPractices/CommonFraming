# Blueprint: simple-hardware-controller-service

> **STATUS: DRAFT.** This blueprint is extracted from one instance (LiteController) and is a
> **hypothesis until a second instance proves it** (a Bluetooth light meter). Claims are tagged
> `[V]` verified on a real build · `[D]` documented/researched · `[A]` assumed/untested. Untested
> claims are `[A]`, not fact. Validation is recorded in [`commentary.md`](commentary.md).
>
> Pseudo-code is **normative**; it is language-neutral. Any real-language snippet elsewhere is a
> labeled reference realization, not the contract.

---

## 1. The shape

A **headless, cross-platform service** that:

- **Owns a fleet** of hardware devices of a *simple* class and is the **single source of truth** for
  their state.
- Is **honest about state** — it never reports a value as more certain than it is (see §4, the
  confidence model). This is the load-bearing property, and it is why the core mediates everything
  (§3).
- Exposes that fleet to **any number of surfaces** — a browser admin page, a control surface (e.g. a
  Stream Deck), other software, an AI agent, another service — through a **northbound** edge.
- Talks to hardware through a **southbound** edge.
- Runs **headless** (no UI required; correct with zero clients) and cross-platform, including small
  targets (e.g. a Raspberry Pi).

**Both edges are pluggable and neither is required by the shape** — the northbound *protocol*
(WebSocket / REST / gRPC / …) is a deployment/product choice; the southbound *transport* (Bluetooth /
USB / network / …) is dictated by the hardware. The core does not know or care which.

### 1.1 Scope — what "simple" excludes

"Simple" is a real boundary, not a mood. This blueprint fits hardware where control is essentially
*"set/read named properties on a device, honestly."* It **excludes** the high-complexity class:
multi-device orchestration with cross-device state, heavy real-time media pipelines (live video),
distributed/multi-node control planes, hard-real-time timing. A control service in that class is a
*different shape* and is out of scope here. (A camera-tethering service with live-view and a
master-node topology is the canonical out-of-scope example.)

---

## 2. Architecture: an agnostic core, two module edges, and drivers

This is **hexagonal / ports-and-adapters** (Cockburn) and the **IoT gateway northbound/southbound**
pattern — two independent traditions that converge on the same rules `[D]` (see
[`../../docs/_working/research/`](../../docs/_working/research/) for the research artifact).

```
   surfaces (admin UI, deck, agent, other service)
        │
   ┌────┴──────────────┐   NORTHBOUND MODULE(s)  — protocol conduit (WS / REST / gRPC / …)
   │  driving port      │   device-agnostic; protocol mechanics only; ZERO device/business logic
   └────┬──────────────┘
        │  (core's uniform model)
   ┌────┴───────────────────────────────────────────────┐
   │  CORE  (the hexagon)                                 │  owns: state, confidence, adoption/
   │  protocol- AND transport-agnostic                    │  identity/matching, fleet ownership,
   │  the thing you DO NOT touch                          │  single-flight ordering, config/lifecycle
   └────┬───────────────────────────────────────────────┘
        │  core CALLS the driver (intent → bytes; bytes → state)
   ┌────┴──────────────┐   DRIVER — device-SPECIFIC translator. NOT a module; domain content.
   │  (device semantics)│   the small thing you write per hardware.
   └────┬──────────────┘
        │  core hands the driver's bytes to the transport, and OBSERVES every outcome
   ┌────┴──────────────┐   SOUTHBOUND MODULE(s) — transport conduit (BT / USB / net / …)
   │  driven port       │   device-agnostic dumb pipe; ZERO device logic
   └────┬──────────────┘
        │
      hardware
```

### 2.1 The three roles, kept distinct

| Role | Owns | Knows the device? | Is it a module? |
|---|---|---|---|
| **Core** | state, confidence, adoption, fleet ownership, ordering, the "what" | abstractly (via drivers) | no — it *is* the core |
| **Driver** | device-specific command semantics ("what the bytes mean") | **yes, deeply** | **no** — domain content the core calls |
| **Northbound module** | speaking a surface protocol; carrying the core's model to surfaces | **no** | **yes** |
| **Southbound module** | speaking a transport; carrying bytes to hardware | **no** | **yes** |

**Modules are dumb pipes; the core + drivers are the brains.** A module holds *no* device knowledge
and *no* business logic — it carries already-formed data over a wire. The core owns *what* to say;
the driver owns *what the device-specific bytes are*; the module owns *how the bytes travel*.

Note the southbound split: the **transport** (a module — device-agnostic byte pipe) and the
**driver** (device-specific semantics, *not* a module) are separate. The same Bluetooth transport
module carries a light's commands or a light meter's reads unchanged; only the driver differs. `[D]`
(the IoT tradition states this as "southbound protocol adaptors manage protocol *drivers*.")

### 2.2 Modules are compile-time, in-tree — not dynamically loaded

Modules live in the source tree and are **compiled in**. Adding a transport or protocol means writing
a module against its contract and recompiling — but the **service core is not touched.** `[V]`
(LiteController chose exactly this.)

**The value is "do not touch the service core," not "avoid recompiling."** A recompile is cheap and
safe; editing the core to add a connection method is expensive and dangerous. Compile-time in-tree
modules deliver the whole payoff while keeping every reliability benefit of one statically-verified
binary.

Dynamic loading (in-process `.so`/`.dll`/`.dylib`, OBS-style) and out-of-process modules are
**deliberately not used.** In-process dynamic loading trades away crash isolation — *a crashing
module takes down the whole service* — which directly attacks the top value of a reliability-first
service. If third-party or cross-application module distribution ever becomes a real requirement, the
honest path for *this* family is **out-of-process** (which preserves crash isolation), not in-process
dynamic loading — but that is a future decision, not this shape. Compile-time in-tree is the model.
`[A]` (rationale; the future path is unproven.)

### 2.3 Connection methods are surfaced richly, required never

The blueprint **names, discusses, and provides contracts + pseudo-code for real transports and
protocols** — Bluetooth/BLE, USB-serial, TCP/UDP on the south; WebSocket, REST, gRPC on the north —
because a blueprint that only says "implement an abstract seam, good luck" is useless. But it
**requires none.** The hardware defines what transport it needs; the deployment defines what protocol
it exposes. A product using a transport or protocol the blueprint never anticipated is a first-class
case: implement the seam, done.

---

## 3. The core owns every contract; the arrows point inward

**The core defines all three contracts below, in the core's own vocabulary. Modules and drivers
depend on the core — never the reverse.** `[D]` This is the load-bearing rule of ports-and-adapters
and it is what keeps the core protocol/transport-agnostic: a module conforms *to the core*, so no
protocol- or transport-specific concept can enter the core.

**Everything flows through the core (core mediation).** The driver never touches a transport
directly. The core calls the driver to produce bytes, hands those bytes to the southbound module,
and **observes every delivery outcome and connection event.** This is not incidental — it is *why*
it is the core:

- It is the only way the core can maintain honest confidence (§4): the core sees whether a write was
  queued/written/failed and whether the connection is up, so it — and only it — can say `commanded`
  vs `stale` vs `unknown`.
- It gives **single-flight ordering** for free: everything funnels through one chokepoint, so
  surfaces cannot race the hardware pipe.

If the driver called the transport directly, the core would be blind to outcomes and could only
*guess* at state — which would break the one property the whole shape exists to guarantee.

---

## 4. Contract A — the Core Model (the interior vocabulary both edges speak)

This is the crown jewel: the **canonical model both edges render.** Because every product of this
shape exposes *this same model*, a northbound module written for one product is reusable verbatim by
another, and a southbound module likewise. Define this well and the edges become thin renderers.

```
Device {
  id            : opaque stable id (the SERVICE's id, not a vendor's — see device-model doctrine)
  friendly_name : human label
  identity      : { method: mesh|address|…, confidence: stable|fragile, … }   # how we know it's THIS device
  model         : reference to its descriptor (device-model + SMT doctrines)
  state         : map<property, Value>
  write_policy  : effective delivery policy (resolved; see §7)
  health        : liveness/connection summary
}

Value      { value, confidence, since }          # NEVER a bare value — see below
Confidence = confirmed | commanded | unknown | stale

Intent     { device, property, value, write_policy_override? }   # a thin, fire-and-forget command
Fleet      = collection of Device + adoption/identity/matching operations
Event      = an internal change notification (what the northbound stream is built from)
```

**Every reported value is `{value, confidence, since}` — never a bare value.** `[V]` (LiteController's
core rule.) The confidence enum:

- `confirmed` — the device reported this back (readable devices).
- `commanded` — we sent it, the delivery pipeline succeeded as far as it can, but no readback exists
  (write-only devices' normal state).
- `unknown` — never commanded this session, or delivery could not be confirmed.
- `stale` — we had a value, but a disconnect / suspected power-cycle invalidated confidence.

This is the [Confidence & Provenance Scoring Doctrine](../../../CommonMind/confidence-scoring-doctrine.md)
applied to **live runtime state**: a value dressed as more certain than it is, is a lie the service
tells its surfaces. Write-only hardware is the exception that proves the rule (`commanded`, never
`confirmed`); readable hardware gets `confirmed` for free. The same enum serves both — which is the
litmus test that the abstraction spans read- and write-oriented devices.

---

## 5. Contract B — Northbound (driving port): the control-surface protocol

The core exposes a uniform surface; a module renders it into a concrete protocol. **What the core
offers (the module maps each to its protocol's mechanics):**

```
full_state()            -> Snapshot        # the complete current fleet state
state_stream()          -> stream<Delta>   # subscribe; only-what-changed thereafter
submit_intent(Intent)   -> Accepted        # fire-and-forget. ACCEPTED ≠ DONE (see below)
log_stream()            -> stream<LogLine>  # loosely-parsed activity/observability feed
identity()              -> Hello           # app-identity handshake; a surface confirms it reached the right service
```

- **`submit_intent` is fire-and-forget; the acknowledgement is the resulting state delta, never a
  trusted send.** `[D]` (ports-agnostic form of the WebSocket doctrine's "confirm by observation.") A
  surface verifies an intent took effect by observing `state_stream`, not by trusting `Accepted`.
  This is *especially* valuable to a REST product, which would otherwise read a `200` as "the
  hardware obeyed."
- **The module owns protocol mechanics only** — framing, serialization (strict JSON per the [Data
  Format Doctrine](../../../CommonMind/data-format-doctrine.md)), bind/port/auth *mechanics*,
  multi-client fan-out, keepalive. It never interprets the model it carries.
- **Adapter mapping (illustrative):** WebSocket → `hello`/`state`/`delta` text frames + an intent
  message ([WebSocket Control Doctrine](../../../CommonMind/ws-control-doctrine.md) is the
  worked adapter). REST → `GET /state`, SSE or long-poll for the stream, `POST` an intent. gRPC →
  server-streaming state, unary intents.

> The WebSocket Control Doctrine is **one northbound adapter** — but most of what it says (source of
> truth, asymmetric shape, confirm-by-observation, headless-clean, identity handshake) is
> **protocol-independent** and belongs to *this* contract; only the frame mechanics are WS-specific.

---

## 6. Contract C — Southbound (driven port): the device transport

A device-agnostic byte/message pipe. **Async throughout.** The core (mediating for a driver) drives
it; the module signals back asynchronously.

```
# core/driver calls DOWN:
open(target)            -> ConnectionHandle | Error     # target is opaque to the core (a BLE UUID, a serial path, host:port)
send(handle, bytes)     -> DeliveryOutcome              # queued | written | failed  — NOT "success"
close(handle)
capabilities()          -> TransportCaps                # readback? notifications/subscribe? MTU/segmentation?

# module signals UP (async):
inbound(handle, bytes)                                   # data arrived, IF the transport supports it (driver decodes)
connection_event(handle, connected|disconnected|degraded|reconnecting)   # feeds the confidence model
delivery_signal(handle, …)                               # any post-hoc delivery info (filter-confirmed, ack, error)
```

- **`DeliveryOutcome` is `queued | written | failed`, never `success`.** `[V]` A transport-level
  "sent" says nothing about whether the device acted — the "a GATT write succeeding means nothing"
  lesson, promoted to contract. Only observed state (or an explicit readback) is truth.
- **`capabilities()` lets the core adapt honestly** — a notify-capable transport can yield
  `confirmed`; a fire-and-forget one caps at `commanded`.
- **`connection_event` feeds confidence directly** — `disconnected` → affected devices' state goes
  `stale`/`unknown`, surfaced loudly.
- **The module holds no device semantics.** It moves bytes; the driver (§8) knows what they mean.

---

## 7. The core's owned responsibilities (transport/protocol-agnostic)

These live in the core and are the same for every product of the shape — the reusable 90%:

- **State & confidence** (§4) — the honest source of truth.
- **Adoption / identity / matching** — discovering devices, binding a stable identity, a risk-graded
  re-match with a configurable confidence threshold (preview-and-confirm below the bar). `[V]`
- **Fleet ownership** — the collection; correct with zero clients (headless).
- **Single-flight ordering** — surfaces cannot race the hardware pipe (falls out of core mediation).
- **Write policy** — a per-device delivery policy (burst / keepalive) with a precedence chain
  (per-command → per-device → global default), reconnect re-assertion. The *engine* is core; the
  device-specific *hints* come from the driver (§8). `[V]`
- **Config / lifecycle / platform integration** — per the [Service Foundations
  Doctrine](../../../CommonMind/service-foundations-doctrine.md) (one config-resolution chain,
  settable bind/port, loud bind failure, graceful shutdown that flushes unrecoverable state, refuse a
  second instance) and the [Conventions Doctrine](../../../CommonMind/conventions-doctrine.md)
  (platform-standard file locations, launch idioms).
- **Asymmetric failure handling** `[D]` — a **southbound** failure degrades *device* confidence
  (physical consequence); a **northbound** failure is *client-facing only* and never touches device
  truth. The two edges fail differently; the confidence model is the common currency.

---

## 8. The Driver contract (device-specific; not a module)

The small per-hardware thing you write. The core calls it:

```
encode(Intent)          -> bytes            # device-specific wire bytes for a core intent
decode(bytes)           -> StateUpdate      # inbound bytes -> core state (readable devices; write-only devices never emit)
describe()              -> DeviceModel      # properties, ranges, capabilities (device-model + SMT doctrines)
write_policy_hints()    -> PolicyHints      # device-specific delivery defaults (the engine is core)
required_transport()    -> TransportReq     # what this hardware needs to connect — "the hardware defines it"
```

The driver is a **pure translator** — it holds no connection and races nothing; the core mediates its
bytes to a transport (§3). Device descriptors it produces follow the [Device-Model
Doctrine](../../../CommonMind/device-model-doctrine.md) (single-inheritance manufacturer →
family → device; the **wire protocol rides the chain at its truthful tier**) and the [SMT
Doctrine](../../../CommonMind/smt-doctrine.md) (relabels = one definition + an open identity
array, sameness earned; a name kept across a behavioural break is marked and its confidence capped).

---

## 9. Composed doctrine (referenced, not restated)

A conformant product follows these — the blueprint **cites** them and adds only shape-specific glue
(per the [Charter](../../CHARTER.md) §4). Restating any of them here would be a defect.

| Doctrine | What it governs for this shape |
|---|---|
| [WebSocket Control](../../../CommonMind/ws-control-doctrine.md) | One northbound adapter + the protocol-agnostic interaction principles behind Contract B. |
| [Service Foundations](../../../CommonMind/service-foundations-doctrine.md) | Config resolution, bind/port, loud bind failure, graceful shutdown + durable-state flush, single-instance, headless-clean, logging, auto-start. |
| [Conventions](../../../CommonMind/conventions-doctrine.md) | Platform-standard file locations, launch idioms, native integration. |
| [Data Format](../../../CommonMind/data-format-doctrine.md) | Strict JSON for our interchange + config/data; markers as fields. |
| [Confidence & Provenance Scoring](../../../CommonMind/confidence-scoring-doctrine.md) | The confidence model (§4), applied to live runtime state. |
| [Device-Model](../../../CommonMind/device-model-doctrine.md) | Driver descriptors: inheritance chain; protocol rides it at its truthful tier. |
| [SMT](../../../CommonMind/smt-doctrine.md) | Relabels (Face A) and name-kept-across-a-break (Face B) in descriptors + relabel-robust recognition. |
| [Forward-Compatible Format](../../../CommonMind/forward-compatible-format-doctrine.md) | Versioning the contracts and on-disk formats so they evolve without breaking. |
| [Artifact Persistence](../../../CommonMind/artifact-persistence-doctrine.md) | Load-bearing research/derivations become dated files, cited bidirectionally. |

---

## 10. Conformance checklist — "you have built one of these when…"

- [ ] The **core is protocol- and transport-agnostic** — no protocol- or transport-specific concept
      appears in core logic. (The test: could you add a REST module and a USB module touching *only*
      new module files?)
- [ ] The core **owns and defines all three contracts**; modules and drivers depend on the core, not
      the reverse.
- [ ] **Northbound = pluggable protocol modules**; **southbound = pluggable transport modules**;
      **compile-time, in-tree**; neither protocol nor transport required by the core.
- [ ] **Drivers are device-specific and are not modules**; the **core mediates** driver↔transport (a
      driver never touches a transport); every delivery outcome + connection event is observed by the
      core.
- [ ] **Every reported value carries `{value, confidence, since}`** — never bare; `DeliveryOutcome`
      is `queued|written|failed`, never "success"; **an intent ack is "accepted," not "done"**
      (confirm by observation).
- [ ] **Async** southbound; **single-flight** ordering through the core.
- [ ] **Failures are asymmetric** — southbound degrades device confidence, northbound is
      client-facing only.
- [ ] **Headless / zero-client-correct**; cross-platform incl. a small target.
- [ ] The product **composes the doctrines in §9** and restates none of them.
- [ ] Connection methods are **surfaced (documented + pseudo-code) but not required**; a novel
      transport/protocol is a new module, not a core change.
- [ ] The build's deviations and unproven claims are recorded in [`commentary.md`](commentary.md) —
      untested claims tagged `[A]`, not fact.
