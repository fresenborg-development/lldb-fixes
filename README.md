# lldb-fixes

Our fixes to LLDB until LLVM has them: patches against LLVM's sources, a build or build notes for each program,
and built files as [releases](https://gitlab.com/fresenborg-development/lldb-fixes/-/releases). A part
goes away once its fixes are in an LLVM release.

| Part | What it fixes |
| --- | --- |
| [lldb-server-windows](lldb-server-windows/) | `lldb-server.exe` for Windows: a stray stop and a crash when two threads hit the same breakpoint (LLVM 23.1.3) |
| [lldb-dap-linux](lldb-dap-linux/) | `lldb-dap`: a crash on a frame whose symbol has no name (LLVM 23; patch and build notes, no release) |

## License

The patches change LLVM and are under LLVM's license, the Apache License 2.0 with LLVM Exceptions; so is everything
else here ([LICENSE](LICENSE)). The built programs contain parts of LLVM and of their toolchains; their licenses come
with each release.

Copyright 2026 Jens Fresenborg
