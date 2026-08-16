# Portable Runtime Design

## Goal

Make Waypie's interaction semantics platform-independent Rust so Linux/Wayland and Windows execute the same state machine. The current Wayland implementation remains the behavioral oracle during extraction.

The port is complete only when platform adapters own OS resources while portable menu behavior lives in shared Rust.

## Non-goals

- Do not rewrite already-portable modules (`geometry`, `model`, `hover`, `animation`, `config`, `style`, `visual`) without evidence that their API blocks extraction.
- Do not duplicate menu selection, navigation, animation, or timing semantics in the Windows host.
- Do not introduce GPUI, Win32, OpenLogi, Wayland, renderer, filesystem, or process-execution types into the portable runtime.
- Do not change observable Waypie behavior while extracting it.

## Architecture

Add `src/runtime.rs` as the portable interaction state machine. It owns semantic state and consumes explicit inputs. Platform adapters translate native events into runtime inputs and execute returned effects.

`src/app.rs` becomes the Wayland adapter incrementally. The future Windows adapter will use the same `Runtime`.

Portable ownership includes `MenuState`, `HoverDetector`, pointer-hold/turbo state, modifier and quick-key state, animation profile, `Animator`, visibility/closing state, and render-state synthesis.
## Runtime Contract

Inputs are deterministic semantic events such as pointer motion, left-button press/release, key/modifier changes, ticks with caller-supplied time, show, and hide. The runtime must not call `Instant::now()` internally for behavior that affects parity.

Effects are platform-neutral requests such as `RequestRender`, `ExecuteCommand`, `BeginClose`, and `Closed`. The runtime never launches processes or manipulates native windows itself.

A runtime snapshot must expose enough semantic state for differential tests: menu path, centers, pointer, active target, hover/turbo state, animation state, lifecycle state, and render-state identity.

## Extraction Order

1. Targeting and navigation.
2. Hover detection integration.
3. Pointer hold, turbo, and hold-to-turbo semantics.
4. Keyboard modifiers, quick keys, and back keys.
5. Center actions, submenu open, and return navigation.
6. Animation profile and animator ownership.
7. Visual-node and render-state synthesis.
8. Show/hide/close lifecycle and action effects.

Each slice moves behavior from `app.rs` only after equivalent tests exist against the original behavior.

## Oracle and Testing

Before extracting a behavior, pin the current Wayland semantics with an oracle test. Add the corresponding runtime test first and observe it fail for the intended missing behavior. Implement the smallest runtime change, then run both oracle and runtime suites to green.
Differential traces compare the same logical input sequence and caller-supplied timestamps. Floating-point parity checks use exact bit representations where practical rather than rounded textual output.

No extraction slice is complete if the Wayland adapter still owns a second copy of the migrated semantic decision.

## Error Handling

Invalid semantic operations return explicit no-op/false results or typed runtime effects; platform failures remain adapter errors. A platform failure must not corrupt runtime state. Command execution failures are reported by the adapter and do not mutate menu semantics after the runtime has emitted the command request.

## Platform Boundaries

Wayland retains compositor, layer-surface, input-inhibitor, SHM, native keyboard/pointer translation, and presentation responsibilities. Windows will own global input capture, monitor/DPI translation, GPUI window/presentation integration, IPC, and process execution.

Both adapters consume the same config/style/model data and the same runtime outputs. Neither adapter may independently choose targets, submenu paths, turbo transitions, animation targets, or close semantics.

## Definition of Done

- `runtime.rs` contains all portable interaction decisions currently embedded in `app.rs`.
- Wayland behavior remains regression-green after every extraction slice.
- The Windows host can drive the same runtime without importing Wayland crates.
- Every migrated semantic branch has deterministic tests.
- Differential oracle traces show no unexplained state divergence for the covered corpus.
- `app.rs` is reduced to platform translation, resource management, and effect execution.
- There is one semantic owner for each behavior; no duplicate Windows implementation is accepted.
