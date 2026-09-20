# DLSS 5 Neural Rendering on Linux — field notes

Getting NVIDIA **DLSS 5 Neural Rendering** (NGX *feature 18*) running under Proton on an
**RTX 4070 Ti (Ada)** — a card NVIDIA does not officially support for it, on an OS NVIDIA
does not support it on at all.

It works. These are the notes, including the parts that don't work and exactly why.

Tested 2026-09-18 → 20 on Nobara 44 (Fedora), KDE Plasma 6.7.3 Wayland, NVIDIA **595.84**,
Proton Experimental, RTX 4070 Ti.

---

## TL;DR

| Game | Graphics API | Result |
|---|---|---|
| FINAL FANTASY VII REBIRTH | D3D12, native DLSS | ✅ works — 45,000+ evaluates, 22–29 fps @ 5120×1440 |
| DuckStation (PS1 emulator) | **D3D11** + DLSS5-Feeder | ✅ works — **59.8 fps, 0 stalls** |
| DuckStation (PS1 emulator) | **D3D12** + DLSS5-Feeder | ⚠️ every call succeeds, output is **black** |
| Tomb Raider I–III Remastered | OpenGL (64-bit) + Feeder | ❌ `D3D12 fence -> GL semaphore import: FAILED` |
| FINAL FANTASY VIII Remastered | OpenGL (32-bit) + Feeder | ❌ same failure, cross-process |

**Conclusion: DLSS 5 works on Linux for Direct3D games, and is blocked for OpenGL games**
by one missing Wine feature — importing a D3D12 fence handle as a GL semaphore.

---

## The two hard-won facts

### 1. On Ada, the "tested" model is the one that cannot work

