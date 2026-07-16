# blueprints — Decision Log

The record of what has been decided, rejected, and left open for the `blueprints` repo and its
blueprints. Per decision-doctrine §8: status + why + provenance; per §10, nothing deleted to tidy.

**Provenance:** `[V]` verified · `[D]` documented · `[A]` assumed. **Statuses:** proposed · accepted
· superseded. Only the owner promotes to accepted.

---

### D-001 — `blueprints` repo created; meta-layer = index + thin charter
- **Status:** accepted
- **Date:** 2026-07-15
- **What:** Repo `~/repositories/blueprints` created (local git, no remote, CBB-free). Holds **many**
  reusable product-shape blueprints. Meta-layer = a `README` index + a thin `CHARTER.md` defining a
  blueprint's required **anatomy** (README/spec/commentary/optional reference), the folder convention
  `blueprints/<name>/`, and the **compose-don't-duplicate** rule. Charter is deliberately anemic —
  silent on what future blueprints' *domains* are (don't over-abstract from one instance).
- **Why:** Owner decision, brainstormed. Charter lives in the repo (self-describing), not in
  design-doctrine. `[V]`

### D-002 — First blueprint: `simple-hardware-controller-service`
- **Status:** accepted
- **Date:** 2026-07-15
- **What:** The first blueprint — a headless, cross-platform control service owning a fleet of
  *simple* hardware, honest source of truth, with pluggable **northbound** (protocol) and
  **southbound** (transport) edges. Extracted from **LiteController**; to be **validated against a BT
  light meter** (the read-oriented second instance). Explicitly excludes the high-complexity class
  (multi-device orchestration, media pipelines, distributed control).
- **Why:** Owner decision. `[V]` (design); the blueprint is a **hypothesis until proven on a second
  instance** — see the blueprint's `commentary.md`.

### D-003 — Architecture: agnostic core, two module edges, drivers; core mediates; compile-time modules
- **Status:** accepted
- **Date:** 2026-07-15
- **What:** Hexagonal/NB-SB. **Core** owns state/confidence/adoption/ordering, is protocol- and
  transport-agnostic, and **mediates** driver↔transport (driver never touches transport → core
  observes every outcome → honest confidence + single-flight). **Modules** (NB protocol + SB
  transport) are device-agnostic dumb pipes, **compile-time and in-tree** (not dynamically loaded —
  reliability: no crash-coupling). **Drivers** are device-specific translators, **not** modules.
  **Async** southbound. Connection methods **surfaced richly, required never** — the hardware defines
  its transport.
- **Why:** Owner decisions across the brainstorm (modules-not-plugins; core-mediation "there is a
  reason it is the core"; async). Grounded in research
  (`docs/_working/research/2026-07-15-core-module-boundary-research.md`). `[V]` (design); core
  mediation is `[A]` until a running build (commentary V-2).

### D-004 — Three contracts, core-owned, in the core's vocabulary
- **Status:** accepted
- **Date:** 2026-07-15
- **What:** Core Model (Device/Value{value,confidence,since}/Confidence enum/Intent/Fleet/Event) +
  Northbound (full_state/state_stream/submit_intent[accepted≠done]/log_stream/identity) + Southbound
  (open/send[queued|written|failed]/close/capabilities down; inbound/connection_event/delivery_signal
  up). Core owns and defines all three; modules/drivers depend on the core, never the reverse.
- **Why:** Owner-approved; derived from LiteController's needs + ports-and-adapters research. `[V]`
  (design); the same-confidence-enum-spans-read-and-write claim is the key `[A]` litmus test, unproven
  until the light meter (commentary V-1).

---

## Open items — do NOT build on these

### O-201 — The blueprint is unproven on a second instance
- **State:** open. Extracted from one instance (LiteController design). The BT light meter is the
  designated proof (read-oriented, exercises `confirmed` + transport-agnosticism).
- **Closes when:** a real build against the light meter confirms the contracts hold (or the spec is
  corrected). Tracked in the blueprint's `commentary.md` (V-1/V-2/V-3).
