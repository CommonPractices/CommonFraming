# debug-channel

A **break-glass, root-privileged diagnostic surface** a family product exposes so that, when the
normal interface cannot explain what is wrong, an operator can see everything and do anything.
It is **root** — a strict superset of the product's control capability plus deep introspection,
fault injection, and (audited) secret reveal — and *because* it is that powerful it is **off by
default, absent until deliberately enabled**, reached off-box only through two dependent gates.

The shape is deliberately generic: a service, a renderer, a plugin — any family product can carry
one. It is a **separate, discoverable surface** (not a privileged mode on the control channel),
and it **supersets** the [WebSocket Control](../../../CommonMind/ws-control-doctrine.md)
contract rather than reinventing it.

**Status:** DRAFT — first extraction. The principle is the
[Debugging Doctrine](../../../CommonMind/debugging-doctrine.md); this blueprint is the
*shape*. Every claim is a hypothesis (`[A]`) until proven on a real build — **LiteController**
(`litecontrollerd`) is the intended first proving instance, and its needs are why this exists.

- **[`spec.md`](spec.md)** — the blueprint: architecture, the contract/seams, the capability set,
  and the conformance checklist.
- **[`commentary.md`](commentary.md)** — the living validation logbook (`[V]`/`[D]`/`[A]`).

This blueprint **composes** the CommonMind family (it cites the doctrines a conformant
product must follow and adds only what is specific to the shape); it restates none of them. See
the [Blueprint Charter](../../CHARTER.md).
