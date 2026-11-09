# RFC 001: Introduce a `QuantumAdder` Class

- Feature Name: `QuantumAdder` Class
- Start Date: 2026-11-05
- RFC PR: https://github.com/purva-thakre/qrisp-rfcs/pull/2
- Qrisp Issue: N/A

## Summary
[summary]: #summary

Qrisp has seven in-place adders. They all do the same thing (add value stored in one input into another), but each one has a different signature, different optional parameters, and different input handling. There is no shared type, no shared validation, and no way to check at runtime whether a given function is actually a valid in-place adder. The tooling that exists (amending adders to handle mixed inputs, verifying correctness) is split across separate files, and a user who provides a custom adder has to discover each piece on their own.

This RFC walks through every in-place adder in Qrisp and the tooling around them, then proposes a thin wrapper `QuantumAdder` class that gives them a standard interface without changing how any of them works.

## Motivation
[motivation]: #motivation

In-place addition is the single primitive underlying all arithmetic in Qrisp. Subtraction is performed by negating before adding; multiplication reduces to repeated addition; comparisons are implemented as a subtraction followed by a sign check; and Montgomery reduction decomposes into a series of additions and shifts. In short, the in-place adder is the foundation on which the entire arithmetic stack rests.

Today, if you want to use a specific adder for your computation, you have to know which function to import, what arguments it expects, and whether it works in your execution mode (static vs.dynamic). If you want to swap one adder for another, you have to check that the replacement accepts the same input types and handles carry/control qubits the same way. There is no common interface that lets you treat adders interchangeably.

This also makes it hard to write a custom adder. Qrisp has tooling for amending adders (so they handle mixed input types like integers and quantum registers) and for verifying the custom adder works as expected (running a standard correctness test suite). But this tooling is easy to miss. It lives in separate files, uses a different calling convention than the adders themselves, and is not referenced anywhere. Although these functions were originally defined without the leading underscore, they are not yet part of the public API.

A user who implements a new adder algorithm has to read the source to find out that `ammend_inpl_adder` and `inpl_adder_test` exist, call them in the right order, and work around their respective quirks. Note that both functions need significant refactoring on their own, which this RFC also addresses.

Wrapping each adder in a `QuantumAdder` class gives them a shared type, puts the tooling where users can find it, and lets any arithmetic function accept an adder as a parameter and pass it through to the operations underneath.

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
