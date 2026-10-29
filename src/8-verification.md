# Chapter 8. Verification

Unsafe code enables low-level operations while circumventing Rust's safety guarantees, and may introduce undefined behavior (UB) if misused. The verification module ([`rapx::verify`](https://github.com/safer-rust/RAPx/tree/main/rapx/src/verify/)) provides a staged pipeline for checking that unsafe call sites satisfy their callee's safety preconditions. It combines bounded path-sensitive MIR traversal, symbolic VM execution, property-specific rules, and Z3 queries where needed. Property checks can be `Proved`, `Failed`, or `Unknown`; a function-level `SOUND` result covers only the paths and operations the verifier examined, as described in [§8.1](./8.1-overview.md).

## Chapter Outline

- [8.1 Design Principles and Verification Modes](./8.1-overview.md) — the three assumptions, contract-based verification, and the `scan` / `targeted` modes
- [8.2 Verification Target Collection](./8.2-collection.md) — how targets are gathered and how contracts are resolved (annotation → JSON → call-chain)
- [8.3 Safety Property Contracts](./8.3-contracts.md) — built-in property reference, user-defined contracts (the `def` DSL), contract annotation (direct + JSON), and inspecting with `--debug-contracts`
- [8.4 The Verification Pipeline](./8.4-pipeline.md) — a walkthrough using a linked-list case study: loop-repeat planning, path extraction, backward slicing, symbolic VM execution, and property checking
