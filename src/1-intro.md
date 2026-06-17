# Chapter 1. Introduction

RAPx is a static analysis and verification platform for Rust programs. It has two objectives:

* To serve as a companion to the Rust compiler in detecting semantic bugs related to unsafe code.
* To provide ready-to-use program analysis features for tool developers.

Although Rust has made significant progress in ensuring memory safety and protecting software from undefined behavior, the Rust compiler's capability is inherently limited due to Rice's theorem. In general, it refrains from detecting undefined behaviors related to unsafe code, as achieving both precise and efficient analysis is extremely challenging. This limitation arises because the Rust compiler prioritizes usability, as developers cannot tolerate false positives. RAPx is not bound by this constraint. It may produce false positives, as long as they remain within an acceptable range.

Writing program analysis tools is challenging. In this project, we also aim to provide a user-friendly framework for developing new static program analysis features for Rust. In particular, our project integrates dozens of classic program analysis algorithms, including those for pointer analysis, value-flow analysis, control-flow analysis, and more. Developers can choose and combine them like Lego blocks to achieve new bug detection or verification capabilities. The project repository is at [github.com/safer-rust/RAPx](https://github.com/safer-rust/RAPx).

## Architecture

RAPx is organized into three top-level modules:

| Module | Crate path | Purpose |
|--------|-----------|---------|
| **Analysis** | `rapx::analysis` | Foundational static analyses — path enumeration, alias analysis, dataflow, range analysis, call graphs, API dependency graphs, owned-heap classification, and safety-flow analysis. Each analysis implements the `Analysis` trait and can be composed downstream. |
| **Check** | `rapx::check` | Bug detection passes built on top of the analysis layer — use-after-free / dangling pointer detection (`SafeDrop`), memory leak detection (`rCanary`), and performance anti-pattern checks (`Opt`). |
| **Verify** | `rapx::verify` | Contract-based verification of unsafe code. Collects verification targets, resolves callee safety contracts from `#[rapx::requires]` annotations, enumerates SCC-aware CFG paths to each unsafe callsite, performs backward/forward MIR state tracking, and dispatches SMT queries via Z3 to prove or disprove safety preconditions. |

## Vision

The verification module (`rapx::verify`) is positioned to become the **harness for Rust Agentic Coding** — an automated safety gate for AI-generated Rust code. As LLM-based coding agents increasingly produce `unsafe` blocks, a fast, contract-driven verifier that can independently confirm or reject the soundness of generated unsafe code is essential. RAPx's verification pipeline — from target collection through path-sensitive state tracking to SMT-backed property checking — provides the foundation for this role. Future directions include tighter integration with IDE and CI workflows, richer contract expression, and inter-procedural summary propagation to scale verification to whole crates.
