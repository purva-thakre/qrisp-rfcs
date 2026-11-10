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

In-place addition is the single primitive underlying all arithmetic in Qrisp. Subtraction is performed by negating before adding; multiplication reduces to repeated addition; comparisons are implemented as a subtraction followed by a sign check; and Montgomery reduction decomposes into a series of additions and shifts. Any out-of-place arithmetic operations store the result of the in-place adder in a new `QuantumVariable`. In short, the in-place adders are the foundation on which the entire arithmetic stack rests.

Today, if you want to use a specific adder for your computation, you have to know which function to import, what arguments it expects, and whether it works in your execution mode (static vs. dynamic). If you want to swap one adder for another, you have to check that the replacement accepts the same input types and handles carry/control qubits the same way. There is no common interface that lets you treat adders interchangeably.

This also makes it hard to introduce a custom adder. Qrisp has tooling for amending adders (so they handle mixed input types like integers and generic quantum variables) and for verifying the custom adder works as expected by running a standard correctness test suite. But this tooling is easy to miss. It lives in separate files, uses a different calling convention than the adders themselves, and is not referenced anywhere. A user who implements a new in-place adder in Qrisp has to read the source to find out that `ammend_inpl_adder` and `inpl_adder_test` exist, call them in the right order, and work around their respective quirks. Note that both functions require significant refactoring on their own, which this RFC also addresses.  While these functions are not named with a leading underscore (which would signal they're private), they should not be treated as part of the public API.

Wrapping each adder in a `QuantumAdder` class gives them a shared type, puts the tooling where users can find it, and lets any arithmetic function accept an adder as a parameter and pass it through to the operations underneath

## Background and current architecture
[background-and-current-architecture]: #background-and-current-architecture

> [!NOTE]
> Whenever a generic `QuantumVariable` is described as an input to an in-place adder, it refers only to the subtypes that allow a numerical value to be encoded in them. Thus, this document excludes `QuantumChar` and `QuantumString` from the discussion.

Every adder does the same fundamental thing: $b \mathrel{+}= a$ on generic `QuantumVariable` objects in static and/or dynamic mode. The first operand `a` can be either a classical integer or a `QuantumVariable`. The second operand `b` is always a `QuantumVariable` and is where the result is stored such that the addition happens in place. Beyond the two operands, an adder may accept a carry-in qubit (an incoming carry from a previous addition), a carry-out qubit (to capture overflow), and a control qubit (so the addition only happens when the control `QuantumVariable` is in state `1`). The adders differ in how they compute and propagate carries through the register, and in which of these optional inputs they support.

But, as stated previously, they are all inconsistently defined in Qrisp. This section provides a description of how each in-place adder works and where it stands in terms of moving towards a unified interface. The Cuccaro and Gidney adders were refactored recently and now serve as the reference design that the other adders are being brought in line with. The Thapliyal adder, Remaud adder, and Fourier adder all have open PRs that refactor them to match the Cuccaro/Gidney interface.


| Adder | How it adds | Depth | Ancillas | Carry-in | Carry-out | Control | Modes | Refactored? |
|-------|------------|-------|----------|:--------:|:---------:|:-------:|-------|-------------|
| `cuccaro_adder` | MAJ/UMA ripple carry | $O(n)$ | 1 | ✅ | ✅ | ✅ | Both | ✅ |
| `gidney_adder` | AND-based ripple carry | $O(n)$ | $O(n)$ | ✅ | ✅ | ✅ | Both | ✅ |
| `gidney_cq_venting` | Measured carry chain | $O(n)$ | 3 (streaming) | ✅ | ✅ | ✅ | JASP only | ✅ |
| `fourier_adder` | QFT + phase rotations | $O(n^2)$ gates | 0 | ❌* | ❌* | ❌* | Both | 🚧 |
| `remaud_adder` | Ladder synthesis | $O(\log^2 n)$ | 0 | ❌* | ✅ | ❌* | Both | 🚧 |
| `qcla` | Brent-Kung carry tree | $O(\log n)$ | $O(n)$ | ❌ | ❌ | ✅ | Both | Removing |
| `thapliyal_adder` | TR-gate ripple carry | $O(n)$ | 1 | ❌* | ❌* | ❌* | Static only (old) | 🚧 |

**Legend:**

- Carry/Control columns
    - ✅ supported today
    - ❌ not supported
    - ❌* not supported yet, planned as part of the refactor
- Refactored? column
    - ✅ done
    - 🚧 in progress
    - `Removing` no refactor

To summarize, they all do $b \mathrel{+}= a$, but there's no way to write code that says "give me any adder" without knowing which specific one you're dealing with.

### Cuccaro Adder

Cuccaro is a ripple-carry adder based on [arXiv:quant-ph/0410184](https://arxiv.org/abs/quant-ph/0410184).

The adder walks through the register one bit at a time, from least significant to most significant. At each position it applies a MAJ (majority) gate sequence, which computes whether the addition at that position produces a carry. The carry feeds into the next position. After all carries are computed, the adder walks back through the register applying UMA (unmajority-and-add) gate sequences, which compute the final sum bits and clean up the carry ancillas.

Cuccaro is the simplest adder in Qrisp: linear depth (proportional to the number of qubits), one ancilla qubit for the carry chain. Input `a` can be an integer, a binary string, a `BigInteger`, or a `QuantumVariable`. Input `b` must be a `QuantumVariable`. When `a` is classical, the Qrisp Cuccaro adder function encodes it into a temporary quantum variable internally.

It has the `@custom_control` decorator, supports carry-in, carry-out, and control qubits, and works in both static and dynamic (JASP) modes. This is the reference implementation that other adders are being refactored to match.

### Gidney Adder

Gidney is an AND-based adder from [arXiv:1709.06648](https://arxiv.org/abs/1709.06648). Same signature and same input flexibility as Cuccaro.

Like Cuccaro, this is a ripple-carry adder that walks through the register bit by bit. The difference is in how it computes carries. Instead of MAJ/UMA gates, it uses AND gates (a type of Toffoli gate that can be uncomputed with a single measurement). The AND gate computes whether two bits are both 1. By chaining these together with CNOT gates, the adder builds up the carry chain.

The big advantage shows up when `a` is classical: the adder uses a semi-classical carry chain that never encodes `a` into qubits at all. It just applies controlled gates based on the bits of the classical value. This skips the step to define a temporary register that Cuccaro needs, saving qubits and gates.

The Gidney adder has same depth and ancilla cost as the Cuccaro adder for quantum-quantum addition. For classical-quantum addition it's more efficient because it avoids the temporary encoding.

The adder has `@custom_control`, full carry support, and works in both modes.

### Fourier Adder

Fourier is a Draper-style adder that works in the Fourier basis, based on [arXiv:quant-ph/0008033](https://arxiv.org/abs/quant-ph/0008033).

Instead of computing carries like the other adders, this one transforms `b` into the Fourier basis using a Quantum Fourier Transform (QFT). In the Fourier basis, addition becomes a series of phase rotations. Each qubit of `a` adds a small phase to each qubit of `b`, and the phases add up to the correct sum. Then the inverse QFT transforms `b` back to the computational basis. No ancillas needed, but the number of phase gates is quadratic (every qubit of `a` interacts with every qubit of `b`).

The `perform_QFT` flag lets you skip the QFT/inverse-QFT when you're chaining multiple additions. You apply the QFT once at the start, do all your additions in the Fourier basis, and apply the inverse QFT once at the end.

Currently `a` can be an integer, a `QuantumVariable`, or a list of qubits. Same for `b`. The signature is different from Cuccaro/Gidney: no carry-in, carry-out, or control parameters, no `@custom_control` decorator.

> [!IMPORTANT]
> Work is in-progress to consolidate the separately defined static and dynamic versions of the Fourier adder into a single function compatible in both modes. This refactoring also works to bring the Fourier adder signature in-line with the Cuccaro adder. 

### Remaud Adder

Remaud is an ancilla-free adder with polylogarithmic depth, based on [arXiv:2501.16802](https://arxiv.org/abs/2501.16802).

The other adders process carries one after another (linear depth) or use extra qubits to compute them in parallel (QCLA). Remaud takes a different approach: it uses a "ladder synthesis" technique that computes the sum bits through two stages of CNOT ladders, without any ancilla qubits at all. The first ladder propagates information from the least significant bits upward, and the second ladder propagates it back down to compute the final sum. The result is $O(\log^2 n)$ depth with zero ancillas, at the cost of a higher gate count.

`a` and `b` must be the same size. `z` is a single qubit for carry overflow. The signature is completely different from Cuccaro/Gidney: no decorator, no carry-in/carry-out/control parameters. Instead it takes an explicit carry qubit as a required third argument.

> [!IMPORTANT]
> Work is in progress to refactor this adder and bring it in-line with the Cuccaro adder.

### QCLA Adder

In a ripple-carry adder, each carry depends on the one before it, so the computation is sequential. QCLA computes all the carries at once using a tree structure called a Brent-Kung tree. First, it computes "propagate" and "generate" signals for each pair of input bits (does this pair produce a carry? does it pass one through?). Then it combines these signals up the tree in $O(\log n)$ steps, so the final carry at every position is known in parallel. Finally, it uses the carries to compute the sum bits. The tradeoff is that the tree needs $O(n)$ ancilla qubits.

a can be a QuantumFloat, a list of qubits, or an integer. b can be a QuantumFloat or a list of qubits. Internally it dispatches to cq_qcla for classical a or qq_qcla for quantum a.

No decorator. Has ctrl but no carry-in or carry-out. The radix_base, radix_exponent, and t_depth_reduction parameters control the tree structure and are specific to this algorithm.

> [!CAUTION]
> Planned for deprecation and removal. 

### Thapliyal Adder

Similar to Cuccaro, this walks through the register bit by bit. But instead of MAJ/UMA gates, it uses TR gates (a variant of the Toffoli gate built from rotation and phase gates). The TR gate computes a carry and a partial sum in one step. The carry ripples forward through the chain, and on the way back the partial sums are completed. Linear depth, similar ancilla cost to Cuccaro.

Currently this exists as thapliyal_procedure, a low-level function that takes a QuantumCircuit object and raw qubit lists. It only works in static mode. inpl_add (the legacy dispatcher in ripple_carry_adder.py) calls it internally.

> [!IMPORTANT]
> Work is in progress to refactor this adder and bring it in-line with the Cuccaro adder.

### Gidney Venting Adder

Gidney Venting is a carry-venting classical-quantum adder that uses mid-circuit measurements to save qubits.

Like the regular Gidney adder, this builds a semi-classical carry chain. The trick is that after computing each carry, the adder measures it in the X basis (apply H, then measure). This "vents" the qubit: the measurement collapses it, freeing it for reuse. The measurement outcome is tracked classically, and later corrections are applied based on the accumulated measurement results. This trades qubit count for measurement count: only 3 clean ancillae are needed, two alternating as the streaming carry window and one for scratch, regardless of register size.

The catch: `a` must be a classical integer or `BigInteger`. This is a classical-quantum adder only, it can't add two quantum registers. And it only works in JASP mode, because static mode doesn't have mid-circuit measurements. For small registers (under 6 qubits), the overhead isn't worth it, so it falls back to the Gidney adder function.

Has `@custom_control` and full carry support.

### Adder Tooling

The low-level helpers that were duplicated across adder files have been moved into a shared `adder_utilities.py` module (PR #990). This includes shared input validation (`_validate_adder_inputs`), type rejection for non-numeric quantum types (`_reject_unsupported_quantum_types`), optional qubit unwrapping (`_normalize_optional_input_arg`), binary string parsing, list-to-DynamicQubitArray conversion, bit extraction, and shared type aliases (`QuantumRegister`, `OptionalQubit`).

This was a necessary first step. But the higher-level tooling (amending, testing, dispatching) is still scattered, and none of the adders have a shared type.

**`ammend_inpl_adder(raw_inpl_adder, ammend_cl_int=True)`** in `adder_tools.py` takes a raw adder function and wraps it so it can handle mixed input types: padding shorter quantum registers, truncating longer ones, and encoding classical integers into temporary quantum variables. Issues: the argument order is reversed (`(value_to_add, target)` vs the raw adder's `(a, b)`), there's a large block of dead code (a QuantumFloat-to-QuantumFloat branch gated by `and False`), and there's no way to discover this function exists unless you read the source.

**`inpl_adder_test(inpl_adder)`** in `utility.py` runs a correctness test suite on an amended adder: quantum-quantum and classical-quantum addition, both uncontrolled and controlled, for register sizes 1-6. It also checks for spurious phase introduction. The problem is that it lives in a 2000+ line utility grab bag, nowhere near the adder code.

**`inpl_add(qf1, qf2, ..., adder="thapliyal")`** in `ripple_carry_adder.py` is the QuantumFloat-level addition dispatcher. It handles exponent alignment, sign handling, and ancilla management, then calls an adder for the actual bit-level addition. Despite having an `adder` parameter, it only accepts `"thapliyal"`. The one place that handles QuantumFloat alignment is hardcoded to one adder.



## Proposed design
[proposed-design]: #proposed-design



## Unresolved questions
[unresolved-questions]: #unresolved-questions

- TODO

## Future possibilities
[future-possibilities]: #future-possibilities

TODO

## Public API impact
[public-api-impact]: #public-api-impact

TODO: Describe additions, changes, or removals in Qrisp's public API, with migration guidance. Write "None" if this is internal only.
