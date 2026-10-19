# lldb-server.exe for Windows: LLVM's sources with the fixes in patches/, cross-built for x86_64-w64-mingw32 (ucrt)
# with the Linux toolchain of llvm-mingw; only the target lldb-server, linked statically (no libLLVM or libc++ DLLs
# next to it). Build with any OCI builder that supports cache mounts, e.g.
#
#   podman build --tag lldb-server-windows:23.1.3-1 .
#
# The tag is <LLVM_VERSION>-<revision of the patches>; a new patch or LLVM version gets a new tag.
#
# Result in the image, under /out:
#   lldb-server.exe         stripped, for use
#   lldb-server-debug.exe   the same build with its DWARF debug info, for symbolizing crashes
#   SHA256SUMS, versions.txt, licenses/

FROM docker.io/library/debian:trixie-slim@sha256:a99cfc517144bc59b1978475ec53b46ecabec7e43635402ee5b77cc54cd1b20a

# ---- Debian packages from a fixed snapshot, kept in the apt build cache ---------------------
ARG DEBIAN_SNAPSHOT=20261001T000000Z
RUN rm -f /etc/apt/sources.list.d/debian.sources /etc/apt/apt.conf.d/docker-clean \
 && echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache \
 && printf '%s\n' \
    "Types: deb" \
    "URIs: http://snapshot.debian.org/archive/debian/${DEBIAN_SNAPSHOT}" \
    "Suites: trixie trixie-updates" \
    "Components: main" \
    "Check-Valid-Until: no" \
    "Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg" \
    > /etc/apt/sources.list.d/snapshot.sources

# g++ builds LLVM's own build tools (tablegen) for this computer; cmake and ninja drive the build.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl xz-utils patch python3 cmake ninja-build g++

# fetch <url> <sha256> <file>: download into the build cache /downloads unless already there.
RUN printf '%s\n' '#!/bin/sh' 'set -eu' \
      'f="/downloads/$3"' \
      'if ! echo "$2  $f" | sha256sum -c --status 2>/dev/null; then' \
      '  curl -fL --progress-bar -o "$f.part" "$1" && mv "$f.part" "$f"' \
      '  echo "$2  $f" | sha256sum -c -' \
      'fi' > /usr/local/bin/fetch \
 && chmod +x /usr/local/bin/fetch

# ---- llvm-mingw for Linux: the cross compiler --------------------------------------------------
ARG LLVM_MINGW=20260922
ARG LLVM_MINGW_SHA256=bb7bb7654b33d5aa8712acb837c963b2e0c56352560c76105270a3268c665c21
RUN --mount=type=cache,target=/downloads \
    fetch https://github.com/mstorsjo/llvm-mingw/releases/download/${LLVM_MINGW}/llvm-mingw-${LLVM_MINGW}-ucrt-ubuntu-22.04-x86_64.tar.xz \
          ${LLVM_MINGW_SHA256} llvm-mingw-${LLVM_MINGW}-ucrt-ubuntu-22.04-x86_64.tar.xz \
 && mkdir /opt/llvm-mingw \
 && tar -xJf /downloads/llvm-mingw-${LLVM_MINGW}-ucrt-ubuntu-22.04-x86_64.tar.xz -C /opt/llvm-mingw --strip-components=1
ENV PATH=/opt/llvm-mingw/bin:${PATH}

# ---- LLVM's sources ----------------------------------------------------------------------------
ARG LLVM_VERSION=23.1.3
ARG LLVM_SHA256=c44186a7762ed28954be72e5ff6df9808e0779d4f1bf014ecc4e7e211d31ee34
RUN --mount=type=cache,target=/downloads \
    fetch https://github.com/llvm/llvm-project/releases/download/llvmorg-${LLVM_VERSION}/llvm-project-${LLVM_VERSION}.src.tar.xz \
          ${LLVM_SHA256} llvm-project-${LLVM_VERSION}.src.tar.xz \
 && mkdir /src \
 && tar -xJf /downloads/llvm-project-${LLVM_VERSION}.src.tar.xz -C /src --strip-components=1

