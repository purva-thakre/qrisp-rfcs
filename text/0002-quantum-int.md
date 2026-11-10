# RFC 002: Split the `QuantumFloat` class into `QuantumInt` and `QuantumFloat`

- Feature Name: `QuantumInt` Class
- Start Date: 2026-11-10
- RFC PR: (leave empty until the pull request is opened)
- Qrisp Issue: (link the tracking issue, if one exists)

## Summary
[summary]: #summary

`QuantumFloat` currently serves double duty: it is the integer type when `exponent=0`
and a fixed-point type otherwise. There is no dedicated integer class. `QuantumModulus`
inherits from `QuantumFloat` directly, which forces it to suppress float-specific methods
that would produce incorrect results on modular integers.

This RFC introduces `QuantumInt` as a subclass of `QuantumFloat` with `exponent` fixed
to 0, and re-parents `QuantumModulus` under `QuantumInt`. The type hierarchy becomes:

```python
    QuantumVariable → QuantumFloat → QuantumInt → QuantumModulus
```

This decouples integer arithmetic from floating-point arithmetic, gives modular types a
clean base that doesn't carry float-only operations, and lets adders and other integer
infrastructure target `QuantumInt` without pulling in exponent alignment or precision
concerns that only matter for floats.

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
