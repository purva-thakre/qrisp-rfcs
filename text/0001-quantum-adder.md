# RFC 001: Introduce an in-place adder class

- Feature Name: `QuantumAdder` Class
- Start Date: 2026-11-05
- RFC PR: https://github.com/purva-thakre/qrisp-rfcs/pull/2
- Qrisp Issue: N/A

## Summary
[summary]: #summary

Qrisp has seven in-place adders. They all do the same thing (add a value stored from one input into another), but each one has a different signature, different optional parameters, and different input handling. There is no shared type, no shared validation, and no way to check at runtime whether a given function is actually a valid in-place adder. The tooling that exists (amending adders to handle mixed inputs, verifying correctness) is split across separate files, and a user who provides a custom adder has to discover each piece on their own.

This RFC walks through every in-place adder in Qrisp and the tooling around them, then proposes a thin wrapper `QuantumAdder` class that gives them a standard interface without changing how any of them works.

## Motivation
[motivation]: #motivation

In-place addition is the single primitive underlying all arithmetic in Qrisp. Subtraction is performed by negating before adding; multiplication reduces to repeated addition; comparisons are implemented as a subtraction followed by a sign check; and Montgomery reduction decomposes into a series of additions and shifts. In short, the in-place adder is the foundation on which the entire arithmetic stack rests.

Today, if you want to use a specific adder for your computation, you have to know which function to import, what arguments it expects, and whether it works in your execution mode (static vs. dynamic). If you want to swap one adder for another, you have to check that the replacement accepts the same input types and handles carry/control qubits the same way. There is no common interface that lets you treat adders interchangeably.

