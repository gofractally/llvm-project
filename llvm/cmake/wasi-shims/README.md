# `llvm/cmake/wasi-shims/` — libc++ threading shims for wasi cross-builds

These shim headers let LLVM's source tree compile against wasi-sdk's
`wasm32-wasip1` libc++ (the variant built without thread support) without
either rewriting every `std::mutex`/`std::thread`/`std::future` usage in LLVM
as `llvm::sys::` wrappers or splitting half of `libLLVMSupport` behind
`#if LLVM_ENABLE_THREADS`.

## How the shims work

Each shim file sits at a well-known name (`mutex`, `condition_variable`,
`shared_mutex`, `thread`, `future`) and is placed ahead of the real libc++
include path via `-isystem` by the wasi cross-compile toolchain in the
psiserve repo.

The shim starts with `#include <__config>` to learn libc++'s threading
configuration and then:

* When libc++ has threads, the shim is a pure passthrough: `#include_next`
  delivers the real header, no stub types are emitted. Zero runtime cost.
* When libc++ is built without threads (`_LIBCPP_HAS_NO_THREADS` defined,
  or `_LIBCPP_HAS_THREADS == 0`), the shim supplies no-op stub types
  sufficient to satisfy type lookup for LLVM code that is itself dead
  under `LLVM_ENABLE_THREADS=OFF`.

## When this is unnecessary

If you're compiling LLVM for a target whose libc++ has real thread support
(`wasm32-wasip1-threads`, Linux, macOS) you do not need to opt into these
shims. The shim is designed to be a no-op on threaded configurations, so
leaving it on the `-isystem` path is harmless — but it's also not needed.

## Upstreaming path

The shims are a workaround for a broader problem: LLVM's Support library
uses `std::mutex` / `std::thread` / `std::future` in places where the
primitives are required to compile but never executed at runtime (because
the surrounding code is guarded by `#if LLVM_ENABLE_THREADS`).

A better upstream fix — for the common cases — is to move the `<mutex>` /
`<condition_variable>` includes and the struct members behind
`LLVM_ENABLE_THREADS`. This fork carries those fixes as individual commits
on top of `llvmorg-22.1.2`. The shims cover the residual files where the
fix would require invasive API changes (e.g. `ThreadPoolInterface::async`
returning `std::shared_future`).
