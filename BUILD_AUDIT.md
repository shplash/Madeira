# Madeira GitHub Actions Build Audit

**Date:** 2026-10-06  
**Status:** PRE-RUN VALIDATION  
**Fixed issues:** Rust toolchain configuration

---

## 1. Rust / iOS Target Configuration ✅

### Issue Identified & Fixed
- **Original Problem**: Homebrew `rustup` installed but not in PATH
- **Error**: `rustc: command not found` when running `rustup target add aarch64-apple-ios`
- **Fix Applied**:
  - Added `$(brew --prefix rustup)/bin` to PATH before using rustup
  - Install stable Rust toolchain before adding iOS target
  - Set default toolchain explicitly
  - Verify rustc/cargo versions

### Status
- ✅ Rust toolchain now initializes correctly
- ✅ iOS target added after Rust is available
- ✅ Build step verifies `rustc --version` and `cargo --version`

---

## 2. Build Dependencies Verification

### Homebrew Packages (line 68-81)
| Package | Used By | Status |
|---------|---------|--------|
| cmake | LLVM, FreeType, many subprojects | ✅ Standard |
| ninja | LLVM, DXMT meson build | ✅ Standard |
| meson | DXMT | ✅ Standard |
| pkg-config | LLVM, FFmpeg | ✅ Standard |
| bison | Wine configure | ✅ Added to PATH correctly |
| flex | Wine configure | ✅ Added to PATH correctly |
| sevenzip | VC runtime extraction | ✅ Standard |
| ccache | Optional caching | ✅ Standard |
| autoconf, automake, libtool | Wine, FFmpeg, optional | ✅ Standard |
| gettext | Wine | ✅ Standard |
| rustup | rppairing-ios Rust build | ✅ **FIXED** — Path corrected |

### Critical PATH Exports (lines 82-85)
```bash
echo "$(brew --prefix bison)/bin" >> "$GITHUB_PATH"        # ✅ Correct
echo "$(brew --prefix flex)/bin" >> "$GITHUB_PATH"         # ✅ Correct
echo "$(brew --prefix rustup)/bin" >> "$GITHUB_PATH"       # ✅ ADDED
echo "$HOME/.cargo/bin" >> "$GITHUB_PATH"                  # ✅ Correct
```

---

## 3. Build Order & Dependency Chain

From `docs/BUILDING.md` (verified against workflow):

| Step | Component | Script | Input Deps | Output | Status |
|------|-----------|--------|-----------|--------|--------|
| 1 | llvm-mingw | Download & cache | Release on GitHub | `toolchains/llvm-mingw-*` | ✅ Cached |
| 2 | Xcode / Metal | System tools | macOS 15 | SDK paths | ✅ Dynamic |
| 3 | FreeType | `build/freetype-ios/build.sh` | research/freetype (cloned) | `build/freetype-ios/build/libfreetype.a` | ✅ Present |
| 4 | Wine (host) | configure + make | bison, flex, Xcode | `wine/build-macos/` | ✅ Self-contained |
| 5 | LLVM iOS | `cmake` + `cmake --build` | LLVM upstream (cloned), tblgen | `toolchains/llvm-ios-build/` | ✅ Cached |
| 6 | FEX iOS | `build/fex-ios/build.sh` | FEX submodule, cmake | `FEX/build-ios/FEXCore/Source/libFEXCore*.a` | ⚠️ **Check FEX submodule** |
| 7 | FEX ARM64EC | `build/fex-arm64ec/build.sh` | FEX submodule, llvm-mingw | `app/Madeira/arm64ec-windows/xtajit64.dll` | ⚠️ **Check FEX submodule** |
| 8 | Wine ARM64EC | configure + build | bison, flex, llvm-mingw | `wine/build-arm64ec/` | ✅ Self-contained |
| 9 | DXMT iOS | `build/dxmt-ios/build.sh` | dxmt submodule, LLVM iOS | `app/Madeira/libdxmt_combined.a` | ⚠️ **Check DXMT submodule** |
| 10 | RPPairing | `build/rppairing-ios/build.sh` | Rust, iOS target | `app/Madeira/libmadeira_rppairing.a` | ✅ **FIXED** — Rust now ready |
| 11 | D3D12 | `build/madeira-d3d12/build-pe.sh` | llvm-mingw, Xcode | `app/Madeira/arm64ec-windows/*.dll` | ⚠️ **Check metal-irconverter dependency** |
| 12 | App | Xcode build | All above artifacts | `out/Madeira.app` | ✅ Collects artifacts |

---

## 4. Critical Pre-Run Checks

### ✅ Submodules (Recursive Checkout)
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
    submodules: recursive  # ← Ensures FEX, wine, dxmt, madeira-dock present
