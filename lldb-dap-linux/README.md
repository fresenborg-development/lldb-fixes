# lldb-dap: crash on a frame whose symbol has no name

A fix for LLVM's `lldb-dap`, found with the Linux `lldb-dap` of [apt.llvm.org](https://apt.llvm.org) (LLVM 23)
debugging a Windows program through a remote `lldb-server`. There is no built program here: the fix is small enough
to build `lldb-dap` from the sources of the installed LLVM packages (see below).

## The fix

**A symbol without a name** ([patches/0001](patches/0001-lldb-dap-symbol-without-name.patch)).
A symbol can have no name, for example a function of a Windows DLL known only from its unwind information.
`DAP::ResolveAssemblySource` turned `SBSymbol::GetName()` into a `std::string`; for such a symbol it is `nullptr`, and
`lldb-dap` crashed as soon as a client asked for a stack that contains such a frame (e.g. Eclipse CDT, which fetches
the stacks of all threads). With the fix the frame is named by its load address, as a frame without a symbol already
was.

This will be offered to LLVM; this part ends once it is in a release.

## Building

`lldb-dap` uses only the public SB API of `liblldb`, so it can be built on its own and linked against the
`liblldb.so` of the installed packages. Besides the patched `lldb-dap` sources it needs `lldbHost` and `lldbUtility`
of the same sources. The sources must be of exactly the commit the packages were built from: apt.llvm.org builds
snapshots of the release branch, and the package version names the commit
(`1:23.1.4~++20261019064016+52e90e1917a4-…`).

In short, with `llvm-23-dev`, `liblldb-23-dev`, `libclang-23-dev` (of the same version), `cmake` and `ninja-build`
installed and the sources of that commit (at least `lldb/`, `cmake/`, `llvm/cmake/`, `third-party/unittest/`) in
`src/` with the patch applied:

```sh
L=/usr/lib/llvm-23
cmake -G Ninja -S src/lldb -B build -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang-23 -DCMAKE_CXX_COMPILER=clang++-23 \
  -DLLVM_DIR=$L/lib/cmake/llvm -DClang_DIR=$L/lib/cmake/clang \
  -DLLVM_TABLEGEN=$L/bin/llvm-tblgen -DLLDB_TABLEGEN_EXE=$L/bin/lldb-tblgen \
  -DLLDB_ENABLE_PYTHON=OFF -DLLDB_ENABLE_LUA=OFF -DLLDB_ENABLE_LIBEDIT=OFF -DLLDB_ENABLE_CURSES=OFF \
  -DLLDB_ENABLE_LIBXML2=OFF -DLLDB_ENABLE_LZMA=OFF -DLLDB_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_TESTS=OFF
ninja -C build lib/liblldbDAP.a lib/liblldbHost.a lib/liblldbUtility.a \
  tools/lldb-dap/tool/CMakeFiles/lldb-dap.dir/lldb-dap.cpp.o
cd build
clang++-23 -fPIC -O3 -Wl,--gc-sections -Wl,--exclude-libs,ALL \
  tools/lldb-dap/tool/CMakeFiles/lldb-dap.dir/lldb-dap.cpp.o -o lldb-dap -Wl,-rpath,$L/lib \
  lib/liblldbDAP.a $L/lib/liblldb.so lib/liblldbHost.a lib/liblldbUtility.a \
  $L/lib/libclang-cpp.so.23.1 $L/lib/libLLVM.so.1 -ldl
```

This takes less than a minute; the result replaces `/usr/lib/llvm-23/bin/lldb-dap`.

Licenses: see the [README](../README.md) of the repository.
