# debug-channel — Blueprint Specification

The shape of a family product's **root diagnostic surface.** Normative form is prose + the
message-shape descriptions here; any concrete code is a labeled reference realization, never the
normative statement (Charter §1).

**This blueprint is the SHAPE. The PRINCIPLE is the
[Debugging Doctrine](../../../CommonMind/debugging-doctrine.md)** — it owns *what the
capability owes and how it is postured* (root; off-by-default; two dependent gates; the secret
carve-out). This spec owns *how you build one* and does not restate the doctrine's rules; it
cites them.

---

## 1. What this shape is

A **separate, discoverable, root-privileged surface** exposed by a product, reachable only when
explicitly enabled, that lets an operator **observe everything** and **do anything** the running
product can do — the tool of last resort when the ordinary interface cannot explain a fault.

It is **not** a privileged mode bolted onto the control channel. It is its own surface (its own
listener, its own credential, its own discovery), which is precisely what keeps it from being the
"privileged back channel" [WS-Control §0](../../../CommonMind/ws-control-doctrine.md)
forbids: it is **explicit and advertised**, like `sudo` alongside a user account, not concealed
(Debugging Doctrine §2).

**Superset, not a parallel feature set.** Everything the control interface can do, debug can do
(root ⊇ control). The invariant that keeps the front door honest: **nothing a normal client
legitimately needs may live only in debug** (Debugging Doctrine §1). Debug adds only what no
normal client should do.

## 2. Architecture

```
                 ┌─────────────────────────────────────────┐
                 │              the product                 │
                 │   (single source of truth for its state) │
                 │                                           │
   control  ─────┤  control surface  ──┐                     │
   clients       │  (ws-control:       │   one authoritative │
                 │   hello/state/delta,│   model + single-   │
                 │   fire-and-forget)  ├── flight path to    │
                 │                     │   the work          │
   debug    ─────┤  DEBUG surface  ────┘                     │
   client        │  (root superset;                          │
   (rare)        │   separate listener,                      │
                 │   credential, discovery)                  │
                 └─────────────────────────────────────────┘
```

- **Same source of truth.** The debug surface reads and drives the *same* authoritative model
  the control surface does — it is a second door into one product, never a second product.
- **Separate listener.** Debug binds its own endpoint (§4), distinct from the control port.
- **Normal mutation flows through the control contract** (§5); only abnormal, root-only actions
  are debug-native verbs. This is what keeps the front door complete.

## 3. Security posture (cited, not restated)

The posture is owned by [Debugging Doctrine §3](../../../CommonMind/debugging-doctrine.md).
The shape must implement it exactly:

- **Off by default; absent until enabled.** No listener, no advert, until Gate 1 is set.
- **Gate 1 — local debug** (Android USB-Debugging analog): enables the root surface
  **loopback-only.**
- **Gate 2 — network debug** (Android Wireless-Debugging analog): a **dependent** setting
  (impossible unless Gate 1 is on). Enabling it opens the surface off-box and, in the same act,
  sets its three coupled sub-settings:
  1. a **settable credential**, distinct from the control credential (root ⊋ control → its own
     key);
  2. the **off-box bind**;
  3. a **host/network allowlist** of the **operator's own** Host(s)/Network(s) — arbitrary
     operator-supplied IPs/CIDRs, *not* family address space — defaulting **most-restrictive**
     (empty = nothing reaches root even with the credential).
- **Two independent off-box gates:** a valid credential **and** an allowed source. Either alone
  is insufficient.

## 4. Attach & discovery

- **Transport:** a WebSocket surface (same family idiom as ws-control), on an **ephemeral port**
  (OS-assigned, IANA 49152–65535 per
  [network-allocation](../../../CommonMind/network-allocation-doctrine.md) §1.3) — a debug
  surface is transient and need not hold a fixed port.
- **Discovery:** advertised over **mDNS as `<product-slug>-debug`**
  ([network-allocation §2](../../../CommonMind/network-allocation-doctrine.md)), **only
  while enabled** (absent from discovery when off).
- **Both gates advertise.** Gate 1 (local) advertises loopback-scoped; Gate 2 (network)
  advertises on the reachable interface. *This is a deliberate deviation from the adb de-facto
  convention (which advertises wireless only, not USB), chosen for discoverability / ease of use;
  recorded as a conscious deviation per
  [conventions-doctrine](../../../CommonMind/conventions-doctrine.md) — see
  [commentary](commentary.md).*

## 5. The contract — a superset of ws-control

The debug surface **inherits the entire [WS-Control](../../../CommonMind/ws-control-doctrine.md)
contract** and adds three verb families. It does not reinvent the message shape.

**Inherited, unchanged:**
- `hello → state → delta`; a client builds its model from `state`, keeps it current with `delta`.
- **Fire-and-forget intents** and **confirm-by-observation** ([WS-Control §2](../../../CommonMind/ws-control-doctrine.md))
  — the ack is the resulting `delta`, never a trusted send. Root does not exempt debug from this
  truth about real hardware.

**Added family A — Observe (root visibility, controllable).** Beyond the control channel's public
`state`/`delta`, debug can subscribe to four internal streams, **each independently
on/off/filterable** (by level, subsystem, property, device — logcat-style filter specs), never an
undifferentiated firehose:

