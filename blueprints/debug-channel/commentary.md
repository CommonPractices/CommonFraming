# debug-channel — Validation Logbook

A blueprint extracted before any instance is built is a **hypothesis.** This logbook records
where the shape held, where it leaked, and what was learned — each claim tagged `[V]` (held on a
real build) / `[D]` (documented / reasoned but unbuilt) / `[A]` (assumed, untested). Charter §3:
an untested claim is `[A]`, not fact.

**Status:** no instance built yet. Everything below is `[A]` or `[D]`. **LiteController**
(`litecontrollerd`) is the intended first proving instance; this blueprint currently blocks its
engine work, so the shape is expected to be exercised and corrected soon.

---

## Provenance

`[D]` This blueprint was **not** extracted from a running debug channel. It was derived from a
design conversation (2026-07-20) grounded in the family doctrines and in Android's `adb` model as
prior art. The [Debugging Doctrine](../../../CommonMind/debugging-doctrine.md) (the principle)
and this blueprint (the shape) were authored together; neither has a built instance yet. That
makes the whole spec a hypothesis pending LiteController.

## Claim ledger

| # | Claim | Tag | Note |
|---|-------|-----|------|
| 1 | A root debug surface is a superset of control, and "nothing a normal client needs lives only in debug" keeps the front door complete (§1). | `[A]` | Sound in principle (ws-control §0); untested that a real product's debug needs actually *do* decompose cleanly into "normal intent" vs "root-only verb." LiteController will show whether the line holds. |
| 2 | Explicit privilege is not the "privileged back channel" §0 forbids — `sudo`-model (§2). | `[D]` | Owner-ratified reconciliation of a real doctrine collision; reasoned, not yet exercised by an integration. |
| 3 | Two dependent gates (local → network), Android USB→Wireless model (§3). | `[D]` | Design decision; the dependency (network impossible unless local on) is stated, not yet enforced in code. |
| 4 | The three coupled Gate-2 sub-settings — settable credential (distinct from control), off-box bind, host/network allowlist of the operator's own addresses, default most-restrictive (§3). | `[A]` | Untested that the allowlist + credential compose cleanly with ws-control §3's auto-provision-on-bind. Verify on first off-box build. |
| 5 | Attach = ephemeral port + mDNS `<slug>-debug`, advertised only while enabled (§4). | `[A]` | Mechanic borrowed from network-allocation; not yet run. mDNS-while-enabled lifecycle (advertise on enable, withdraw on disable) is the untested part. |
| 6 | **Both gates advertise** — a deliberate deviation from adb (which advertises wireless only, not USB). | `[D]` | **Recorded deviation** (owner choice 2026-07-20, rationale: discoverability / ease of use). PD run: Best-Practices lens leaned toward network-only (adb convention; mDNS is a network protocol), NS leaned to ease-of-use; resolved as a stated, recorded deviation per conventions-doctrine — allowed *because* recorded here. Revisit if the loopback advert proves to be noise. |
| 7 | Contract supersets ws-control (`hello/state/delta`, fire-and-forget, confirm-by-observation) + observe/mutate/reveal families (§5). | `[A]` | The superset claim is untested; whether confirm-by-observation stays coherent when debug can *force* internal state (a forced state produces a delta that is not a real device readback) is a sharp open question for LiteController's write-only case. |
| 8 | Observe streams are independently on/off and filterable, not a firehose (§5.A). | `[A]` | Owner-required capability; filter-spec design (level/subsystem/property/device) is sketched, not specified to the wire. First build defines the actual filter grammar. |
| 9 | Root-only verbs (force-state / inject-fault / override-confidence / force-test-state) are for the abnormal only (§5.B). | `[A]` | The *set* is illustrative, not proven complete or minimal. LiteController's write-only device makes `override-confidence` concrete; the others are speculative until a product needs them. Do not enumerate speculatively — grow the set from real need. |
| 10 | §4 redaction floor holds on every observe stream, sole exception the audited reveal (§5.C, §8). | `[A]` | The structural redaction + the audited-reveal path must both be **verified by attack** on the first build (assert a wrapped secret and a raw `trace` frame reach the debug client as `***`; assert a reveal is logged). Nothing attacked yet. |

## Open questions for the first instance (LiteController)

- **Claim 7 / confirm-by-observation under force-state:** when debug forces an internal state, the
  resulting `delta` is not a device readback — how does the surface mark that honestly so a debug
  client does not mistake a forced value for a confirmed one? (Ties to the write-only confidence
  model.)
- **Claim 9 / verb minimality:** does LiteController actually need all four root-only verbs, or
  fewer? The set must be *grown from need*, not asserted.
- **Claim 4 / gate composition:** does the debug credential + allowlist sit cleanly beside
  ws-control §3's control-side auth, or do they contend?
- **Claim 8 / filter grammar:** what is the actual on-the-wire filter spec, and is it shared with
  the logging level system?

Each answer, once built, promotes its claim from `[A]`/`[D]` toward `[V]` — or corrects the shape.
