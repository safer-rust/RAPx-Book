# Chapter 8. Verification

Unsafe code enables low-level operations while circumventing Rust's safety guarantees, and may introduce undefined behavior (UB) if misused. The verification module ([`rapx::verify`](https://github.com/safer-rust/RAPx/tree/main/rapx/src/verify/)) provides a staged pipeline for checking that unsafe call sites satisfy their callee's safety preconditions. It combines path-sensitive MIR traversal, abstract interpretation, and Z3-based SMT solving to produce verdicts of Proved or Unproved for each safety property at each unsafe call site.

## Chapter Outline

- [8.1 Design Principles and Verification Modes](./8.1-overview.md) — the three assumptions, contract-based verification, and the `scan` / `targeted` / `invless` modes
- [8.2 Verification Target Collection](./8.2-collection.md) — how targets are gathered and how contracts are resolved (annotation → JSON → call-chain)
- [8.3 Safety Property Contracts](./8.3-contracts.md) — complete syntax reference: property kinds, argument grammar, `ValidNum` predicates, direct annotation syntax, and the JSON contract database format
- [8.4 The Verification Pipeline](./8.4-pipeline.md) — a complete walkthrough of the verification pipeline using a linked-list case study: path extraction, backward slicing, forward verification, SMT check, and auto-repeat detectors
