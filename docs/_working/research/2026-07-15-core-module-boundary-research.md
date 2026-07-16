# Core↔Module Boundary — research

**Feeds:** the `simple-hardware-controller-service` blueprint spec §§2–6 (architecture + the three
contracts). **Purpose:** ground the boundary design in real best-practice and known pitfalls rather
than inventing it. **Method:** thin-then-deep web sweep (July 2026), inline (no subagents).

**Provenance:** `[D]` documented from the sources below unless noted.

---

## 1. The one-line finding

Our architecture is a **textbook instance of two independently-developed patterns that converge on
the same rules** — **Hexagonal / Ports & Adapters** (Cockburn; enterprise software) and the **IoT
Gateway northbound/southbound** pattern (edge/device). Convergence from unrelated traditions is strong
evidence the shape is right.

---

## 2. Ports & Adapters (Hexagonal) — the rules we adopt

- **The core (hexagon) owns and DEFINES the port interfaces; adapters depend on the core, never the
  reverse.** The dependency arrow points *toward* the core. This is *the* load-bearing rule — it is
  what keeps the core free of protocol/transport concepts. `[D]`
- **Ports are contracts in the core's own language**, "without any dependency on external libraries
  or frameworks." Adapters "strictly implement these interfaces, encapsulating all external
  dependencies." `[D]`
- **Two port directions:** *driving / primary / inbound* (external actors invoke the core — our
  northbound) vs. *driven / secondary / outbound* (the core invokes the outside — our southbound).
  Both are defined by the core, implemented by adapters; they differ only in invocation direction.
  `[D]`
- **Several adapters can implement one port** — "data can be provided by a user through a GUI or a
  command-line interface, by an automated data source, or by test scripts." This is exactly our
  "many protocols on one northbound port / many transports on one southbound port." `[D]`

### Pitfalls (directly applicable)
- **Protocol/infrastructure leakage into the core** — the #1 failure: "teams may inadvertently allow
  infrastructure concerns … to leak into the domain layer." Guard: core pure, all specifics in
  modules. `[D]`
- **Over-abstraction** — "too many small, granular ports and adapters, making the overall structure
  difficult to navigate." Guard: few, meaningful, purpose-specific contracts. `[D]`
- **Adapter maintenance overhead** — each adapter is a thing to maintain, "justified only if the
  application component requires several input sources and output destinations." We do — multiple
  protocols and transports. `[D]`
- **Contract versioning** — "regularly update and document port contracts to ensure clear boundaries
  and facilitate swaps of implementations." Ties to the forward-compatible-format doctrine. `[D]`

---

## 3. IoT Gateway northbound/southbound — the domain mapping

The same pattern, in our exact domain and vocabulary:

- **Southbound** = toward hardware. "Southbound protocol adaptors manage protocol **drivers**
  responsible for posting collected data to core services." → **confirms the module-vs-driver split:
  the transport adapter and the device driver are separate things.** `[D]`
- **Northbound** = toward surfaces/apps. "Northbound protocol adaptors … translate data using various
  protocols (HTTP(S), WebSocket, CoAP, MQTT)." → the northbound edge is a protocol conduit, protocol
  chosen per deployment. `[D]`
- **Southbound transports are heterogeneous and often vendor-defined** — "Zigbee, Bluetooth, WiFi,
  USB, SimpliciTI." → the transport catalog is open-ended; the hardware dictates it. `[D]`

### The reliability-relevant finding (matters because NS #1 = reliability)
**The two edges fail *differently*:** `[D]`
- **Southbound failures have immediate *physical* consequences** — a device may need resetting; the
  hardware may be in an unknown state.
- **Northbound failures are *systemic / at scale*** — governance, security exposure, throughput.

→ The boundary should treat them **asymmetrically**: a southbound failure is a *"device state is now
uncertain → confidence degrades"* event (feeds `confirmed/commanded/unknown/stale` directly); a
northbound failure is a *"a surface got a degraded response"* event — far less dangerous, never
touches device truth. This is now spec §7 (asymmetric failure handling).

---

## 4. How this shaped the blueprint

| Finding | Where it landed in the spec |
|---|---|
| Core owns contracts; arrows inward | §3 (core owns all three contracts) |
| Driving vs. driven ports | §5 northbound (driving), §6 southbound (driven) |
| Adapter ≠ driver (southbound has both) | §2.1, §8 (driver is not a module) |
| One port, many adapters | §2.3 (catalog + open seam, none required) |
| Protocol-leakage pitfall | §10 conformance test ("no protocol/transport concept in core") |
| Over-abstraction pitfall | contracts kept thin (§4–6), few and purpose-specific |
| Contract versioning pitfall | §9 cites forward-compatible-format doctrine |
| Asymmetric edge failure | §7 (asymmetric failure handling) |

---

## 5. Sources
- Hexagonal architecture (Wikipedia): https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)
- Ports & Adapters — Real-World Survival Tips (SmartCR): https://smartcr.org/architecture/hexagonal-architecture-tutorial/
- Northbound and Southbound interfaces in Edge IoT (WizzDev): https://wizzdev.com/blog/northbound-and-southbound-interfaces-in-edge-iot/
- IoT Gateway Architecture (Wevolver): https://www.wevolver.com/article/iot-gateway-architecture-edge-vs-cloud-protocol-translation-and-deployment-patterns
- Hexagonal architecture pattern (AWS Prescriptive Guidance): https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/hexagonal-architecture.html

*(Sources are `[D]` — community/vendor architecture writing and one canonical reference. The
convergence across independent traditions is the strength; specifics were cross-checked, not
single-sourced.)*
