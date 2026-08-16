# Portable Runtime Navigation Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Waypie's targeting and submenu navigation executable and testable as a platform-independent Rust runtime on Windows without changing the existing semantics.

**Architecture:** Keep `MenuState` as the single owner of geometry, targeting, path, and history rules. Add a thin `Runtime` owner around it so platform adapters feed semantic events instead of reimplementing decisions. Gate Wayland-only dependencies so the portable library can compile natively on Windows.

**Tech Stack:** Rust 2024, Cargo features, existing Waypie `Config`/`MenuState`/`Point`/`Target`, built-in Rust test framework.

## Global Constraints

- Do not rewrite already-portable modules without evidence that their API blocks extraction.
- Do not duplicate targeting or navigation algorithms in `runtime.rs`; delegate to `MenuState`.
- Portable tests run with `cargo test --lib --no-default-features` on Windows.
- Default builds retain the existing Wayland application feature.
- Runtime code must contain no Wayland, GPUI, Win32, OpenLogi, filesystem, process-execution, or wall-clock APIs.
- Every behavior change follows RED → minimal implementation → GREEN → review → atomic commit.

---## File Structure

- Modify `Cargo.toml`: make Wayland-only crates optional behind a default `wayland` feature.
- Modify `src/lib.rs`: compile `app` only with the `wayland` feature and export `runtime` unconditionally.
- Create `src/runtime.rs`: own `Config` + `MenuState`, expose deterministic navigation inputs and snapshots.
- Create `tests/runtime_navigation.rs`: differential tests comparing `Runtime` snapshots against direct `MenuState` oracle sequences.

### Task 1: Establish a Windows-portable library build

**Files:**
- Modify: `Cargo.toml`
- Modify: `src/lib.rs`

**Interfaces:**
- Produces: Cargo feature `wayland` enabled by default.
- Produces: `waypie` library compilable with `--no-default-features` on Windows.

- [ ] **Step 1: Record the existing RED build**

Run: `C:\Users\volke\.cargo\bin\cargo.exe test --lib`
Expected: FAIL in `wayland-sys` because `std::os::unix` is unavailable on Windows.

- [ ] **Step 2: Make Wayland dependencies optional**

Set `[features] default = ["wayland"]` and `wayland = ["dep:smithay-client-toolkit", "dep:wayland-client", "dep:wayland-protocols-wlr"]`. Mark those three dependencies `optional = true`.

- [ ] **Step 3: Gate the Wayland app module**

Change `src/lib.rs` so `pub mod app;` is guarded by `#[cfg(feature = "wayland")]` while all existing portable modules remain unconditional.

- [ ] **Step 4: Verify portable GREEN**

Run: `C:\Users\volke\.cargo\bin\cargo.exe test --lib --no-default-features`
Expected: all portable unit tests PASS on Windows.

- [ ] **Step 5: Commit**

Commit only `Cargo.toml`, `Cargo.lock` if Cargo changes it, and `src/lib.rs` with message `build: isolate wayland dependencies`.
### Task 2: Add the Runtime boundary and exact snapshot contract

**Files:**
- Create: `src/runtime.rs`
- Modify: `src/lib.rs`
- Test: `tests/runtime_navigation.rs`

**Interfaces:**
- Produces: `pub struct Runtime` owning `Config` and `MenuState`.
- Produces: `pub struct RuntimeSnapshot { path: Vec<usize>, centers: Vec<Point>, pointer: Option<Point>, active: Option<Target> }`.
- Produces: `Runtime::new(config: Config) -> Runtime` and `Runtime::snapshot(&self) -> RuntimeSnapshot`.

- [ ] **Step 1: Write the failing integration test**

Create a test importing `waypie::runtime::{Runtime, RuntimeSnapshot}` and asserting a new runtime snapshot is empty: empty path/centers and `None` pointer/active.

- [ ] **Step 2: Run the focused test to verify RED**

Run: `C:\Users\volke\.cargo\bin\cargo.exe test --no-default-features --test runtime_navigation runtime_starts_with_empty_menu_state`
Expected: FAIL because `waypie::runtime` does not exist.

- [ ] **Step 3: Implement the minimal runtime owner**

Create `src/runtime.rs` with `Runtime { config: Config, state: MenuState }`, exact snapshot cloning from `MenuState`, and no platform imports. Export it from `src/lib.rs`.

- [ ] **Step 4: Run focused and portable suites**

Run the focused test, then `cargo test --lib --no-default-features` and `cargo test --no-default-features --test runtime_navigation`.
Expected: all PASS.

