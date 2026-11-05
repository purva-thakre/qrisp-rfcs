# RFC 001: Introduce a `QuantumAdder` Class

- Feature Name: `QuantumAdder` Class
- Start Date: 2026-11-05
- RFC PR: https://github.com/purva-thakre/qrisp-rfcs/pull/2
- Qrisp Issue: N/A

## Summary
[summary]: #summary

Introduce a `QuantumAdder` abstraction that provides a single interface for adding quantum values, so arithmetic code can depend on an adder implementation rather than a concrete addition routine. This RFC is self-contained and uncontroversial.

## Motivation
[motivation]: #motivation

TODO: What problem does the adder abstraction solve, who is affected, and why is it worth doing now?

## Background and current architecture
[background-and-current-architecture]: #background-and-current-architecture

TODO: Describe the current addition code paths, the duplication or inconsistency they cause, and the constraints the design must account for.

## Proposed design
[proposed-design]: #proposed-design

TODO: Define the interface, how implementations are selected, and how it lowers to circuits. Include examples and corner cases.

### Test example

```python
from qrisp import QuantumFloat
from qrisp.arithmetic import QuantumAdder

a = QuantumFloat(4)
b = QuantumFloat(4)
a[:] = 3
b[:] = 4

QuantumAdder(a, b).apply()

assert b.get_measurement() == {7: 1.0}
```

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- TODO

## Future possibilities
[future-possibilities]: #future-possibilities

TODO

## Public API impact
[public-api-impact]: #public-api-impact

TODO: Describe additions, changes, or removals in Qrisp's public API, with migration guidance. Write "None" if this is internal only.
