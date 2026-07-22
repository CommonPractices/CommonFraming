# CommonFraming

The family's repository of **product-shape blueprints** — language-agnostic descriptions of
recurring product *shapes*, so building the next product of a known shape is *"supply the
specifics,"* not *"re-derive the whole architecture."* (The repo is `CommonFraming`; a *blueprint*
is what it holds — the term of art is unchanged.)

A blueprint is **not** a library or a framework. It describes the architecture, contracts, and
conformance of a shape — in prose and pseudo-code — so anyone can build one, in any language or
stack. See **[CHARTER.md](CHARTER.md)** for what a blueprint is, its required anatomy, and how this
repo is organized.

## Blueprints

| Blueprint | Shape | Status |
|---|---|---|
| [simple-hardware-controller-service](blueprints/simple-hardware-controller-service/) | A headless, cross-platform service that owns a fleet of *simple* hardware, is the honest source of truth for its state, and exposes it through pluggable northbound (protocol) and southbound (transport) edges. | **DRAFT** — first blueprint; being proven against a BT light meter. Extracted from [LiteController](../LiteController/). |
| [debug-channel](blueprints/debug-channel/) | A break-glass, **root-privileged** diagnostic surface a product exposes when the normal interface can't explain a fault — a separate, discoverable surface that supersets the control contract (observe internals/wire/logs + root-only force/inject verbs + audited secret reveal). Off by default; two dependent gates (local→network). Shape for the [Debugging Doctrine](../CommonMind/debugging-doctrine.md). | **DRAFT** — hypothesis; intended first proving instance is [LiteController](../LiteController/) `litecontrollerd`. |

## How this repo is organized

- **[CHARTER.md](CHARTER.md)** — the meta-layer: what a blueprint is, the required anatomy every
  blueprint has, the folder convention, and the compose-don't-duplicate rule.
- **`blueprints/<name>/`** — one folder per blueprint, each following the charter's anatomy.
- **`docs/_working/`** — the repo's own drafts, research, and decision record (documentation-doctrine
  layout).

This repo follows the [CommonMind](../CommonMind/) family conventions.
