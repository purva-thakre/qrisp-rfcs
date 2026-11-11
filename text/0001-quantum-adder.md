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

Every adder does the same fundamental thing: $b \mathrel{+}= a$ on generic `QuantumVariable` objects in static and/or dynamic mode. The first operand `a` can be either a classical integer, a `BigInteger` or a `QuantumVariable`. The second operand `b` is always a `QuantumVariable` and is where the result is stored such that the addition happens in place. Beyond the two operands, an adder may accept a carry-in qubit (an incoming carry from a previous addition), a carry-out qubit (to capture overflow), and a control qubit (so the addition only happens when the control `QuantumVariable` is in state `1`). The adders differ in how they compute and propagate carries through the qubit chain, and in which of these optional inputs they support.

But, as stated previously, they are all inconsistently defined in Qrisp. This section provides a description of how each in-place adder works and where it stands in terms of moving towards a unified interface. The Cuccaro and Gidney adders were refactored recently and now serve as the reference design that the other adders are being brought in line with (except the venting based Gidney adder). The Thapliyal adder, Remaud adder, and Fourier adder all have open PRs that refactor them to match the Cuccaro/Gidney interface.


| Adder | How it adds | Depth | Ancillas | Carry-in | Carry-out | Control | Modes | Refactored? |
|-------|------------|-------|----------|:--------:|:---------:|:-------:|-------|-------------|
| `cuccaro_adder` | MAJ/UMA ripple carry | $O(n)$ | 1 | ✅ | ✅ | ✅ | Both | ✅ |
| `gidney_adder` | AND-based ripple carry | $O(n)$ | $O(n)$ | ✅ | ✅ | ✅ | Both | ✅ |
| `gidney_cq_venting` | Measured carry chain | $O(n)$ | 3 clean | ✅ | ✅ | ✅ | JASP only | ✅ |
| `fourier_adder` | QFT + phase rotations | $O(n)$ parallel or $O(n^2)$| 0 | ❌* | ❌* | ❌* | Both | 🚧 |
| `remaud_adder` | Log-depth ripple carry | $O(\log^2 n)$ | 0 | ❌* | ✅ | ❌* | Both | 🚧 |
| `qcla` | Brent-Kung carry tree | $O(\log n)$ T-depth | $O(n)$ | ❌ | ❌ | ✅ | Both | Removing |
| `thapliyal_adder` | TR-gate ripple carry | $O(n)$ | 1† | ❌* | ❌* | ❌* | Static only (old) | 🚧 |

**Legend:**

- Carry/Control columns
    - ✅ supported today
    - ❌ not supported
    - ❌* not supported yet, planned as part of the refactor
- Ancillas column
    - † The Qrisp implementation of the Thapliyal adder allocates one output qubit for the carry; the paper claims 0 ancilla inputs
- Refactored? column
    - ✅ done
    - 🚧 in progress
    - `Removing` no refactor

To summarize, they all do $b \mathrel{+}= a$, but there's no way to write code that says "give me any adder" without knowing which specific one you're dealing with.

### Cuccaro Adder

