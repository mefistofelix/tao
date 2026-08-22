# Tauriless patch stack

This directory documents the changes carried by the `mefistofelix/tao` fork for Tauriless.
The rest of this repository intentionally keeps the upstream Tao layout unchanged.

## Why this fork exists

Tauriless embeds Tauri in a host process that owns the GUI/main thread. The host must be able to enter the native event loop for a bounded amount of time, return control to the host, and call it again later without destroying or recreating the native application state.

Upstream Tao's normal `run` model owns the event loop until application exit, while the existing return-oriented paths do not provide the exact cross-platform repeated bounded-pump contract needed by Tauriless. This fork therefore adds a small desktop API named `EventLoop::run_timeout`.

`run_timeout(timeout, handler)` runs native event-loop work, may wait for native work for at most `timeout`, returns early when appropriate, and returns control without treating the slice as application shutdown. Repeated calls must preserve event-loop state.

## The three-repository stack

| Repository | Role in the patch stack | Why it is needed |
| --- | --- | --- |
| [`mefistofelix/tao`](https://github.com/mefistofelix/tao) | Lowest-level native event-loop patch. Implements `EventLoop::run_timeout` for supported desktop backends. | Native AppKit/Win32/GTK event processing has to be bounded without emitting teardown semantics between slices. |
| [`mefistofelix/tauri`](https://github.com/mefistofelix/tauri) | Runtime/framework adapter. Its `tauri-runtime-wry` depends on this Tao fork and exposes the bounded pump as `run_timeout`; `App<Wry>` exposes the same operation at the Tauri app level. | Tauriless must pump a real Tauri application, not bypass Tauri's runtime, plugins, IPC, resources, or event translation. |
| [`mefistofelix/tauriless`](https://github.com/mefistofelix/tauriless) | Consumer/embedding layer. Calls Tauri `App<Wry>::run_timeout` from `tauriless_run(runtime, timeout_ms)` while keeping the host in control of the main thread. | Provides the small C ABI used by native/FFI hosts while retaining normal Tauri behavior. |

WRY is deliberately **not forked or patched**. The required behavior belongs to event-loop ownership/pumping in Tao and to the Tauri runtime adapter above it.

## Changes carried in this Tao fork

The patch is intentionally concentrated around the event loop:

- `EventLoop::run_timeout(Duration, handler)` is the public desktop entry point.
- Windows uses a timeout-aware message pump and returns from a slice without running normal loop-destruction/reset behavior.
- macOS runs a bounded native AppKit slice while preserving callback/application state for the next call.
- Linux/GTK preserves activation/control-flow state and performs a timeout-aware iteration without tearing down the event-loop state between slices.
- `examples/run_timeout_probe.rs` exercises repeated timeout slices and native wake-up behavior.

The exact implementation evolves with upstream `dev`; this document describes the contract rather than freezing file offsets or a particular SHA.

## Dependency direction

The intended dependency direction is strictly:

```text
Tauriless
    -> mefistofelix/tauri (dev)
        -> mefistofelix/tao (dev)
            -> native platform event loops
```

Tao does not depend on Tauri or Tauriless. The cross-links here exist only so someone arriving at this fork can understand why the Tao delta exists and where it is consumed.

## Upstream synchronization

This fork tracks Tao upstream `dev`. When updating it:

1. bring the latest upstream `dev` into this fork's `dev` branch;
2. preserve or adapt the `run_timeout` contract to the current Tao internals;
3. run `cargo run --example run_timeout_probe` on the supported desktop platforms;
4. update the Tauri fork so its lock/dependency resolves the new Tao `dev` head;
5. update Tauriless' Cargo lock and rerun its native/WebView regression tests.

Do not reintroduce a separate Tauriless patch branch or vendored Tao copy. The maintained patch lives directly on this fork's `dev` branch.