| Stream | What it exposes |
|--------|-----------------|
| **internal state** | what the control channel hides: queue depth, retry counters, the raw confidence source, mode-gate reasons, connection internals, caches. |
| **wire frames** | live tap of the southbound transport — raw BLE-mesh frames, PTP chatter, per-byte codec steps (the `trace` firehose, on demand). |
| **log firehose** | the complete log stream at any level incl. `trace`, backlog-on-connect then live (the [logging §1](../../../CommonMind/logging-diagnostics-doctrine.md) published stream, root-privileged). |
| **introspection** | interactive read queries against the running product — dump a structure, evaluate a state expression, list active sessions (an `adb shell`-like read surface). |

Subscribe / unsubscribe / set-filter are the verbs; every stream obeys the **§4 redaction floor**
(secrets `***`) unless explicitly revealed (family C).

**Added family B — Mutate (root-only verbs, for the abnormal only).** Any **normal** state change
is sent as an ordinary control **intent** (debug, being root, may send those) and confirmed by
observation — so nothing normal lives only in debug. Debug-native verbs exist **only** for what no
normal client should ever do:

- **force-state** — set an internal state directly, bypassing a mode gate the control path
  enforces.
- **inject-fault** — induce an error/edge condition for diagnosis.
- **override-confidence** — override the honesty field a write-only device reports (a diagnostic
  lie, deliberately marked).
- **force-test-state** — drive the product/device into a known test condition.

These are loaded guns; they are root-only by construction and never mirror a front-door capability.

**Added family C — Reveal (the one secret carve-out).** A **per-secret, explicit, audited**
request to unveil a real secret value, owned as a principle by
[Debugging Doctrine §4](../../../CommonMind/debugging-doctrine.md) (reveal-not-enable applied
to secrets). Default is redacted; the reveal is a conscious request for *one* value and is itself
logged. The passively-observed session still sees `***`.

## 6. Composed doctrines (cited, never restated)

| Doctrine | What it governs here |
|----------|----------------------|
| [Debugging Doctrine](../../../CommonMind/debugging-doctrine.md) | the principle: root, posture, gates, the secret carve-out. **This blueprint is its shape.** |
| [WS-Control](../../../CommonMind/ws-control-doctrine.md) | §0 (explicit ≠ back channel), the inherited contract, confirm-by-observation. |
| [Logging & Diagnostics](../../../CommonMind/logging-diagnostics-doctrine.md) | the log stream (§1) debug reads; the §4 redaction floor binding every observe stream. |
| [Secrets & Credentials](../../../CommonMind/secrets-credentials-doctrine.md) | the debug credential's handling; the value a reveal unveils. |
| [Decision Doctrine](../../../CommonMind/decision-doctrine.md) | §4–§5 safe/survivable default; reveal-not-enable (secret reveal; the two gates). |
| [Service Foundations](../../../CommonMind/service-foundations-doctrine.md) | the config chain that resolves the gates/credential/allowlist; bind-loud-and-fatal. |
| [Network Allocation](../../../CommonMind/network-allocation-doctrine.md) | the ephemeral-port + `<slug>-debug` mDNS attach mechanic. |

## 7. Known adapters and the open seam

- **First (intended) instance:** LiteController `litecontrollerd` — write-only BLE-mesh lighting;
  the wire-frame stream (raw mesh frames) and override-confidence verb are the sharp cases (a
  write-only device *only* has commanded confidence, so a diagnostic override is meaningful).
- **The open seam:** the four observe streams and the root-only verb set are the
  product-specific fill — a product declares which streams it can source (a product with no
  southbound wire has no `wire frames` stream) and which root verbs its domain supports. The
  *shape* (separate surface, superset contract, gates, redaction floor) is invariant; the
  *catalog of streams and verbs* is the specific.

## 8. Conformance checklist — you have built one when…

- [ ] **Separate surface, not a mode.** Debug is its own listener/endpoint with its own
      credential and discovery — not a privileged flag on the control channel (§1, §2).
- [ ] **Off by default, two dependent gates** (§3) — no listener/advert until Gate 1; Gate 2
      impossible unless Gate 1 is on and couples a distinct credential + off-box bind + a
      most-restrictive host/network allowlist of the operator's own addresses.
- [ ] **Two independent off-box gates enforced** — a valid credential **and** an allowed source;
      neither alone admits.
- [ ] **Discovered as `<slug>-debug` only while enabled** (§4); both gates advertise (deviation
      recorded).
- [ ] **Contract supersets ws-control** (§5) — inherits `hello/state/delta`, fire-and-forget,
      **confirm-by-observation**; adds observe/mutate/reveal families.
- [ ] **Normal mutation goes through control intents; root-only verbs only for the abnormal** —
      nothing a normal client needs lives only in debug (front door stays complete).
- [ ] **Observe streams are controllable** — independently on/off and filterable, not a single
      firehose.
- [ ] **§4 redaction floor holds on every observe stream** (incl. `wire frames` and `log
      firehose`), **verified by attack**; the sole exception is the per-secret, explicit,
      **audited** reveal.
- [ ] **Every composed doctrine is cited, none restated** (Charter §4).