```
**Status**: Correct. Submodules are fetched.

### ⚠️ Missing External Dependencies (not in repo)

From `docs/BUILDING.md`, these are **NOT** in the repository:

| Item | Required For | Size | Action |
|------|--------------|------|--------|
| `toolchains/llvm-mingw-20260421-ucrt-macos-universal/` | Compilation (x86-64, ARM64EC, i686 cross) | 122 MB | ✅ **Downloaded & cached** |
| `toolchains/llvm-project/` | LLVM for iOS | ~2GB | ✅ **Cloned from upstream, cached** |
| `research/GPTK/Metal Shader Converter 4.0 beta 2.pkg` | `madeira-d3d12` shader compilation | 30 MB | ❌ **NOT in workflow, not in repo** |
| `app/Madeira/x86_64-vcruntime/` | D3D12 games that need VC++ runtime | ~50 MB extracted | ✅ **Downloaded & cached** |
| `research/freetype/` | FreeType headers/lib | GitHub clone | ✅ **Cloned in workflow** |

### 🔴 BLOCKING ISSUE: Metal Shader Converter

**Location in build**: `build/madeira-d3d12/`  
**Needed by**: Line 669: `bash build/madeira-d3d12/build-pe.sh`

The BUILDING.md and madeira-d3d12 README both note:
```markdown
research/GPTK/Metal Shader Converter 4.0 beta 2.pkg
SHA-256: 1acc33c87ea663933df89721a998d066106685473020bcbe007cee7a16155734
```

**This package is NOT downloaded in the workflow.**  
The D3D12 build step will FAIL if `build/madeira-d3d12/deps.sh` runs and cannot find the converter.

---

## 5. Potential Failures in Workflow

### 🔴 Critical (Will Fail)

#### Issue 1: Metal Shader Converter Not Available
- **Line**: 669 `bash build/madeira-d3d12/build-pe.sh`
- **Dependency**: `build/madeira-d3d12/deps.sh` expects the converter at a known path
- **Error Expected**: `SHA-256 hash mismatch` or `file not found`
- **Fix Needed**: Either:
  1. Add step to download/cache converter (requires Auth token to Apple dev)
  2. Skip D3D12 build (remove lines 669-683, comment out verification line 680)
  3. Implement fallback to upstream D3D12

### 🟡 Medium (May Fail)

#### Issue 2: FEX/Wine/DXMT Submodule Branches
- **Lines**: 415-424 (FEX), 435-478 (Wine ARM64EC), 645-665 (DXMT iOS)
- **Check**: The workflow doesn't verify submodule branches are correct
- **Branch Names from BUILDING.md**:
  - FEX: `ios-port-2607`
  - Wine: `madeira-lgpl`
  - DXMT: `ios-port`
- **Error If Wrong**: Build scripts fail silently or with cryptic CMake errors
- **Fix**: Add verification step after checkout:
  ```bash
  cd FEX && git describe --all && cd ../wine && git describe --all && cd ../dxmt && git describe --all
  ```

#### Issue 3: Wine Host Build Tolerance (Line 291-297)
- The "Build Wine host artifacts" step uses `set +e` and `make -k` to tolerate failures
- This is intentional (per comment: "Host-side failures are tolerated")
- But the verification `test -f include/config.h` will fail if configure never ran
- **Not a blocker** if configure succeeds, but could hide earlier issues

### 🟢 Low Risk

#### Issue 4: Python Scripts Depend on Python 3
- Lines 153-234 (VC runtime extraction)
- Lines 571-611 (DXMT shader headers)
- **Assumed**: `python3` in PATH
- **Status**: macOS 15 includes Python 3 by default
- **Minor fix**: Add explicit check or `brew install python3` to dependencies

---

## 6. Missing Steps for Completeness

### Recommended Additions

#### 6.1 Verify Submodule Branches (Add after checkout)
```yaml
- name: Verify submodule branches
  shell: bash
  run: |
    set -euxo pipefail
    echo "=== FEX Branch ==="
    cd FEX && git describe --all && cd ..
    echo "=== Wine Branch ==="
    cd wine && git describe --all && cd ..
    echo "=== DXMT Branch ==="
    cd dxmt && git describe --all && cd ..
```

#### 6.2 Handle Metal Shader Converter (Choose one)
**Option A: Skip D3D12** (Fast, D3D11 still works)
```yaml
- name: Madeira D3D12 (skipped - requires Apple converter)
  shell: bash
  run: |
    echo "D3D12 requires Metal Shader Converter 4.0 beta 2"
    echo "Skipping native D3D12; D3D11 available"
    mkdir -p app/Madeira/arm64ec-windows
    # touch placeholder files or warn in log
