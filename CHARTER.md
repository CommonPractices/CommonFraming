# Blueprint Charter

The meta-layer of this repo: **what a blueprint is, what every blueprint contains, and how they are
organized.** Thin on purpose — it defines the *container*, not the contents of any particular
blueprint.

---

## 1. What a blueprint IS

A blueprint is a **reusable, language-agnostic description of a recurring product *shape*.** When a
class of product keeps recurring — the same architecture, the same contracts, the same behaviours,
differing only in specifics — the blueprint captures the invariant shape once so the next instance
is *"supply the specifics,"* not *"re-derive the architecture."*

A blueprint describes:

- **The shape** — what this class of product is, and its architecture.
- **The contracts** — the language-neutral interfaces/seams that define conformance.
- **Conformance** — how you know you have built one.

**It is expressed in prose and pseudo-code.** The blueprint is language- and stack-agnostic:
pseudo-code is the normative form. Real code appears only as **clearly-labeled reference
realization**, never as the normative statement — a concrete language snippet would quietly privilege
one stack.

---

## 2. What a blueprint is NOT

- **Not a library or framework.** It ships no code you depend on. It describes the shape so anyone
  can build it, in any language.
- **Not a stack mandate.** It never requires a language, runtime, or packaging.
- **Not a restatement of doctrine.** See §4 — it *composes* the CommonMind family, it does not
  copy it.

---

## 3. Required anatomy

Every blueprint lives in `blueprints/<blueprint-name>/` and contains:

| File | What it holds |
|---|---|
| `README.md` | One paragraph: what this shape is, and its status. |
| `spec.md` | **The blueprint itself** — the shape: architecture, the contracts/seams, the catalog of known adapters + the open seam, the composed doctrines, and a **conformance checklist**. |
| `commentary.md` | A **living validation logbook** — the record of proving the blueprint on a real instance: where it held, where it leaked, what was learned. Each claim tagged `[V]` (held on a real build) / `[D]` (documented) / `[A]` (assumed, untested). This is how the shape *earns* correctness rather than asserting it. |
| `reference/` (optional) | Pseudo-code, adapter catalogs, example realizations — aids, never normative. |

The **`commentary.md` is not optional decoration.** A blueprint extracted from one instance is a
hypothesis until a *second* instance proves it. The commentary is where that proof (or its failure)
is recorded honestly — an untested blueprint claim is `[A]`, not fact.

---

## 4. A blueprint COMPOSES doctrine — it does not duplicate it

The [CommonMind](../CommonMind/) family already states cross-project principles (how to
keep the record trustworthy, how a service binds, how confidence is scored, how devices are modeled,
and so on). A blueprint **references** the doctrines a conformant product must follow and adds **only
what is specific to the shape.**

Restating a doctrine inside a blueprint is a defect: it creates a second copy that drifts. If a
blueprint needs a rule that turns out to be portable beyond the shape, that rule belongs in
CommonMind (and the blueprint cites it), not inline.

---

## 5. This charter is deliberately anemic

The charter defines the **container** — the anatomy, the folder convention, compose-don't-duplicate —
and **says nothing about what a blueprint's *domain* must be.** A blueprint is not assumed to be a
service, or hardware-related, or anything in particular. The first blueprint happens to be a hardware
controller service; the next may be an entirely different genre.

This is deliberate, and it is the same discipline the [Device-Model
Doctrine](../CommonMind/device-model-doctrine.md) applies to hardware: **do not over-abstract
from one instance.** With one blueprint in hand, any "all blueprints are X" rule would be a guess.
The charter therefore constrains only the shape of the folder, not the nature of what fills it. When
a second, different blueprint exists and a genuine cross-blueprint pattern emerges, *then* it can be
stated — grounded in two instances, not one.
