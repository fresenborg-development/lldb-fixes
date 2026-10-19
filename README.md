# lldb-server for Windows, with fixes for breakpoints hit by several threads

A build of LLVM's `lldb-server.exe` for 64-bit Windows with two fixes that are not (yet) in LLVM. Both concern a
breakpoint that two threads hit at almost the same time; with the stock `lldb-server.exe` of LLVM 23 a debug session
then stops for no reason or loses its connection.

The `.exe` is cross-built on Linux with [llvm-mingw](https://github.com/mstorsjo/llvm-mingw) (ucrt) and linked
statically: it needs no DLLs besides those of Windows.

## Download

Each [release](../../releases) has:

| File | Contents |
| --- | --- |
| `lldb-server.exe` | the program, stripped |
| `lldb-server-debug.exe` | the same build with its DWARF debug info, for symbolizing crashes |
| `SHA256SUMS` | checksums of both |
| `versions.txt` | LLVM and llvm-mingw versions and the checksum of each patch |
| `licenses/` | licenses of LLVM, libc++ and libunwind, which are linked into the `.exe` |

Releases are tagged `<LLVM version>-<revision of the patches>`, e.g. `23.1.3-1`.

## The fixes

1. **A breakpoint removed while a second hit is pending**
   ([patches/0001](patches/0001-windows-resume-thread-at-removed-breakpoint.patch)).
   When two threads hit the same breakpoint, the second exception is still queued while the client handles the first
   stop. If the client removes the breakpoint meanwhile (or lifts it to step the first thread over it), the queued
   exception arrives for a trap that is no longer in memory. `lldb-server` reported it as an unknown exception
   (`Exception 0x80000003 encountered`) and left the program counter in the middle of the original instruction. With
   the fix the thread is set back and resumed: it runs the original instruction, or hits the breakpoint again.

2. **A crash when stepping over the breakpoint**
   ([patches/0002](patches/0002-windows-guard-active-exception.patch)).
   The debugger thread and the thread that resumes the process both used the active exception record without
   synchronization; while stepping two threads over the breakpoint, `lldb-server.exe` crashed with an access violation
   (`0xC0000005`) and the client reported a lost connection. The record is now guarded by a mutex, and the delegate
   gets its own copy.

Both fixes will be offered to LLVM; this project ends once they are in a release.

## Building

Any OCI builder with cache mounts works, e.g. Podman:

```sh
podman build --tag lldb-server-windows:23.1.3-1 .
podman create --name out lldb-server-windows:23.1.3-1
podman cp out:/out ./out
podman rm out
```

The build downloads the LLVM sources and llvm-mingw, both pinned by SHA-256, and Debian packages from a fixed
snapshot. It takes about 10 minutes and up to 8 GB of memory; lower `COMPILE_JOBS` (default 6) if the compiler is
killed for lack of memory:

```sh
podman build --build-arg COMPILE_JOBS=3 --tag lldb-server-windows:23.1.3-1 .
```

## License

The patches change LLVM and are under LLVM's license, the Apache License 2.0 with LLVM Exceptions; so is everything
else here ([LICENSE](LICENSE)). The built `.exe` contains LLVM, libc++, libunwind and the mingw-w64 runtime; their
licenses come with each release.

Copyright 2026 Jens Fresenborg
