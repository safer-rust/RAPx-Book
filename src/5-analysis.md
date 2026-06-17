# Chapter 5. Analysis

This chapter introduces the analysis modules of RAPx, which implement commonly used program analysis features including path analysis, alias analysis, dataflow analysis, control-flow analysis, safety-flow analysis, and more. Please refer to the corresponding sub-chapters for more information.

## Layered Design

Since many static analysis tasks are inherently undecidable due to Rice's Theorem, the analysis modules adopt a layered design that enables users to customize their own analysis routines:

- **`Analysis` trait** (top layer): Defines `name()`, `run()`, and `reset()` — the minimum interface every analysis must implement.
- **Feature subtrait** (middle layer): Each analysis category (e.g., `AliasAnalysis`, `DataflowAnalysis`) extends `Analysis` with domain-specific query methods.
- **Default implementation** (bottom layer): A struct (e.g., `AliasAnalyzer`, `DataflowAnalyzer`) provides a concrete algorithm implementing the subtrait.

Users can utilize a feature by creating an instance of the struct and invoking `run()`, or implement their own subtrait to plug in an alternative algorithm:

```rust
pub trait Analysis {
    fn name(&self) -> &'static str;
    fn run(&mut self);
    fn reset(&mut self);
}

pub trait AliasAnalysis: Analysis {
    fn get_fn_alias(&self, def_id: DefId) -> Option<FnAliasPairs>;
    fn get_all_fn_alias(&self) -> FnAliasMap;
}

pub struct AliasAnalyzer<'tcx> { /* ... */ }

impl Analysis for AliasAnalyzer<'tcx> { /* ... */ }
impl AliasAnalysis for AliasAnalyzer<'tcx> { /* ... */ }
```
