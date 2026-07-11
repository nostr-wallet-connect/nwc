# Contributing

NWC specs in this repository are meant to be small, focused, and optional.
The goal is to document useful behavior that can be implemented on top of the
core Nostr Wallet Connect protocol without turning the core spec into a large
grab bag of features.

This process is intentionally lightweight. The goal is clarity, not process for
its own sake.

---

## What belongs here

A spec in this repository should generally be:

1. Optional relative to NWC core.
2. Narrow in scope.
3. Useful for real wallet or app interoperability.
4. Backward-compatible with implementations that do not support it.

If something should be required for all NWC implementations, it probably
belongs in the core spec, not here.

---

## Before opening a PR

For a new spec or major new feature:

1. Open an issue first.
2. Describe the problem being solved.
3. Explain why it should be a separate optional spec instead of part of core.
4. Mention any existing implementations, experiments, or prior discussion if
   they exist.

Small clarifications, typo fixes, and editorial improvements can go directly to
a PR.

---

## Writing a spec

Keep specs simple and practical.

- Focus on one feature or one closely related set of behaviors.
- Reuse existing NWC terms and message shapes where possible.
- Prefer one clear way to do something.
- Use `MUST`, `SHOULD`, and `MAY` when normative behavior matters.
- Include request, response, event, or notification examples when relevant.
- Explicitly reference other specs when there is a dependency.

Avoid adding optionality inside optional specs unless it is clearly needed.

---

## File naming

- Use `XX.md` as a placeholder in PRs for a new spec.
- Do not self-assign the final number.
- Maintainers will assign or confirm numbering when merging.

If a new spec depends on an existing one, reference that dependency explicitly
in the document.

---

## Review criteria

A spec should usually meet these criteria before merge:

1. The problem is real and specific.
2. The design is small enough to stand on its own.
3. It does not duplicate an existing spec.
4. It fits the direction of NWC as a wallet interoperability layer.
5. It is written clearly enough that independent implementations can follow it.

Production deployment is useful evidence, but it is not required for every PR.
Early specs may be merged to support experimentation if the scope is clear and
the design is reasonable.

---

## Review and merge

- Maintainers make the final merge decision.
- Technical objections should be addressed before merge.
- Numbering and file renames happen at merge time when needed.
- Specs may evolve through follow-up PRs as implementations improve.

---

## Editing existing specs

Clarifications are encouraged, but avoid changing behavior casually.

If you are changing semantics rather than wording:

1. Explain the compatibility impact clearly.
2. Say whether the change is tightening, relaxing, or extending behavior.
3. Prefer a new optional spec over making an existing spec broader when that
   keeps things simpler.
