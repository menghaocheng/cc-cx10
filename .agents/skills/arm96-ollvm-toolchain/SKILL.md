---
name: arm96-ollvm-toolchain
description: >-
  Build, restore, or troubleshoot the OLLVM code-obfuscation toolchain
  (clang-14 + ObfusPass CFF/SUB/BCF plugin) used to compile the arm96 loader and
  arm96server. Use this whenever OBFUSCATE_LOADER=1 builds fall back to gcc,
  obfuscation appears to be missing/inert, the ObfusPass plugin crashes clang
  ("clang frontend command failed due to signal"), clang-14 / llvm-14-dev are
  absent, libObfusPass.so ABI-mismatches the clang in use, you are setting up the
  arm96 build on a NEW/different build server, or you need to confirm OLLVM is
  actually being applied to the loader. Pushy reminder: reach for this skill any
  time the words OLLVM, ObfusPass, OBFUSCATE_LOADER, clang-14, libObfusPass.so, or
  "obfuscation toolchain" come up in the arm96 hardening project, even if the user
  hasn't named the skill — the failure mode here is SILENT (gcc fallback ships a
  loader with zero obfuscation), so it must be checked deliberately.
---

# arm96 OLLVM toolchain (clang-14 + ObfusPass)

## Why this skill exists

`OBFUSCATE_LOADER=1` is meant to compile `arm96` (loader) and `arm96server` with
**clang-14 + the ObfusPass plugin** (CFF control-flow flattening, SUB instruction
substitution, BCF bogus control flow). But the build script historically
**fell back to plain gcc -O2 — zero obfuscation — silently** whenever either:

1. **`clang-14` is not on `PATH`** (the build uses the bare name `clang-14`, but on
   this project clang-14 ships only as an **AOSP prebuilt**, not an apt package), or
2. **`libObfusPass.so` is ABI-mismatched** to the clang loading it. The plugin must
   be built against the *same* LLVM as the clang that runs it. A plugin built
   against apt `llvm-14-dev` loaded into the AOSP prebuilt clang-14 (Android's LLVM
   fork) makes the clang frontend **SIGSEGV** ("clang frontend command failed due
   to signal").

Because the fallback was silent, OLLVM stayed **off for several releases**
(V0.69→V0.72.x: loader = gcc -O2, no obfuscation) without anyone noticing. This
skill makes the toolchain **reproducible on any build machine without sudo**, and
the build is now **fail-loud** (see "Fail-loud guard" below) so this can't regress
silently again.

## The one command

Everything is automated by a single, version-controlled, idempotent script:

```sh
cd <repo>/android10/vendor/hello/arm96
bash hardening/setup_ollvm_toolchain.sh          # detect → rebuild plugin → validate → emit env
bash hardening/setup_ollvm_toolchain.sh --check  # validate only, don't rebuild the plugin
```

On success it prints `OLLVM active (.text +NN% over baseline)` and writes
`build/ollvm_toolchain.env`. Run it **once per build machine** (and again after a
`clang-r*` prebuilt changes or `build/` is wiped). Then build normally:
`./make.sh --build cx`.

This script is the canonical setup mechanism — prefer running it over hand-rolling
clang/plugin commands, so the ABI-matching and validation logic stay in one place.

## What the script does (and what to check if it fails)

It runs on the **build machine** (the Linux compile host for the current network —
see `.Codex/instructions/build_and_debug*.md`; the repo path is the mount of that
host). The five steps, in order — when something breaks, this is the order to debug:

1. **Detect clang-14.** Prefer `command -v clang-14`; otherwise the newest AOSP
   prebuilt `android10/prebuilts/clang/host/linux-x86/clang-r*/bin/clang-14`.
   `clang++` and `llvm-config` are taken from the **same `bin/`** so the plugin is
   ABI-matched by construction.
   *Fails if:* neither source has clang-14. Fix: ensure the AOSP `prebuilts/clang`
   tree is present, or install apt `clang-14`.

2. **Detect gcc-cross for static crt.** clang needs `--gcc-toolchain=<prefix>` to
   find `crtbeginT.o` / `libgcc.a` for `-static`. The script looks under
   `/usr/lib/gcc-cross/aarch64-linux-gnu/*/`.
   *Fails if:* no aarch64 gcc-cross. Fix: `g++-aarch64-linux-gnu` must be installed.

3. **Rebuild the plugin ABI-matched.** Compiles `hardening/obfus_pass/ObfusPass.cpp`
   directly with `clang++ $(llvm-config --cxxflags) -std=c++17 -fPIC -shared`
   (the cmake project isn't needed and cmake is often absent on the host). The
   `llvm-config --cxxflags` carries the critical `-fno-rtti` / stdlib flags that
   must match clang. Output overwrites `hardening/obfus_pass/build/libObfusPass.so`
   (the path the build expects).
   *Fails if:* `llvm-config` missing next to clang, or the .cpp doesn't compile
   against that LLVM's API.

4. **Validate (no silent inertness).** Compiles `loader.c` *with* and *without* the
   plugin and asserts the OLLVM build's `.text` is **larger** (CFF/BCF inflate
   annotated functions, typically +30–40%). A plugin that loads but transforms
   nothing is treated as a failure — we refuse to claim OLLVM is active when it
   isn't.
   *Fails if:* clang+plugin crashes (ABI mismatch → step 3 used wrong LLVM), or the
   `.text` delta is zero. Needs `gen/` to exist (run `./make.sh` once first).

5. **Emit `build/ollvm_toolchain.env`** with `OBFUS_CC`, `OBFUS_PASS_SO`,
   `OBFUS_GCC_TOOLCHAIN` for the build pipeline to source.

## How the build consumes it (fail-loud guard)

`build_arm96_runtime_artifacts.sh` sources `build/ollvm_toolchain.env` so both
arm96server and the loader use the resolved clang + ABI-matched plugin +
`--gcc-toolchain`. When `OBFUSCATE_LOADER=1` but the toolchain isn't ready, the
build now **errors out (`exit 21`)** pointing back to this script — it no longer
degrades to gcc. To build *intentionally* without obfuscation, set
`OBFUSCATE_LOADER=0`.

Verify OLLVM actually engaged in a real build by checking the log for these (and
the **absence** of `gcc fallback`):

```
[V0.40] arm96server: clang-14 + ObfusPass (CFF/SUB)
[V0.21] OBFUSCATE_LOADER=1 → clang-14 + ObfusPass plugin (CFF/SUB)
```

## Relationship to VMP

OLLVM (broad, all annotated functions) and VMP (deep, a few functions, eats the
loader's rx-cave) share the same RX segment. Turning OLLVM on **inflates `.text`
and shrinks the VMP cave** (≈40KB → ≈27KB usable; ≈12.8KB of that is the fixed VM
interpreter blob). VMP target functions deliberately turn OLLVM **off** (`noinline`)
so VMP is their sole layer; non-target functions keep OLLVM. Changing OLLVM on/off
materially changes the loader `.text`, which feeds Prong5 / sentinel hash / runtime
self-check — so after enabling OLLVM, re-run a **cold-boot** `verify_mustpass`
(container rebuild / reboot), not just a hot restart. See
`.Codex/context/arm96_loader_vmp_plan.md` and the `ollvm-silently-off` memory.

## Important: this is portable, but not committed-binary

The plugin `.so` is a build artifact, not a source-of-truth — never rely on a
checked-in `libObfusPass.so` being ABI-correct for the current machine. On any new
or changed build host, **run `setup_ollvm_toolchain.sh` first**. That's the whole
point of this skill: the *recipe* travels with the repo; the binary is regenerated
per machine.
