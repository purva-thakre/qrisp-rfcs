# Qrisp RFCs

A lightweight, temporary place to write up and discuss substantial changes to Qrisp, taking inspiration from the [Rust RFC process](https://github.com/rust-lang/rfcs). It is currently focused on the arithmetic refactoring proposal.

## Proposing a change

1. Copy `0000-template.md` to `text/0000-short-name.md`.
2. Fill it in.
3. Open a pull request.

Discussion happens on the pull request. RFC files are numbered after the PR number once opened.

## Proposals

The arithmetic refactor will be proposed as a series of RFCs covering the `QuantumAdder` abstraction, `QuantumFloat`/`QuantumModulus` cleanup, Montgomery arithmetic consolidation, SBP removal and comparison unification, and a fixed-point framework to handle, overflow, truncation etc. consistently across all arithmetic functions.

## Links

- [Qrisp repository](https://github.com/eclipse-qrisp/Qrisp)
- [Qrisp Discord](https://discord.gg/v5np7DeBaq)
