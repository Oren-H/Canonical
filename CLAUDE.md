# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Canonical is a sound and complete type inhabitation solver for dependent type theory (ITP 2025 paper: https://arxiv.org/abs/2504.06239). Given a type, it exhaustively searches for a term of that type — used for automated theorem proving and program synthesis in Lean 4. This repo is the Rust solver; the Lean tactic frontend lives in a separate repo (https://github.com/chasenorman/CanonicalLean) and loads this code as a shared library over the C FFI.

## Commands

- `cargo build` — build the workspace (three crates under `crates/`).
- `python3 build_lean.py` — build the `canonical_lean` cdylib in release mode and install it into `lean/.lake/packages/Canonical/.lake/build/lib/`, replacing the prebuilt one that ships with the CanonicalLean package. This is the main dev loop: edit Rust → run this script → restart the Lean language server so it reloads the dylib. The script deletes the old dylib before copying — overwriting in place reuses the inode, which invalidates macOS's cached code signature and gets the Lean process SIGKILLed. Don't "simplify" it to a plain copy.
- `cargo run -p canonical-compat` — standalone CLI debug entry point. Loads a problem from `lean/debug.json` (produce one with the tactic's `+debug` option in Lean) and runs the prover outside of Lean, printing steps/sec. Useful for profiling and debugging without FFI in the way.
- There is no Rust test suite; testing happens through the Lean project.

### Lean test project (`lean/`)

- Open the `lean/` directory in VSCode (or run `lake build` inside it) to fetch dependencies: the prebuilt CanonicalLean package (pinned to the `program-synthesis` rev, matching this git branch) and Mathlib v4.30.0. Toolchain: `leanprover/lean4:v4.30.0`.
- `lean/Test.lean` is the scratch file for exercising the solver; `lean/Results/` holds benchmark files (NNG, Lean prelude, monomorphization experiments, etc.).
- CI (`.github/workflows/main.yml`) builds the dylib per-platform and uploads to a GitHub release; Windows builds use the `x86_64-pc-windows-gnu` target with Lean's bundled clang as linker.

## Architecture

Data flows: Lean tactic → C FFI (`canonical-lean`) → string-named IR (`canonical-compat`) → compiled core representation (`canonical-core`) → search → found terms flow back out the same way.

### `crates/canonical-core` — the solver

All terms are β-normal, η-long (BNEL): every term is `λ params. let lets. head args`, and types are the analogous Π-form. Static arity means a variable's type alone determines its argument count, so refinement can create all argument metavariables at once. Search starts from one metavariable for the goal type and repeatedly picks a metavariable, chooses a head symbol from its local context, and spawns argument metavariables; typing constraints become equational constraints checked eagerly so violated branches die early.

- `core.rs` — the type theory: `Meta` (metavariables), `Term`/`Type`, `Decl`, explicit substitutions (`ES`, `Subst`) that make β-reduction O(1) on partial terms, `Equation`/`RedexConstraint` constraints, and weak-head normalization (`whnf`).
- `search.rs` — entropy computation, next-metavariable selection, assignment testing. Owns the global `RUN: AtomicBool` used for cancellation across threads.
- `heuristic.rs` — the entropy metric (product of branching factors, predicted for unrefined metavariables) and refinement ordering.
- `prover.rs` — `Prover::prove(callback, verbose)`: parallelized (rayon fork-join) iterative-deepening DFS where the deepening metric is entropy, doubled per iteration. The callback fires per found term; backtracking is direct mutation, so nothing persistent to snapshot.
- `memory.rs` — `S<T>`/`W<T>` strong/weak pointer wrappers used throughout; callers keep `owned_linked: Vec<S<Linked>>` alive so weak refs into contexts stay valid.
- `compiler.rs` compiles declarations, `stats.rs` tracks search statistics fed back into entropy prediction, `print.rs` handles term display.

### `crates/canonical-compat` — IR and tooling

- `ir.rs` — the serde-serializable intermediate representation: `IRDecl` (name, optional type, defining `IREquation`s), `IRExpr` (params/lets/spine), `IRSpine` (head + args). Heads are plain strings; `IRDecl::to_problem` resolves them and builds the core `Decl`. JSON save/load lives here.
- `refine.rs` — an Axum web server on port 3000 serving `static/index.html`: an interactive UI for manually stepping/undoing refinements, with autofill for forced moves. Driven from Lean via the `refine`/`get_refinement` FFI calls, sharing `GLOBAL_STATE`.
- `reduction.rs` — free-variable and pattern analysis for equations; `ai.rs` — MessagePack serialization of `Example`s (problem + unification statistics) for training data.

### `crates/canonical-lean` — Lean FFI (cdylib)

Hand-written marshalling between Lean runtime objects and the IR: `#[repr(C)]` structs mirror Lean's object layouts (`lean.h`), tags are asserted, and construction goes through Lean's exported allocator. Exported entry points: `canonical` (run the prover with timeout and solution count), `cancel`, `refine`/`get_refinement` (drive the web UI), `save_problem`, and to-string helpers. Invariants to preserve:

- Allocate Lean objects only via `lean_alloc_object` — the crate graph links its own mimalloc via `canonical-core`, and mixing heaps crashes when Lean frees the object.
- Only one prover instance runs at a time (the `INSTANCE` mutex); `cancel` spins on `RUN` until the instance lock frees.
- All entry points wrap work in `catch_unwind` and convert panics into Lean IO errors — don't let a panic cross the FFI boundary.

`.cargo/config.toml` sets the dylib's macOS `install_name` to `@loader_path/libcanonical_lean.dylib` so the copied library resolves inside the Lean package.
