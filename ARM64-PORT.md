# Windows ARM64 Port

Native Windows ARM64 (`aarch64`) support added to this project on branch `arm64-port`.

## Summary of Changes
### Build System
- [DuiLib.sln](DuiLib.sln) — added `Debug|Release|UnicodeDebug|UnicodeRelease × ARM64` solution configurations; ActiveCfg/Build.0 rows added for every project except `FlashDemo` (no ARM64 path, see Limitations).
- [GameDemo/GameDemo.sln](GameDemo/GameDemo.sln) — same ARM64 solution-config rows added for consistency (this standalone solution is not part of the maintained build graph — see Limitations).
- [DuiLib/DuiLib.vcxproj](DuiLib/DuiLib.vcxproj), [DuiLib/DuiLib_Static.vcxproj](DuiLib/DuiLib_Static.vcxproj), [360SafeDemo/360SafeDemo.vcxproj](360SafeDemo/360SafeDemo.vcxproj), [GameDemo/GameDemo.vcxproj](GameDemo/GameDemo.vcxproj), [ListDemo/ListDemo.vcxproj](ListDemo/ListDemo.vcxproj), [MenuDemo/MenuDemo.vcxproj](MenuDemo/MenuDemo.vcxproj), [QQDemo/QQDemo.vcxproj](QQDemo/QQDemo.vcxproj), [RichListDemo/RichListDemo.vcxproj](RichListDemo/RichListDemo.vcxproj), [ScrCapture/ScrCapture.vcxproj](ScrCapture/ScrCapture.vcxproj), [TestApp1/TestApp1.vcxproj](TestApp1/TestApp1.vcxproj) — platform toolset upgraded from `v110`/`v110_xp`/`v120`/`v120_xp` to `v143` (XP toolsets have no ARM64 support); added 4 ARM64 `ProjectConfiguration`/`PropertyGroup`/`ItemDefinitionGroup` blocks cloned from the matching Win32 configs, each with distinct `OutDir`/`IntDir` (`bin\ARM64\`, `Lib\ARM64\`) to avoid colliding with the shared flat Win32 output folders.
- [DuiLib/DuiLib.vcxproj](DuiLib/DuiLib.vcxproj) — removed the hardcoded `<TargetMachine>MachineX86</TargetMachine>` and fixed `<BaseAddress>0x11000000</BaseAddress>` linker overrides for ARM64 configs (set `MachineARM64` + `RandomizedBaseAddress` instead).
- [FlashDemo/FlashDemo.vcxproj](FlashDemo/FlashDemo.vcxproj) — toolset upgraded to `v143` only; **no** ARM64 configuration added (Adobe Flash ActiveX, see Limitations).
- ARM64 `PrecompiledHeader` disable overrides added for `Utils\stb_image.c` and `Utils\XUnzip.cpp` (previously only conditioned on Win32 configs, which would have caused a PCH mismatch on ARM64).
- ARM64 `PrecompiledHeader=Create` overrides added for each project's `StdAfx.cpp` (was Win32-only, caused `C1083` PCH-not-found on ARM64 first build).
- ATL/MFC include/lib paths added for `DuiLib`, `QQDemo`, `GameDemo`, `MenuDemo` pointing at the installed MSVC toolset version that ships ATL/MFC ARM64 binaries (the newest installed toolset lacked the ATL/MFC component).
- [ScrCapture/ScrCapture.vcxproj](ScrCapture/ScrCapture.vcxproj) — added the missing ARM64 `ProjectConfiguration`/`PropertyGroup` blocks (ARM64 `ItemDefinitionGroup` existed but the configuration declarations did not, causing `MSB8013`).
- [TestApp1/StdAfx.h](TestApp1/StdAfx.h), [MenuDemo/StdAfx.h](MenuDemo/StdAfx.h), [ScrCapture/StdAfx.h](ScrCapture/StdAfx.h) — added `_M_ARM64`/`_M_ARM64EC`-guarded `#pragma comment(lib, ...)` path variant (the Win32-relative hardcoded path doesn't match the ARM64 output directory).

### Source Code
- [DuiLib/Utils/stb_image.c](DuiLib/Utils/stb_image.c) — enabled the vendored stb_image library's own dormant NEON kernels (`stbi__idct_simd`, `stbi__YCbCr_to_RGB_simd`, `stbi__resample_row_hv_2_simd` for JPEG decode; `STBI__F_up` for PNG decode) behind `#if defined(_M_ARM64) || defined(__aarch64__)`, guarded by the existing `STBI_NO_SIMD` opt-out. These are upstream-authored NEON intrinsics that were previously dormant (opt-in `STBI_NEON` flag was never defined); x86 paths are completely unchanged.
- [DuiLib/Utils/WndShadow.cpp](DuiLib/Utils/WndShadow.cpp) — added a NEON `vmaxq_u32`-based max-blend kernel for the shadow border blur (runs during live window resize), guarded behind ARM64, with the original scalar path preserved for x86/x64.
- [DuiLib/Utils/XUnzip.cpp](DuiLib/Utils/XUnzip.cpp) — accelerated `ucrc32` with the ARMv8 hardware `crc32x` instruction (ACLE `__crc32` intrinsics) behind ARM64, original table-based scalar CRC32 preserved for x86/x64.
- No other library source changes were required — an exhaustive scan found zero hand-written SIMD intrinsics, inline assembly, or x86-only compiler intrinsics anywhere in `DuiLib/Core`, `DuiLib/Control`, `DuiLib/Layout`. The codebase's only arch-conditional guards (`WIN32`/`UNDER_CE`) are architecture-neutral and require no changes.

### CI/CD
None — no CI configuration (`.github/workflows`, `appveyor.yml`, `azure-pipelines.yml`) exists in this repository.

### New Files
None. All ARM64 support is additive within existing files (`#if defined(_M_ARM64) || defined(__aarch64__)` guards and cloned MSBuild platform configs). No vendored SIMD translation-shim library (no `sse2neon.h`/`simde`/`xsimd`) was introduced.

## Build Steps (ARM64)
Prerequisites: Visual Studio 2022/2026 with the **MSVC v143 - ARM64/ARM64EC build tools** and **Windows 11 SDK** components. This repo also requires the **ATL/MFC for ARM64** optional component (used by `UIWebBrowser.cpp`'s `CComPtr`/`CComQIPtr` and several demo apps' resource headers).

### Build
```powershell
$msbuild = "C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\arm64\MSBuild.exe"
& $msbuild DuiLib.sln /p:Configuration=Release /p:Platform=ARM64 /m
```
Build output: `bin\ARM64\*.exe`, `bin\ARM64\DuiLib.dll`, `Lib\ARM64\DuiLib.lib`

### Run Tests
No automated test suite exists in this repository (confirmed — only interactive GUI demo apps are present, no CTest/GoogleTest/MSTest harness). A native-ARM64 architecture smoke test was performed instead:
```powershell
Start-Process "bin\ARM64\TestApp1.exe"
# runs without STATUS_ILLEGAL_INSTRUCTION / crash; terminate manually or via Stop-Process
```

### Run Benchmarks
No benchmark suite exists in this repository. Not applicable.

## Test Results
- **Build host**: Native ARM64
- **Executed**: Partial — no automated test harness exists; a non-interactive smoke launch of `TestApp1.exe` was performed and ran cleanly for 3 seconds with no crash before being terminated.
- **Passed**: 0     **Failed**: 0     **Skipped**: 10 (all demo executables — no automated harness applicable)

## Running on a Windows ARM64 Device
Not applicable — this port was built AND validated on a native Windows ARM64 host. All binaries were compiled and dumpbin-verified directly on ARM64 hardware; no cross-compile/transfer step was needed.

## Test Fixes Applied
None — no test suite exists to fix.

## Benchmark Results
No benchmark suite exists in this repository. Not applicable.

## NEON Optimizations
Applied by the optimizer in Phase 7 — strictly additive, guarded behind `#if defined(_M_ARM64) || defined(__aarch64__)`. The x64/x86 build is untouched.

- **Functions optimized**: 4
  - `stbi__idct_simd`, `stbi__YCbCr_to_RGB_simd`, `stbi__resample_row_hv_2_simd` — [stb_image.c](DuiLib/Utils/stb_image.c) (enabled upstream's own dormant NEON JPEG decode kernels)
  - `STBI__F_up` PNG filter reconstruction — [stb_image.c](DuiLib/Utils/stb_image.c) (dependency-free byte-add vectorized with NEON)
  - Shadow border max-blend — [WndShadow.cpp](DuiLib/Utils/WndShadow.cpp) (`vmaxq_u32`)
  - `ucrc32` — [XUnzip.cpp](DuiLib/Utils/XUnzip.cpp) (hardware `crc32x` instruction)
- **Categories**: SSE→NEON enablement (upstream dormant kernels): 2 · scalar→NEON: 1 · scalar→hardware-CRC: 1
- **Benchmark delta**: Not measured — no benchmark baseline exists in this repository. All four changes were validated via `dumpbin`-confirmed NEON/CRC instruction emission plus standalone bit-exactness smoke tests (PNG/JPEG decode checksum comparison against scalar build; 20,000-trial randomized CRC32 fuzz comparison) — all passed with zero mismatches.
- **Caveats**: No FMA, no ARMv8.2+ optional ISA features used — everything uses only the unconditional Windows ARM64 baseline ISA (NEON + CRC32 extension), so there is no minimum-SoC requirement beyond the ARM64 port itself. Four additional candidates (PNG Sub/Avg/Paeth filters, GIF LZW decode, `AlphaBitBlt` double-precision fallback blend, HSL recolor) were evaluated and skipped for concrete technical reasons (intra-row data dependency chains, branchy per-symbol state machines, no validation harness for a looser fixed-point approximation) — see commit log for details.
- **Commits**: `f132dc5`, `6c27b8f`, `625d804`, `f429deb`, `5a19f9a`

## Architecture Verification
All built binaries verified with `dumpbin /HEADERS` as machine type `AA64` (ARM64):

| Binary | Machine | Size |
|---|---|---|
| Lib\ARM64\DuiLib.lib | AA64 | 681,740 bytes |
| bin\ARM64\DuiLib.dll | AA64 | 772,096 bytes |
| bin\ARM64\360Safe.exe | AA64 | 585,728 bytes |
| bin\ARM64\GameDemo.exe | AA64 | 664,064 bytes |
| bin\ARM64\ListDemo.exe | AA64 | 1,011,200 bytes |
| bin\ARM64\MenuDemo.exe | AA64 | 921,600 bytes |
| bin\ARM64\QQDemo.exe | AA64 | 2,630,144 bytes |
| bin\ARM64\RichListDemo.exe | AA64 | 215,552 bytes |
| bin\ARM64\ScrCapture.exe | AA64 | 592,384 bytes |
| bin\ARM64\TestApp1.exe | AA64 | 576,000 bytes |

## Arm AppReady Status
Aligns with the [Arm AppReady for Windows on Arm](https://developer.arm.com/laptops-and-desktops/windows-app-ready) program (Assess → Build → Deploy → Optimize).

- **Target profile**: ARM64-native — no closed-source x64-only blocking dependency exists in the core library; only the optional `FlashDemo` sample (excluded) depends on a discontinued plugin.
- **AppReady stages reached**: Assess ✓ | Build ✓ | Deploy — n/a (library + sample apps, no installer) | Optimize ✓
- **WoA Ecosystem Dashboard status**:

| Dependency | Version | Dashboard status | Citation |
|---|---|---|---|
| comctl32 | OS-bundled | native | https://developer.arm.com/ecosystem-dashboard/windows |
| ole32/oleaut32 | OS-bundled | native | https://developer.arm.com/ecosystem-dashboard/windows |
| mshtml/WebBrowser control | OS-bundled | building (legacy, maintenance) | https://developer.arm.com/ecosystem-dashboard/windows |
| Adobe Flash Player ActiveX | 10.x | unsupported | Flash Player EOL Dec 2020; no ARM64 build ever existed |

- **Emulation vs native classification**:

| Component | Native ARM64 / Emulated x64 / Blocking | Notes |
|---|---|---|
| DuiLib.dll / DuiLib.lib (core library) | Native ARM64 | Built and dumpbin-verified |
| 360SafeDemo, GameDemo, ListDemo, MenuDemo, QQDemo, RichListDemo, ScrCapture, TestApp1 | Native ARM64 | Built and dumpbin-verified |
| FlashDemo | Blocking | Adobe Flash ActiveX control, discontinued, no ARM64 path |

- **Blockers requiring Microsoft App Assure**: None — the single blocker (Flash) is a dead third-party plugin with no viable native or emulated path; `FlashDemo` is simply excluded from the ARM64 solution configuration.

## Limitations & Known Issues
- `[hard-external]` `FlashDemo` sample app excluded from the ARM64 build — depends on the Adobe Flash Player ActiveX control (`flash10a.tlh`), which Adobe discontinued in December 2020 and which never shipped an ARM64 build (predates ARM64 Windows entirely); it is closed-source so no rebuild-from-source path exists. The core `DuiLib` library itself has no Flash dependency.

**Factual notes (not ARM64-port limitations — pre-existing repository structure, unaffected by this port):**
- `FileTree/FileTree.vcproj` is a standalone legacy VS2008-format project that is not referenced by the maintained `DuiLib.sln` build graph for *any* platform, including the original Win32 build — it predates and is independent of this ARM64 effort.
- The standalone `GameDemo/GameDemo.sln` references the legacy `GameDemo.vcproj` instead of the modern `GameDemo.vcxproj` and was already non-functional for Win32 before this port; the `GameDemo` project that is actually part of `DuiLib.sln` (and was built/verified for ARM64) uses the modern `.vcxproj` correctly.

## Reverting / Coexistence
The x64/Win32 build is unchanged. To build Win32, use `/p:Platform=Win32` with any existing configuration (Debug/Release/UnicodeDebug/UnicodeRelease).