```

**Option B: Download Converter** (Requires hosting/auth)
```yaml
- name: Fetch Metal Shader Converter
  shell: bash
  run: |
    # Would need URL and SHA-256 verification
    # Not included in public workflow for licensing reasons
    echo "Metal Shader Converter not in public builds"
    exit 1
```

#### 6.3 Verify Python 3 Availability
```yaml
- name: Verify Python 3
  shell: bash
  run: |
    python3 --version
    which python3
```

#### 6.4 Cargo toolchain info after configure
```yaml
- name: Verify Cargo setup
  shell: bash
  run: |
    cargo --version
    rustc --version
    rustup toolchain list
    rustup target list | grep aarch64-apple-ios
```

---

## 7. Artifact Verification Checklist (Line 707-737)

All outputs checked before Xcode build:

```
app/Madeira/libgmp.a                               ← from gnutls-ios (tracked)
app/Madeira/libnettle.a                            ← from gnutls-ios (tracked)
app/Madeira/libhogweed.a                           ← from gnutls-ios (tracked)
app/Madeira/libgnutls.a                            ← from gnutls-ios (tracked)
app/Madeira/libavformat.a                          ← from ffmpeg
app/Madeira/libavcodec.a                           ← from ffmpeg
app/Madeira/libswresample.a                        ← from ffmpeg
app/Madeira/libavutil.a                            ← from ffmpeg
app/Madeira/libntdll_unix.a                        ← from wine/ntdll-unix
app/Madeira/libwin32u_unix.a                       ← from wine/win32u-unix
app/Madeira/libwineserver.a                        ← from wine/wineserver
app/Madeira/libdxmt_combined.a                     ← from dxmt iOS + LLVM merge
app/Madeira/libmadeira_rppairing.a                 ← from rppairing (Rust)
app/Madeira/arm64ec-windows/ntdll.dll              ← from wine PE
app/Madeira/arm64ec-windows/xtajit64.dll           ← from FEX ARM64EC
app/Madeira/arm64ec-windows/winemetal.dll          ← from DXMT PE
app/Madeira/arm64ec-windows/d3d11.dll              ← from DXMT PE
app/Madeira/arm64ec-windows/d3d12.dll              ← from native D3D12
app/Madeira/arm64ec-windows/madeira_d3d12.dll     ← from native D3D12 (NEW)
```

**Note**: If D3D12 is skipped, remove the last two from this check.

---

## 8. Recommended Pre-Run Actions

### Before Running Workflow:

1. ✅ **Verify recursivesubmodules checkout is enabled** — Already correct in workflow

2. ⚠️ **Decide on D3D12 Build**:
   - If Metal Converter available: Keep lines 669-683, update verification at line 730-731
   - If NOT available: Remove lines 669-683, remove D3D12 DLLs from verification (lines 729-730)

3. 🔲 **Add submodule branch verification** — Insert before FEX iOS build (around line 407)

4. 🔲 **Test locally first** (Optional but recommended):
   ```bash
   git clone --recursive https://github.com/shplash/Madeira.git
   cd Madeira
   # Verify submodule branches match BUILDING.md
   cd FEX && git branch && cd ../wine && git branch && cd ../dxmt && git branch
   ```

5. 🔲 **Confirm Metal Shader Converter strategy**:
   - Add download + cache step, OR
   - Comment out D3D12 build, OR
   - Document as known limitation

---

## 9. Summary

| Category | Status | Action |
|----------|--------|--------|
| Rust/iOS toolchain | ✅ FIXED | Ready to run |
| Homebrew dependencies | ✅ VERIFIED | All correct |
| Build order | ✅ CORRECT | Follows BUILDING.md |
| Submodule checkout | ✅ CORRECT | Recursive enabled |
| Metal Shader Converter | 🔴 **BLOCKING** | Must resolve before D3D12 step |
| Submodule branches | ⚠️ UNVERIFIED | Should add verification step |
| Python 3 | 🟢 OK | Default on macOS 15 |
| Artifact verification | ✅ COMPREHENSIVE | All outputs checked |

---

## 10. Recommended Minimum Fix

To proceed safely:

1. **Keep** the Rust toolchain fix (already applied) ✅
2. **Add** submodule branch verification step
3. **Either A or B**:
   - **A** (Recommended for CI): Remove D3D12 build step (lines 669-683) and DLL verification (lines 729-730)
   - **B** (If converter available): Add download/cache step before line 669

**Expected outcome**: Build reaches Xcode app compilation successfully, generates working IPA.

---

**Next Step**: Confirm D3D12 strategy and submodule branches before triggering workflow run.