# ---- The fixes ---------------------------------------------------------------------------------
COPY patches /patches
RUN cd /src && for p in /patches/*.patch; do [ -e "$p" ] || continue; echo "applying $p"; patch -p1 --forward < "$p"; done

# ---- Configure and build lldb-server only ------------------------------------------------------
# RelWithDebInfo: optimized like a release, with DWARF for the debug copy. The native tablegen tools are built with
# g++ (CROSS_TOOLCHAIN_FLAGS_NATIVE). -static links libc++, libunwind and the mingw runtime into the .exe;
# --no-insert-timestamp leaves the link time out of the PE header, so every build gives the same file.
# COMPILE_JOBS: a compiler with debug info needs up to ~1.3 GB (clang's Sema); one per core ran WSL's default memory
# (half the RAM, 7.8 GB on a 16 GB computer) out and the kernel killed it.
ARG COMPILE_JOBS=6
RUN cmake -G Ninja -S /src/llvm -B /build \
      -DLLVM_PARALLEL_COMPILE_JOBS=${COMPILE_JOBS} -DLLVM_PARALLEL_LINK_JOBS=1 \
      -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_SYSTEM_NAME=Windows \
      -DCMAKE_C_COMPILER=x86_64-w64-mingw32-clang \
      -DCMAKE_CXX_COMPILER=x86_64-w64-mingw32-clang++ \
      -DCMAKE_RC_COMPILER=x86_64-w64-mingw32-windres \
      -DCMAKE_FIND_ROOT_PATH=/opt/llvm-mingw/x86_64-w64-mingw32 \
      -DCMAKE_FIND_ROOT_PATH_MODE_PROGRAM=NEVER \
      -DCMAKE_FIND_ROOT_PATH_MODE_LIBRARY=ONLY \
      -DCMAKE_FIND_ROOT_PATH_MODE_INCLUDE=ONLY \
      "-DCMAKE_EXE_LINKER_FLAGS=-static -Wl,--no-insert-timestamp" \
      "-DCROSS_TOOLCHAIN_FLAGS_NATIVE=-DCMAKE_C_COMPILER=gcc;-DCMAKE_CXX_COMPILER=g++" \
      -DLLVM_HOST_TRIPLE=x86_64-w64-windows-gnu \
      -DLLVM_ENABLE_PROJECTS="clang;lldb" \
      -DLLVM_TARGETS_TO_BUILD=X86 \
      -DLLVM_ENABLE_ASSERTIONS=OFF \
      -DLLVM_ENABLE_ZLIB=OFF -DLLVM_ENABLE_ZSTD=OFF -DLLVM_ENABLE_LIBXML2=OFF \
      -DLLDB_ENABLE_PYTHON=OFF -DLLDB_ENABLE_LUA=OFF -DLLDB_ENABLE_LIBEDIT=OFF -DLLDB_ENABLE_CURSES=OFF \
      -DLLDB_ENABLE_LZMA=OFF -DLLDB_ENABLE_LIBXML2=OFF \
      -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF \
      -DCLANG_INCLUDE_TESTS=OFF -DLLDB_INCLUDE_TESTS=OFF \
 && cmake --build /build --target lldb-server

# ---- Result --------------------------------------------------------------------------------------
RUN mkdir /out \
 && cp /build/bin/lldb-server.exe /out/lldb-server-debug.exe \
 && x86_64-w64-mingw32-strip -o /out/lldb-server.exe /build/bin/lldb-server.exe \
 && { echo "llvm-project ${LLVM_VERSION}"; echo "llvm-mingw ${LLVM_MINGW}"; \
      for p in /patches/*.patch; do [ -e "$p" ] && echo "patch $(basename "$p") $(sha256sum < "$p" | cut -d' ' -f1)"; done; \
      true; } > /out/versions.txt \
 && mkdir -p /out/licenses/llvm-mingw \
 && cp /src/llvm/LICENSE.TXT /out/licenses/llvm.txt \
 && cp /src/libcxx/LICENSE.TXT /out/licenses/libcxx.txt \
 && cp /src/libunwind/LICENSE.TXT /out/licenses/libunwind.txt \
 && cp /opt/llvm-mingw/LICENSE.TXT /out/licenses/llvm-mingw/ \
 && cp /opt/llvm-mingw/x86_64-w64-mingw32/share/mingw32/COPYING* /out/licenses/llvm-mingw/ \
 && cd /out && sha256sum lldb-server.exe lldb-server-debug.exe > SHA256SUMS \
 && x86_64-w64-mingw32-objdump -p lldb-server.exe | grep 'DLL Name' \
 && cat versions.txt SHA256SUMS