This also makes it hard to introduce a custom adder. Qrisp has tooling for amending adders (so they handle mixed input types like integers and generic quantum variables) and for verifying the custom adder works as expected by running a standard correctness test suite. But this tooling is easy to miss. It lives in separate files, uses a different calling convention than the adders themselves, and is not referenced anywhere. A user who implements a new in-place adder in Qrisp has to read the source to find out that `ammend_inpl_adder` and `inpl_adder_test` exist, call them in the right order, and work around their respective quirks. Note that both functions require significant refactoring on their own, which this RFC also addresses.  While these functions are not named with a leading underscore (which would signal they're private), they should not be treated as part of the public API.

Wrapping each adder in a `QuantumAdder` class gives them a shared type, puts the tooling where users can find it, and lets any arithmetic function accept an adder as a parameter and pass it through to the operations underneath

## Background and current architecture
[background-and-current-architecture]: #background-and-current-architecture

Every adder does the same fundamental thing: `b += a` on generic `QuantumVariable` objects in static and/or dynamic mode. The first operand `a` can be either a classical integer or a quantum register. The second operand `b` is always a quantum register and is where the result is stored -- the addition happens in place. Beyond the two operands, an adder may accept a carry-in qubit (an incoming carry from a previous addition), a carry-out qubit (to capture overflow), and a control qubit (so the addition only happens when the control is in state `1`). The adders differ in how they compute and propagate carries through the register, and in which of these optional inputs they support.

> [!NOTE]
> Whenever a generic `QuantumVariable` is described as an input to an in-place adder, it refers only to the subtypes that allow a numerical value to be encoded in them. Thus, this document excludes `QuantumChar` and `QuantumString` from the discussion.

But, as stated previously, they are all inconsistently defined in Qrisp. This section provides a description of how each in-place adder works and where it stands in terms of moving towards a unified interface. The Cuccaro and Gidney adders were refactored recently and now serve as the reference design that the other adders are being brought in line with. The Thapliyal adder, Remaud adder, and Fourier adder all have open PRs that refactor them to match the Cuccaro/Gidney interface.

### Cuccaro Adder

Cuccaro is a ripple-carry adder based on arXiv:quant-ph/0410184.

The adder walks through the register one bit at a time, from least significant to most significant. At each position it applies a MAJ (majority) gate sequence, which computes whether the addition at that position produces a carry. The carry feeds into the next position. After all carries are computed, the adder walks back through the register applying UMA (unmajority-and-add) gate sequences, which compute the final sum bits and clean up the carry ancillas.

Cuccaro is the simplest adder in Qrisp: linear depth (proportional to the number of qubits), one ancilla qubit for the carry chain. Input `a` can be an integer, a binary string, a `BigInteger`, or a `QuantumVariable`. Input `b` must be a `QuantumVariable`. When `a` is classical, the Qrisp Cuccaro adder function encodes it into a temporary quantum variable internally.

It has the `@custom_control` decorator, supports carry-in, carry-out, and control qubits, and works in both static and dynamic (JASP) modes. This is the reference implementation that other adders are being refactored to match.

### Gidney Adder

Gidney is an AND-based adder from arXiv:1709.06648. Same signature and same input flexibility as Cuccaro.

Like Cuccaro, this is a ripple-carry adder that walks through the register bit by bit. The difference is in how it computes carries. Instead of MAJ/UMA gates, it uses AND gates (a type of Toffoli gate that can be uncomputed with a single measurement). The AND gate computes whether two bits are both 1. By chaining these together with CNOT gates, the adder builds up the carry chain.

The big advantage shows up when `a` is classical: the adder uses a semi-classical carry chain that never encodes `a` into qubits at all. It just applies controlled gates based on the bits of the classical value. This skips the step to define a temporary register that Cuccaro needs, saving qubits and gates.

The Gidney adder has same depth and ancilla cost as the Cuccaro adder for quantum-quantum addition. For classical-quantum addition it's more efficient because it avoids the temporary encoding.

The adder has `@custom_control`, full carry support, and works in both modes.

### Fourier Adder

Fourier is a Draper-style adder that works in the Fourier basis, based on arXiv:quant-ph/0008033.

Instead of computing carries like the other adders, this one transforms `b` into the Fourier basis using a Quantum Fourier Transform (QFT). In the Fourier basis, addition becomes a series of phase rotations. Each qubit of `a` adds a small phase to each qubit of `b`, and the phases add up to the correct sum. Then the inverse QFT transforms `b` back to the computational basis. No ancillas needed, but the number of phase gates is quadratic (every qubit of `a` interacts with every qubit of `b`).

The `perform_QFT` flag lets you skip the QFT/inverse-QFT when you're chaining multiple additions. You apply the QFT once at the start, do all your additions in the Fourier basis, and apply the inverse QFT once at the end.

Currently `a` can be an integer, a `QuantumVariable`, or a list of qubits. Same for `b`. The signature is different from Cuccaro/Gidney: no carry-in, carry-out, or control parameters, no `@custom_control` decorator.

> [!IMPORTANT]
> Work is in-progress to consolidate the separately defined static and dynamic versions of the Fourier adder into a single function compatible in both modes. This refactoring also works to bring the Fourier adder signature in-line with the Cuccaro adder. 

### Gidney Venting Adder

Gidney Venting is a carry-venting classical-quantum adder that uses mid-circuit measurements to save qubits.

Like the regular Gidney adder, this builds a semi-classical carry chain. The trick is that after computing each carry, the adder measures it in the X basis (apply H, then measure). This "vents" the qubit: the measurement collapses it, freeing it for reuse. The measurement outcome is tracked classically, and later corrections are applied based on the accumulated measurement results. This trades qubit count for measurement count: only 3 clean ancillae are needed, two alternating as the streaming carry window and one for scratch, regardless of register size.

The catch: `a` must be a classical integer or `BigInteger`. This is a classical-quantum adder only, it can't add two quantum registers. And it only works in JASP mode, because static mode doesn't have mid-circuit measurements. For small registers (under 6 qubits), the overhead isn't worth it, so it falls back to the Gidney adder function.

Has `@custom_control` and full carry support.



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
