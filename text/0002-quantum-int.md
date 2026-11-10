# RFC 002: Split the `QuantumFloat` class into `QuantumInt` and `QuantumFloat`

- Feature Name: `QuantumInt` Class
- Start Date: 2026-11-10
- RFC PR: (leave empty until the pull request is opened)
- Qrisp Issue: (link the tracking issue, if one exists)

> [!WARNING]
> This RFC intentionally leaves two related naming questions open: whether `QuantumFloat` should be renamed to better describe fixed-point values, and whether `QuantumModulus` should be renamed to `QuantumModular`. Both are worthwhile proposals, but they change the public API and deserve broader community feedback and a formal vote. Addressing them here would hold up the refactoring, so they are deferred to a separate discussion. Nothing in this RFC rules out revisiting either name later.

## Summary
[summary]: #summary

`QuantumFloat` currently serves double duty: it is the integer type when `exponent=0` and a fixed-point type otherwise. There is no dedicated integer class. `QuantumModulus` inherits from `QuantumFloat` directly, which means it has access to float-specific methods that produce incorrect results on modular integers. Some of these are suppressed, but not all of them, and there is no systematic way to catch the ones that slip through. On top of this, every arithmetic dunder method has `check_for_tracing_mode()` branches that mix integer/float semantics with static/dynamic dispatch.

This RFC introduces `QuantumInt` as a subclass of `QuantumFloat` with `exponent` fixed to 0, and re-parents `QuantumModulus` under `QuantumInt`. The type hierarchy becomes:

```text
QuantumVariable → QuantumFloat → QuantumInt → QuantumModulus
```

By separating integer arithmetic from floating-point arithmetic, modular types gain a clean base type that carries no float-only operations. The integer/float boundary becomes explicit in the type hierarchy rather than something each subclass has to enforce manually by suppression. Once the adders work in both modes (RFC 001) and SBP arithmetic module is deprecated (RFC 003), the `check_for_tracing_mode()` branches in the arithmetic methods can be removed entirely. `QuantumFloat`'s arithmetic methods become straightforward: align exponents, then call the adder. `QuantumInt` inherits the same methods, but since its exponent is always 0, the alignment step does nothing and the methods reduce to plain adder calls.

## Motivation
[motivation]: #motivation



## Background and current architecture
[background-and-current-architecture]: #background-and-current-architecture

TODO: Describe the current addition code paths, the duplication or inconsistency they cause, and the constraints the design must account for.

## Proposed design
[proposed-design]: #proposed-design

TODO: Define the interface, how implementations are selected, and how it lowers to circuits. Include examples and corner cases.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- TODO

## Future possibilities
[future-possibilities]: #future-possibilities

TODO

## Public API impact
[public-api-impact]: #public-api-impact

TODO: Describe additions, changes, or removals in Qrisp's public API, with migration guidance. Write "None" if this is internal only.
