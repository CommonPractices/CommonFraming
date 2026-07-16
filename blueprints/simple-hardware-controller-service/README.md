# simple-hardware-controller-service

A **headless, cross-platform, FOSS control service** that owns a fleet of *simple* hardware, is the
**honest source of truth** for that hardware's state, and exposes it to any number of surfaces
through **pluggable edges** — a northbound control-surface protocol (WebSocket / REST / gRPC / …) and
a southbound device transport (Bluetooth / USB / network / …), neither required, both surfaced as a
catalog plus an open seam.

The core is protocol- and transport-agnostic and is *the thing you do not touch*; adding a new
protocol or transport is a **compile-time, in-tree module**; adding new hardware is a **driver** (and
a descriptor). "Simple" is load-bearing: this shape deliberately excludes high-complexity control
systems (multi-device orchestration, heavy media pipelines, distributed control) — see the spec's
scope.

> **Status: DRAFT.** Extracted from [LiteController](../../../LiteController/) (the lighting
> prototype); being proven against a **Bluetooth light meter** as the second instance. Blueprint
> claims are `[V]` only where a real build has confirmed them — see [`commentary.md`](commentary.md).

See **[`spec.md`](spec.md)** for the full shape, contracts, and conformance checklist.