The Linux add-on ([NapXDD/addon-dlssnr-linux](https://github.com/NapXDD/addon-dlssnr-linux))
documents stock `dlssnr-310.8.0` (`sha256 e16bcf15…`) as its tested model. On Ada that returns:

```
nr-fwd: snippet init      => 0x1 (Success)
nr-fwd: CreateFeature(18) => 0xbad00001 (FAIL_FeatureNotSupported)
```

The model *loads* and then refuses the feature. The community **RTX40-patched** build
(`dlssnr-310.8.0-RTX40`, `sha256 4b8d19bc…`) works:

```
nr-fwd: CreateFeature(18) => 0x1 (Success)
nr-fwd: EvaluateFeature #45600 => 0x1 (Success)
```

Its "untested model" warning therefore fires on every launch and is unavoidable on Ada.

### 2. Driver dispatch is dead on Linux — the forwarder is the whole trick

Every NGX capability query for feature 18 returns:

```
feature 18 (neural rendering) -> 0xBAD0000C (OutOfDate)
*** unavailable until the driver is updated to 616.56 or newer ***
```

No such Linux driver exists (newest upstream: **615.71.09**). RenoDX's add-on and
DLSS5-Feeder's own neural path both rely on that dispatch and therefore cannot work.
`addon-dlssnr-linux` sidesteps it by driving the game-local model directly through a
forwarder DLL whose filename contains `nvngx.dll`. That is the only route that works today.

---

## Recipes

### A game with native DLSS (FF7 Rebirth) — the easy case

Beside the game exe:
`dxgi.dll` (ReShade ≥6.8 with add-on support) · `dlssnr-linux.addon64` ·
`nvngx.dll_nrfwd.dll` · `nvngx_dlssnr.dll` (**RTX40-patched** on Ada) · `nvngx_dlss.dll`

```
PROTON_ENABLE_NVAPI=1 WINEDLLOVERRIDES="dxgi=n,b" %command%
```

Enable DLSS or DLAA in-game — the pass runs off the DLSS-SR output, so it does nothing without it.
Settings live in `ReShade.ini` under `[ADDON_DLSSNR_LINUX]`, read **once at attach**, so ini edits
need a restart. The overlay is **its own tab**, not inside the Add-ons list.

### A game with no DLSS (emulators, old games) — DLSS5-Feeder

[DLSS5-Feeder](https://github.com/jlrouzies-fr/DLSS5-Feeder) synthesises the DLSS call the game
never makes, from ReShade's colour + depth + estimated motion vectors. `addon-dlssnr-linux` then
hooks *that* evaluate. Each covers the other's gap.

Required: a motion-vector provider shader, and a preset that enables it **before** the feed:

```ini
Techniques=Lumenite_Kernel@lumenite_Kernel.fx,DLSS5_Feed@DLSS5_Feed.fx
```
```ini
PreprocessorDefinitions=DLSS5_MV_PROVIDER=3
```

**Use D3D11, not D3D12.** On D3D12 the Feeder opens a "same-device" session and the output is
black — verified not to be the neural pass's fault (still black with NR disabled, and the add-on's
own *"what the model sees"* debug view is black too). On D3D11 it opens a `D3D11 cross-API` session
and renders correctly.

### 32-bit games — the `host64` helper (undocumented for this add-on)

NVIDIA ships **no 32-bit NGX runtime**, so the Feeder runs NGX in a separate 64-bit helper.
`dlssnr-linux.addon64` *can* serve as the neural consumer there — put **ReShade x64** beside the
helper as **`host64/dxgi.dll`** and it loads:

```
Loading add-on from '…\host64\dlssnr-linux.addon64' ...
Registered add-on "DLSSNR Linux" using ReShade API version 18.
ngx-probe: hooked 5 NGX exports on _nvngx.dll
```

(Then the OpenGL fence problem below still applies, if the game is OpenGL.)

---

## The blockers

### OpenGL ↔ D3D12 fence interop (fatal, Wine-level)

```
[feed32] D3D12 fence -> GL semaphore import: in=FAILED out=FAILED
stopped: cross-process fence import failed
```

Single GPU; the helper logs the same adapter LUID as the game, so it is not the Feeder's
multi-GPU case. Wine does not appear to implement `GL_HANDLE_TYPE_D3D12_FENCE_EXT` import.
`host_creates=0` does not reverse the direction. Failed identically in two structurally
different setups (64-bit in-process, 32-bit cross-process).

Everything *else* in the FF8R chain worked: depth and motion vectors resolved, IPC v9 connected,
`feature ready: 1920x1080 DLAA`, 4 shared D3D12 textures handed over.

### Depth in emulators

The PlayStation had no Z-buffer. DuckStation can synthesise one via PGXP, but writes it to an
**R32F rasterizer-ordered UAV**, and ReShade's Generic Depth only tracks depth-stencil *views* —
so it reports "No depth buffers found" no matter how permissive the heuristics. Fix, all three needed:

1. `[GPU] DisableRasterOrderViews = true` in DuckStation's `settings.ini`
   (section is **`[GPU]`**, key is **`DisableRasterOrderViews`** — Raster, not Rasterizer).
   Costs ~7 fps because ROV also did shader blending.
2. ReShade → Add-ons → Generic Depth → select the **4096×2048** buffer (= 4× internal res).
3. **Untick "Copy depth buffer before clear operations"** — with it on you copy a just-cleared
   (all-zero) buffer. This is the step that finally yields `variance 0.105, 100% finite`.

`duckstation.log` with `LogToFile = true` is what reveals the cause: `Using ROV depth: YES` /
`Using real depth buffer: NO`.

---

## Gotchas that cost hours

- **`[fastopt]`** — ReShade's D3D codegen emits it; Wine's builtin `d3dcompiler_47` (vkd3d-shader)
  rejects it and *every* effect fails to compile. Drop Microsoft's native `d3dcompiler_47.dll`
  beside the exe with `d3dcompiler_47=n,b`. OpenGL games are unaffected (GLSL codegen).
- **`icuuc.dll`** — DuckStation's Qt6Core imports it; it is a Windows system DLL since 10 1703,
  Wine lacks it, and DuckStation bundles none, so the exe dies at load with `c0000135` and no
  message. Proton's `icu.dll` has the matching **unversioned** exports — stage it as `icuuc.dll`.
  `icuuc68.dll` does **not** work (its symbols are suffixed `_68`).
- **Bitness** — a 64-bit ReShade in a 32-bit game fails *silently*: Wine falls back to its builtin
  and you get no log and no overlay. Check `/proc/<pid>/maps` for `i386-windows`.
- **Compiled ≠ enabled** — ReShade effects need a preset entry (`Techniques=`), in the right order.
- **Fullscreen/resize** can crash `CreateFeature` with `0xC0000005`; keep the window fixed.
- **Removing the game-local `nvngx_dlss.dll`** is Windows-only advice. On Linux the driver ships no
  DLSS SR runtime (only `nvngx_dlssg.dll`), so removing it kills DLSS entirely.
- **Static scenes lie** — motion-vector probes read `0% non-zero` when nothing moves. Don't diagnose
  MV while standing still.

---

## Cost

Neural rendering is not free — it is the opposite of what "DLSS" usually implies.

| | |
|---|---|
| FF7 Rebirth @ 5120×1440 | 22–29 fps with the pass on |
| DuckStation @ 982×747 | 59.8 fps, pass = 5.4 ms/frame (33% GPU) |
| DuckStation @ 5120×1440 | 38.7 fps, pass = 23.3 ms/frame (**90% GPU**) |

The cost scales with **output** resolution. Low-resolution content (emulators) is where it is
genuinely usable, because the GPU has headroom to absorb it.

---

## Credits

[addon-dlssnr-linux](https://github.com/NapXDD/addon-dlssnr-linux) ·
[DLSS5-Feeder](https://github.com/jlrouzies-fr/DLSS5-Feeder) ·
[OptiScaler_DLSSNR](https://github.com/Dagherbou/OptiScaler_DLSSNR) ·
[RenoDX](https://github.com/clshortfuse/renodx) · [ReShade](https://github.com/crosire/reshade)

Unofficial, unaffiliated with NVIDIA. Findings only — no NVIDIA binaries are redistributed here.