- [ ] **Step 5: Commit**

Commit `src/runtime.rs`, `src/lib.rs`, and `tests/runtime_navigation.rs` with message `feat: add portable runtime state owner`.

### Task 3: Prove pointer targeting parity against MenuState

**Files:**
- Modify: `src/runtime.rs`
- Modify: `tests/runtime_navigation.rs`

**Interfaces:**
- Produces: `Runtime::place_root(center: Point, pointer: Point, width: u32, height: u32, center_hitbox: f64)`.
- Produces: `Runtime::update_pointer(position: Point, center_hitbox: f64) -> bool`.

- [ ] **Step 1: Add a differential RED test**

Build one `Config`, one direct `MenuState` oracle, and one `Runtime`. Feed identical root placement and pointer positions for center, up-item, and right-item cases. After each operation assert `runtime.snapshot() == snapshot_of(&oracle)`.

- [ ] **Step 2: Run focused test and verify RED**

Run: `cargo test --no-default-features --test runtime_navigation pointer_targeting_matches_menu_state_oracle`
Expected: FAIL because the runtime navigation methods are absent.

- [ ] **Step 3: Implement by delegation only**

`place_root` calls `MenuState::place_root` then `MenuState::update_pointer`; `update_pointer` calls the existing `MenuState::update_pointer`. Do not duplicate angle, distance, or hitbox calculations.

- [ ] **Step 4: Verify exact parity**

Run the focused test and the full `--no-default-features` test suite.
Expected: PASS with exact `Point`, path, pointer, and `Target` equality.

- [ ] **Step 5: Commit**

Commit with message `feat: route pointer targeting through portable runtime`.
### Task 4: Prove submenu open/return parity

**Files:**
- Modify: `src/runtime.rs`
- Modify: `tests/runtime_navigation.rs`

**Interfaces:**
- Produces: `Runtime::open_submenu(index: usize, at: Point, width: u32, height: u32) -> bool`.
- Produces: `Runtime::return_to(depth: usize, at: Point, width: u32, height: u32) -> bool`.

- [ ] **Step 1: Add differential navigation tests**

Construct a two-level submenu config. Run identical `place_root`, `open_submenu`, second pointer update, and `return_to` operations on direct `MenuState` and `Runtime`. Assert exact snapshots after every transition, including path, all history centers, pointer, and active target.

- [ ] **Step 2: Add invalid-operation oracle cases**

Assert both implementations return `false` and preserve snapshots when opening a leaf, opening an out-of-range index, returning to an invalid depth, or opening before root placement.

- [ ] **Step 3: Run tests to verify RED**

Run: `cargo test --no-default-features --test runtime_navigation submenu_navigation_matches_menu_state_oracle`
Expected: FAIL because runtime submenu methods are absent.

- [ ] **Step 4: Implement by delegation only**

Call the existing `MenuState::open_submenu` and `MenuState::return_to` using the runtime-owned config. Preserve each method's boolean result exactly.

- [ ] **Step 5: Run all portable tests**

Run: `C:\Users\volke\.cargo\bin\cargo.exe test --no-default-features`
Expected: all unit and integration tests PASS.

- [ ] **Step 6: Commit**

Commit with message `feat: route submenu navigation through portable runtime`.

### Task 5: Slice acceptance gate

**Files:**
- Review: `src/runtime.rs`, `src/lib.rs`, `Cargo.toml`, `tests/runtime_navigation.rs`

- [ ] **Step 1: Platform-purity scan**

Search `src/runtime.rs` for `wayland`, `smithay`, `gpui`, `windows`, `openlogi`, `std::process`, `std::fs`, and `Instant::now`. Expected: zero matches.

- [ ] **Step 2: Full portable regression**

Run: `C:\Users\volke\.cargo\bin\cargo.exe test --no-default-features`
Expected: PASS.

- [ ] **Step 3: Review semantic ownership**

Confirm runtime navigation contains no copied angle/distance/path algorithms and all such decisions still come from `MenuState`.

- [ ] **Step 4: Verify repository cleanliness**

Run `git diff --check` and `git status --short`. Expected: no whitespace errors and only intentional plan-tracking changes, if any.

## Slice Definition of Done

Windows can compile and test the portable Waypie library without Wayland dependencies. A `Runtime` owns config and menu state, exposes exact snapshots, and matches direct `MenuState` behavior for root placement, pointer targeting, submenu entry, return navigation, and invalid operations. No navigation algorithm exists twice.