Cuccaro is a ripple-carry adder based on [arXiv:quant-ph/0410184](https://arxiv.org/abs/quant-ph/0410184).

The adder walks through the register one bit at a time, from least significant to most significant. At each position it applies a MAJ (majority) gate sequence, which computes whether the addition at that position produces a carry. The carry feeds into the next position. After all carries are computed, the adder walks back through the register applying UMA (unmajority-and-add) gate sequences, which compute the final sum bits and clean up the carry ancillas.

Cuccaro is the simplest adder in Qrisp: linear depth (proportional to the number of qubits), one ancilla qubit for the carry chain. Input `a` can be an integer, a binary string, a `BigInteger`, or a `QuantumVariable`. Input `b` must be a `QuantumVariable`. When `a` is classical, the Qrisp Cuccaro adder function encodes the value into a temporary quantum variable internally.

The paper reports a depth of $2n + O(1)$, using $2n + O(1)$ Toffoli gates, $5n + O(1)$ CNOT gates, and $2n + O(1)$ negations, all with a single carry ancilla. As discussed in the Gidney adder paper, implementing these Toffolis with the standard construction gives a T-count of $8n + O(1)$.

It has the `@custom_control` decorator, supports carry-in, carry-out, and control qubits, and works in both static and dynamic (JASP) modes. This is the reference implementation that other adders are being refactored to match.

### Gidney Adder

Gidney is an AND-based adder from [arXiv:1709.06648](https://arxiv.org/abs/1709.06648). Same signature and same input flexibility as Cuccaro.

Like Cuccaro, this is a ripple-carry adder that walks through the register bit by bit. The difference is in how it computes carries. Instead of MAJ/UMA gates, it uses AND gates (a type of Toffoli gate that can be uncomputed with a single measurement). The AND gate computes whether two bits are both 1. By chaining these together with CNOT gates, the adder builds up the carry chain.

The big advantage shows up when `a` is classical. The paper describes only quantum-quantum addition, but the Qrisp implementation adds a semi-classical carry chain that never encodes `a` into qubits at all. It applies controlled gates based on the bits of the classical value. This skips the temporary register that Cuccaro needs, saving qubits and gates.

The Gidney adder has the same $O(n)$ depth as the Cuccaro adder for quantum-quantum addition, but it uses $O(n)$ carry ancillae (the carry chain keeps one ancilla per bit) rather than Cuccaro's single ancilla. For classical-quantum addition it's more efficient because it avoids the temporary encoding. In detail, the paper's $n$-bit adder has a T-count of $4n + O(1)$, a measurement depth of $2n + O(1)$, and requires $n$ ancillae. Each temporary logical-AND costs 4 T gates and has measurement depth 2, and its uncomputation costs no T gates.

The Qrisp Gidney adder implementation has `@custom_control`, full carry support, and works in both modes.

### Fourier Adder

Fourier is a Draper-style adder that works in the Fourier basis, based on [arXiv:quant-ph/0008033](https://arxiv.org/abs/quant-ph/0008033).

Instead of computing carries like the other adders, this one transforms `b` into the Fourier basis using a Quantum Fourier Transform (QFT). In the Fourier basis, addition becomes a series of phase rotations. Each qubit of `a` adds a small phase to each qubit of `b`, and the phases add up to the correct sum. Then the inverse QFT transforms `b` back to the computational basis. No ancillas needed as the construction does not require temporary carry bits, but the number of phase gates is quadratic (every qubit of `a` interacts with every qubit of `b`).

The exact QFT-based addition uses $O(n^2)$ gates. The paper notes that the approximate QFT (AQFT) reduces this to $O(n \log n)$ operations, and that with enough parallelism ($n/2$ simultaneous two-qubit gates) the circuit runs in about $n + 1$ time slices, or $O(\log n)$ time slices with the AQFT.

The `perform_QFT` flag in the Qrisp implementation lets you skip the QFT/inverse-QFT when you're chaining multiple additions. You apply the QFT once at the start, do all your additions in the Fourier basis, and apply the inverse QFT once at the end.

Currently `a` can be an integer, a `QuantumVariable`, or a list of qubits. Same for `b`. The signature is different from Cuccaro/Gidney: no carry-in, carry-out, or control parameters, no `@custom_control` decorator.

> [!IMPORTANT]
> Work is in-progress to consolidate the separately defined static and dynamic versions of the Fourier adder into a single function compatible in both modes. This refactoring also works to bring the Fourier adder signature in-line with the Cuccaro adder. 

### Remaud Adder

Remaud is an ancilla-free adder with polylogarithmic depth, based on [arXiv:2501.16802](https://arxiv.org/abs/2501.16802).

The other adders process carries one after another (linear depth) or use extra qubits to compute them in parallel (QCLA). Remaud takes a different approach: starting from the Takahashi ripple-carry circuit, it replaces the CNOT and Toffoli ladders that cause the linear depth with logarithmic-depth equivalents. The addition circuit has seven slices: parallel CNOT layers, CNOT ladders, and Toffoli ladders. Each CNOT ladder is decomposed into a recursive $O(\log n)$-depth circuit (Lemma 2) and each Toffoli ladder into an $O(\log^2 n)$-depth circuit (Lemma 4), all without ancilla qubits. The result is $O(\log^2 n)$ depth with zero ancillas, at the cost of an $O(n \log n)$ (linearithmic) gate count.

With the current Qrisp implementation, `a` and `b` must be the same size. `z` is a single qubit for carry overflow. The signature is completely different from Cuccaro/Gidney: no decorator, no carry-in/carry-out/control parameters. Instead it takes an explicit carry qubit as a required third argument.

> [!IMPORTANT]
> Work is in progress to refactor this adder and bring it in-line with the Cuccaro adder.

### QCLA Adder

QCLA is a carry-lookahead adder based on [arXiv:2304.02921](https://arxiv.org/abs/2304.02921) that uses a Brent-Kung tree to compute all carries in parallel, achieving $O(\log n)$ T-depth and $O(n)$ T gates at the cost of $O(n)$ ancilla qubits. It supports classical-quantum and quantum-quantum inputs, has a `ctrl` parameter but no carry-in or carry-out, and does not use the `@custom_control` decorator.

> [!CAUTION]
> Planned for deprecation and removal. 

### Thapliyal Adder

Similar to Cuccaro, this walks through the register bit by bit. But instead of MAJ/UMA gates, it uses TR gates, a distinct reversible gate built from one Controlled-V gate, one CNOT gate, and two Controlled-$V^\dagger$ gates (quantum cost 4), where V is the $\sqrt{X}$ (square root of NOT) gate and $V^\dagger$ is its inverse. The TR gate realizes the Boolean functions $A \cdot \bar{B} \oplus C$ and $A \oplus B$. The carry ripples forward through the chain, and on the way back the sum bits are completed. Linear depth, similar ancilla cost to Cuccaro.

The paper gives two ripple-carry designs, both using TR gates and both without ancilla input bits. The no-input-carry design (Section 4) has a quantum cost of $13n - 8$ and a propagation delay of $(11n - 4)\Delta$. The with-input-carry design (Section 5) has a quantum cost of $15n - 6$ and a delay of $(9n + 1)\Delta$. Qrisp uses the TR gate and allocates one output qubit for the carry; cost and depth remain linear in both cases.

Currently this exists as `thapliyal_procedure`, a low-level function that takes a `QuantumCircuit` object and raw qubit lists. It only works in static mode. `inpl_add` (the legacy dispatcher in `ripple_carry_adder.py`) calls it internally.

> [!IMPORTANT]
> Work is in progress to refactor this adder and bring it in-line with the Cuccaro adder.

### Gidney Venting Adder

Gidney Venting is a carry-venting classical-quantum adder that uses mid-circuit measurements to save qubits.

Like the regular Gidney adder, this builds a classical-quantum carry chain. The trick is that after computing each carry, the adder measures it in the X basis. This "vents" the qubit: the measurement collapses it, freeing it for reuse. The measurement outcome is tracked classically, and later corrections are applied based on the accumulated measurement results. This trades qubit count for measurement count: only 3 clean ancillae are needed regardless of register size.

The paper's constant-workspace construction uses 3 clean ancillae and $4n \pm O(1)$ Toffoli gates at linear ($O(n)$) depth. It also gives a lower-Toffoli variant with $3n \pm O(1)$ Toffolis, 2 clean ancillae, and $n - 2$ dirty ancillae. The Qrisp implementation uses the 3-clean-ancilla construction. It splits the target register in half: the bottom half is added using a clean carry-venting chain (Fig. 2 of the paper), and the top half is added using the paper’s dirty-ancilla construction (Fig. 4), which borrows the bottom-half target qubits as dirty workspace. The 3 clean ancillae provide the streaming carry window and mid-carry bridge throughout.

The catch: `a` must be a classical integer or `BigInteger`. This is a classical-quantum adder only, it can't add two quantum registers. And it only works in JASP mode, because static mode doesn't have mid-circuit measurements. For small registers (under 6 qubits), the overhead isn't worth it, so it falls back to the Gidney adder function.

Has `@custom_control` and full carry support.

### Adder Tooling

The low-level helpers that were duplicated across adder files have been moved into a shared `adder_utilities.py` module. This includes shared input validation (`_validate_adder_inputs`), type rejection for non-numeric quantum types (`_reject_unsupported_quantum_types`), optional qubit unwrapping (`_normalize_optional_input_arg`), binary string parsing, list-to-DynamicQubitArray conversion, bit extraction, and shared type aliases (`QuantumRegister`, `OptionalQubit`). This was a necessary first step. But the higher-level tooling (amending, testing, dispatching) is still scattered, and none of the adders have a shared type.

- `ammend_inpl_adder(raw_inpl_adder, ammend_cl_int=True)` in `adder_tools.py` takes a raw adder function and wraps it so it can handle mixed input types: padding shorter registers, truncating longer ones, and encoding classical integers into temporary quantum variables. This only works in static mode; in dynamic mode the wrapper does nothing and passes the arguments through unchanged. The function also has a reversed argument order compared to the raw adders, dead code that is never reached, and no way to discover it exists unless you read the source.

> [!WARNING]
> The existing function is spelled `ammend_inpl_adder` (double m). This will be corrected to `amend` as part of this refactoring. All references to `ammend` in this document use the current spelling for clarity, but the final implementation will use `amend` everywhere.

- `inpl_adder_test(inpl_adder)` in `utility.py` runs a correctness test suite on an amended adder: quantum-quantum and classical-quantum addition, both uncontrolled and controlled, for register sizes 1-6. It also checks for spurious phase introduction. The problem is that it is nowhere near the adder code and is also not compatible in dynamic mode.

- `inpl_add(qf1, qf2, ..., adder="thapliyal")` in `ripple_carry_adder.py` is the `QuantumFloat`-level addition dispatcher. It handles exponent alignment, sign handling, and ancilla management, then calls an adder for the actual bit-level addition. Despite having an `adder` parameter, it only accepts `"thapliyal"`. The one place that handles `QuantumFloat` alignment is hardcoded to one adder and works only in static mode.



## Proposed design
[proposed-design]: #proposed-design

Ideally, an in-place adder in Qrisp

* Should be compatible in both static and dynamic modes unless the adder relies on mid-circuit measurements with classically controlled corrections (for example, Gidney's venting adder), which require dynamic mode.
    * Note that these adders only work with integers. They don't need to worry about issues like overflow, exponent alignment, signs, or non-integer inputs (like a `QuantumFloat` with a non-zero exponent). Those are handled at a higher level when quantum types are dispatched (at the level of infix arithmetic). The adder documentation will state this clearly. For example, for operators like +, each type handles these issues itself before using the adder: `QuantumFloat.__iadd__` aligns exponents first, while `QuantumInt.__iadd__` passes values straight to the adder etc.
* Should follow the standard signature `(a, b, c_in=None, c_out=None, ctrl=None)` where possible. Here `a` and `b` are the two integer values to be added, `c_in` and `c_out` designate the optional carry in and carry out values.
    * The `c_in` parameter may be a `Qubit`, a `QuantumVariable` of size 1 (including `QuantumBool`), a single-element `list[Qubit]`, or a classical value of 0 or 1. The adder computes $b \leftarrow (a + b + c_{in}) \mod 2^{\text{len}(b)}$.
    * The `c_out` parameter may be a `Qubit`, a `QuantumVariable` of size 1 (including `QuantumBool`), or a single-element `list[Qubit]`. The adder computes $c_{out} \leftarrow c_{out} \oplus \text{carry}$, so that `c_out` holds the bit that would otherwise be discarded on wraparound.
    * Adders that do not natively support carry operations declare this in their metadata; for these, `.amend()` emulates the carry logic. In this emulation, `c_out` is treated as an extra most significant bit of `b`, and `c_in` is incorporated by means of a controlled increment.
* Should accept all types that allow an end user to encode an int or a float value into the quantum type: `QuantumVariable`, `QuantumFloat`, `QuantumInt` (see the next RFC where `QuantumFloat` is going to be split into `QuantumInt` and `QuantumFloat`), `QuantumModulus`, `QuantumBool`, `DynamicQubitArray`, and `list[Qubit]`.
    * The first argument can also be a classical value: `int`, `np.integer`, `str` (binary string), `BigInteger`, or a traced JAX scalar in dynamic mode.
    * Non-numeric quantum types `QuantumChar` and `QuantumString` should be rejected as either input.

    > [!NOTE]
    > `QuantumBool` is a special case as an input because its size is fixed at 1 qubit and cannot be set by the user, so it can only ever represent 0 or 1.

* Should have a custom controlled version (`@custom_control`) compatible in both modes.
* Should safely handle inputs of unequal sizes as the result always has to fit into the second input `b`.
    * If the sum is larger than `b` can hold, it wraps around (the result is the sum modulo $2^{\text{len}(b)}$).
    * When `a` is larger than `b`, `a` is truncated to match `b`'s size.
    * When `a` is smaller, `a` is padded with ancilla qubits that are only used during the addition and safely removed afterward.
    * The second input's size is never adjusted because ancilla qubits written during the addition cannot be safely removed.
* Should leave the first input `a` unchanged. 
* Should cleanup any ancillas that were allocated internally in an adder to compute the in-place addition.
* Should not leave behind any spurious phases on the output. 

### Class definition

The `QuantumAdder` class is intentionally thin. It wraps an existing adder function without changing how that function behaves when called. The wrapper just gives the function a type you can check against and puts the tooling (amending, verification) in a place where users can find it.

```python
import functools

class QuantumAdder:
    def __init__(self, func):
        self._raw = func
        functools.update_wrapper(self, func)

    def __call__(self, *args, **kwargs):
        return self._raw(*args, **kwargs)

    def ammend(self, ammend_cl_int=True):
        """Return an ammended version via ammend_inpl_adder."""
        return ammend_inpl_adder(self._raw, ammend_cl_int=ammend_cl_int)

    def verify(self):
        """Verify correctness via inpl_adder_test (ammends internally first)."""
        ammended = self.ammend()
        return inpl_adder_test(ammended)
```

When you call a `QuantumAdder` instance, it just calls the underlying function. Nothing about the adder's behavior changes at the call site. Wrapping an adder in QuantumAdder does not change how it behaves when called. For example, existing code that calls `cuccaro_adder(a, b)` keeps working exactly as before.

The `.ammend()` method wraps the raw adder through `ammend_inpl_adder` so it can handle mixed input types (different-sized registers, classical integers, etc.). The `.verify()` method ammends the adder first, then runs it through the standard `inpl_adder_test` correctness suite. This way a user who writes a custom adder doesn't have to know about either of those functions; they just call `.verify()` and get a pass/fail confirming whther their custom adder indeed works as expected.

Both `ammend_inpl_adder` and `inpl_adder_test` are being heavily refactored as part of this work. The current implementations are static-mode only, have reversed argument orders, and contain dead code. The refactored versions will work in both static and dynamic modes, follow the standard input argument order, and align with the `QuantumAdder` interface. The class definition above shows the intended API; the internals of `.ammend()` and `.verify()` will change significantly.

### Using as a decorator

For users writing a new custom adder from scratch, `QuantumAdder` works as a decorator:

```python
@QuantumAdder
def my_custom_adder(a, b, c_in=None, c_out=None, ctrl=None):
    ...
```

This is the intended way to define a new adder going forward. The decorator wraps the function at definition time, so `my_custom_adder` is immediately a `QuantumAdder` instance with `.ammend()` and `.verify()` available. You can still call it like a normal function.

If the adder also needs `@custom_control` or another decorator, the stacking order matters: `@QuantumAdder` should be the outermost decorator so it wraps the fully decorated function. Alternatively, users can apply it post-hoc the same way the built-in adders do:

```python
@custom_control
def my_custom_adder(a, b, c_in=None, c_out=None, ctrl=None):
    ...

my_custom_adder = QuantumAdder(my_custom_adder)
```

### Wrapping existing adders

The built-in adders already have their own decorators (several use `@custom_control`). Stacking `@QuantumAdder` on top of `@custom_control` could cause decorator interaction issues, so wrapping is applied post-hoc at the bottom of each adder's module instead:

```python
cuccaro_adder = QuantumAdder(cuccaro_adder)
```

This reassigns the module-level name to the wrapped version. Since `QuantumAdder` is callable, reassigning the module-level name means every existing import and call site keeps working without changes. The same pattern is used to wrap all the other built-in adders.

### Classical-quantum-only adders

Most adders accept both quantum-quantum and classical-quantum input combinations. The Gidney Venting adder is the exception: it only works with classical-quantum inputs (the first argument must be an integer or `BigInteger`, never a quantum register). The `QuantumAdder` interface needs to handle this gracefully rather than letting the user hit a confusing error deep in the carry-venting logic.

The solution is to let each adder declare what it supports up front via metadata, so downstream code can check before calling and either pick a different adder or raise a clear error message.

#### Capability metadata

Each `QuantumAdder` instance carries metadata that describes what it can do:

```python
gidney_cq_venting_adder = QuantumAdder(
    gidney_cq_venting_adder,
    supports_qq=False,
    supports_cq=True,
    supports_control=True,
    supports_carry_in=True,
    modes="dynamic",
)
```

- `supports_qq`: whether the adder accepts quantum-quantum input combinations. True for most adders, False for Gidney Venting.
- `supports_cq`: whether the adder accepts classical-quantum input combinations. This is `True` for all in-place adders.
- `supports_control`: whether the adder natively supports a control qubit. This option is reserved for future work; support for it may be added in a later release.
- `supports_carry_in`: whether the adder natively supports carry-in. Adders that do not support it can still expose carry behavior through `.amend()`, which emulates it.
- `modes`: which execution modes the adder works in: `"static"`, `"dynamic"`, or `"both"`. `"both"` means the adder is mode agnostic.

For the built-in adders, the metadata is known upfront and set when the adder is wrapped. For a custom adder, the user declares it as keyword arguments to `QuantumAdder`. If they don’t, it defaults to the most permissive setting `(supports_qq=True, supports_cq=True, supports_control=True, supports_carry_in=True, modes="both")`.

This lets any function that dispatches to an adder (like `inpl_add`) check these fields before calling it. If the user passes an adder that doesn’t support the current execution mode, input combination, or requested control/carry behavior, the dispatcher raises a clear error instead of letting the adder fail deep in its internals with an unhelpful traceback.

### Adder selection

Today, `inpl_add` is hardcoded to call `thapliyal_procedure`. With `QuantumAdder` and capability metadata, it can accept any adder and validate it against the current context. If no adder is passed, it picks a sensible default. If the passed adder doesn't support the current execution mode or the input combination (for example, passing Gidney Venting for a quantum-quantum addition), it raises an error that explains what went wrong and which adders would work instead.

This also opens the door to smarter defaults in the future. For instance, if both inputs are classical-quantum and the execution mode is dynamic, the dispatcher could prefer Gidney Venting for its low qubit cost. But for now, the goal is just to accept any `QuantumAdder` and validate it, not to build a full auto-selection heuristic.

### Type pluggability

`inpl_add` becomes the single entry point for in-place addition, with an optional `adder` parameter typed as `QuantumAdder`. Instead of being hardcoded to one adder, it gives the user access to all of them:

```python
result = inpl_add(a, b, adder=gidney_adder)
```

When no adder is passed, `inpl_add` picks a default based on the capability metadata: what execution mode is active, whether the inputs are classical-quantum or quantum-quantum, and what each adder supports. The user can override this by passing a specific adder, and the dispatcher validates that it's compatible with the current context before calling it.

The same `adder` parameter propagates to higher-level operations like `multiply`, comparisons, and Montgomery reduction. Each of these ultimately calls `inpl_add` somewhere, and the adder choice flows down through the stack. A user who wants Gidney everywhere can pass it once at the top level of a quantum type, and every addition underneath uses it.

### Input validation

Input validation happens in two layers. The shared `_validate_adder_inputs(a, b)` function in `adder_utilities.py` handles the checks that apply to every adder: is `b` a quantum type? Is `a` a valid classical or quantum type? Are non-numeric types like `QuantumChar` and `QuantumString` rejected?

Everything above that is the responsibility of the quantum type, not the adder. Concerns like exponent alignment, sign handling, precision validation, and modular reduction are handled by each type’s own `__iadd__` before it dispatches to the adder. By the time the adder is called, it only sees integer-valued qubit registers and classical values. The details of what each type needs to do before calling the adder are left to the RFCs that refactor the methods for those types.


## Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **How should the Fourier adder's `perform_QFT` flag work within the `QuantumAdder` interface?** The flag lets users skip the QFT/inverse-QFT when chaining multiple additions in the Fourier basis. This is adder-specific state that doesn't fit the generic signature. Options: make it a parameter on the `QuantumAdder` instance, have the Fourier adder detect internally whether `b` is already in the Fourier basis, or handle it outside the adder entirely.

2. **What verification strategy works in both static and dynamic mode?** The current `inpl_adder_test` relies on statevector inspection, which is static-only. The refactored `.verify()` needs a single test path that works in both modes, consistent with the project's bias towards mode-agnostic code. Measurement-based checking is one option, but it trades exactness for statistical confidence.

3. **What should the default adder be?** When no adder is passed to `inpl_add`, it needs to pick one. The choice depends on what tradeoffs matter most by default (depth vs. ancilla count vs. gate count) and whether the default should vary by execution mode or input type.

### QubitArray integration

There is a proposal to introduce a common `QubitArray` base class that both static
and dynamic modes share, giving them a uniform interface for `.size`, indexing, and
slicing (see Raphael's typing proposal from Community Day). When that work lands:

- `_validate_adder_inputs` simplifies from an isinstance chain to a check against
  `QubitArray | int`.
- The adder type signature becomes `(a: QubitArray | int, b: QubitArray)`, which
  type checkers can verify statically.
- `ammend_inpl_adder`'s size-padding logic can use `QubitArray.size` instead of
  branching on `list` vs. `DynamicQubitArray`.
- Finally, the typing aliases for adder inputs currently in the shared helpers folder become
  redundant and can be removed.

None of this blocks the `QuantumAdder` class. The class and the validation function work with the register types Qrisp has today and will work with `QubitArray` when it arrives.

### Additional adders

Two published adders are worth wrapping as `QuantumAdder` instances in a future release:

- **Takahashi adder** ([arXiv:0910.2530](https://arxiv.org/abs/0910.2530)). An ancilla-free ripple-carry adder with O(n) depth and O(n^2) gates. It occupies a similar niche to the Remaud adder (no workspace qubits needed) but makes a different depth-vs-gate-count tradeoff. Adding it would give users a second zero-ancilla option.

- **DKRS adder** (Draper-Kutin-Rains-Svore, [arXiv:quant-ph/0406142](https://arxiv.org/abs/quant-ph/0406142)). A logarithmic-depth adder using O(n) ancillas. Once QCLA is removed, Qrisp will have no log-depth adder; DKRS would fill that gap with a cleaner construction.

Neither adder is needed for the initial `QuantumAdder` class. They can be implemented later and wrapped with `@QuantumAdder` the same way the existing adders are. For this reason, these adders will be opened up for contributions from community members. 

## Public API impact
[public-api-impact]: #public-api-impact

### Additions

- `QuantumAdder` class: new public type for wrapping adder functions.
- `.ammend()` and `.verify()` methods on every wrapped adder with `ammend` renamed to `amend`.
- `isinstance(x, QuantumAdder)` for type checking.
- Downstream functions (`inpl_add`, `multiply`, etc.) gain an optional `adder` parameter.

### Changes

- Each built-in adder (`cuccaro_adder`, `gidney_adder`, etc.) becomes a `QuantumAdder` instance instead of a plain function. Since `QuantumAdder.__call__` just passes arguments through to the wrapped function, existing code that calls these adders does not need to change.
- `inpl_add` accepts any in-place `QuantumAdder` instead of only `"thapliyal"`.

### Removals

- `qcla` and all its helpers (`cq_qcla`, `qq_qcla`, the Brent-Kung tree implementations). Deprecated first, then removed.

### Migration

- **Users calling adders directly**: no change needed. `cuccaro_adder(a, b)` still works.
- **Users passing adders to `inpl_add`**: replace `adder="thapliyal"` with `adder=thapliyal_adder` (the wrapped instance).
- **Users with custom adders**: wrap with `@QuantumAdder` or `QuantumAdder(my_func)` to get `.ammend()`, `.verify()`, and type compatibility.