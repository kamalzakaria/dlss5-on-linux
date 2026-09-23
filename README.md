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
| Tomb Raider I–III Remastered | OpenGL (64-bit) + Feeder | ❌ GL transport: `D3D12 fence -> GL semaphore import: FAILED` |
| Tomb Raider I–III Remastered | **OpenGL → Vulkan via Mesa Zink** + Feeder | ✅ **works — 31 fps @ 5120×1440 with the neural pass** |
| FINAL FANTASY VIII Remastered | OpenGL (32-bit) + Feeder, native GL **and** via Zink | ❌ Zink gets to a DLAA build, then Wine can't import the host64 helper's D3D12 fence (cross-process) |

**Conclusion: DLSS 5 works on Linux for Direct3D games — and for OpenGL games too, once you make
them Vulkan apps.** The Feeder's OpenGL transport is genuinely impossible under Wine (confirmed with
upstream: neither a D3D12 fence nor resource imports into GL, `GL_INVALID_ENUM` on both). But running
the game's OpenGL through Mesa's PE-side **Zink** lets the Feeder take its Vulkan transport, where the
same import succeeds. See [OpenGL games via Zink](#opengl-games-via-zink-gl--vulkan) below.

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

### OpenGL under Proton — settled: not possible via DLSS5-Feeder

```
[feed32] D3D12 fence -> GL semaphore import: in=FAILED out=FAILED
stopped: cross-process fence import failed
```

Everything *else* in the FF8R chain works: depth and motion vectors resolve, IPC v9 connects,
`feature ready: 1920x1080 DLAA`, 4 shared D3D12 textures hand over. Only the fence won't cross.

**Confirmed dead end.** The Feeder's maintainer added a `memory-import probe` in **1.16.0-beta.7**
specifically to settle whether a CPU-synchronised fallback was possible. Retested there:

```
the fence import failed at glImportSemaphoreWin32HandleEXT(D3D12_FENCE), GL error 0x0500
running under Wine 11.0: it advertises GL_EXT_semaphore_win32 / GL_EXT_memory_object_win32 whatever
  the host's GL driver can do with a Win32 handle, so the extension gate cannot see this
memory-import probe: slot 0..3 does NOT import into GL either
  (failed at glImportMemoryWin32HandleEXT(D3D12_RESOURCE), GL error 0x0500)
```

`0x0500` is `GL_INVALID_ENUM`, identical for both calls — the Win32 handle **type** is rejected
outright. The native Linux NVIDIA driver implements only the fd-based `GL_EXT_memory_object_fd` /
`GL_EXT_semaphore_fd`; Wine advertises the `_win32` variants regardless, which is why the extension
gate cannot detect this. With neither fence nor textures crossing into GL there is nothing for a
CPU-sync fallback to synchronise, so that approach is off the table.

Remaining routes: a staging copy, or **Zink** (GL→Vulkan) in front of the game — **which works**, see below.
See [issue #121](https://github.com/jlrouzies-fr/DLSS5-Feeder/issues/121).

### OpenGL games via Zink (GL → Vulkan)

The route the Feeder's maintainer named in #121, and it works. Proven on Tomb Raider I–III Remastered
(x64). At 5120×1440, measured with `WINEDEBUG=fps`:

| | fps |
|---|---|
| Zink + ReShade | 132 |
| + Feeder, plain DLAA (Vulkan transport) | 88 |
| + **DLSS 5 neural pass** | **31** |

```
[feed] D3D12 fence -> Vulkan timeline semaphore import: in=OK out=OK
nr-fwd: CreateFeature(18) => 0x1 (Success)
```

**The stack** (every piece required):

1. **Mesa for Windows** ([pal1000/mesa-dist-win](https://github.com/pal1000/mesa-dist-win)): its `opengl32.dll`
   + `libgallium_wgl.dll` beside the exe, `GALLIUM_DRIVER=zink`, `WINEDLLOVERRIDES=opengl32=n,b`. Zink turns the
   game's GL into Vulkan *inside* the Wine process. `GALLIUM_HUD=fps` is a quick on-screen proof it's active.
2. **ReShade as a Vulkan layer** — the `opengl32` proxy slot now belongs to Mesa. Wine's builtin `vulkan-1.dll`
   forwards to the host loader and cannot load Windows-side layers, so put the **genuine Khronos loader** beside
   the exe with `vulkan-1=n,b`. NuGet `Silk.NET.Vulkan.Loader.Native` (runtimes/win-x64) is the easy source —
   LunarG's runtime zip ships an empty x64 folder and its installer is a Qt IFW blob. The loader finds the GPU
   through the `winevulkan.json` ICD Proton already registers.
3. **Register ReShade as an *implicit* layer in the prefix registry**:
   `HKLM\Software\Khronos\Vulkan\ImplicitLayers` → `<path to manifest.json>` = `0` (both `/reg:64` and `/reg:32`).
   **Environment variables don't work here**: Wine processes run elevated, and the Khronos loader then ignores
   `VK_LAYER_PATH`, `VK_ADD_LAYER_PATH` *and* `VK_INSTANCE_LAYERS` — silently, because its diagnostics don't
   reach stderr in a GUI process. Debug it with `vulkaninfo.exe` in the same prefix, whose callback does print them.
4. Feeder 1.17+ `dlss5-feed.addon64` + `addon-dlssnr-linux` in the game folder. The Feeder's in-process
   `vkCreateDevice` hook adds the interop extensions itself; no fallback layer needed.

**Traps:**
- ReShade in layer mode may pick `ReShade2.ini` — symlink it to `ReShade.ini`.
- **VRAM.** The first working run hit 2 fps with the GPU "100%" busy at only ~80 W — PCIe paging, because an
  idle ComfyUI held ~7.6 GB. Freed (`POST http://127.0.0.1:8188/free {"unload_models":true,"free_memory":true}`),
  it ran 31 fps at ~190 W. **Low power at 100% utilisation = VRAM starvation**, not compute.
- ReShade assumes reversed depth; Zink's is standard here (values cluster ~0.98) — set
  `RESHADE_DEPTH_INPUT_IS_REVERSED=0`.
- Quality trails a native-DLSS game: motion vectors are optical flow, not engine MVs.
- `GALLIUM_HUD_DUMP_DIR` writes nothing under Wine; measure with `WINEDEBUG=fps` instead.

**32-bit games: blocked by Wine (FF8 Remastered).** The same stack in x86 (Mesa x86, x86 Khronos loader,
ReShade32 registered under `HKLM\Software\Wow6432Node\Khronos\Vulkan\ImplicitLayers`, `dlss5-feed.addon32`
+ `host64` helper) gets all the way to `feature ready: 1920x1080 DLAA`, then the game crashes:
```
fixme:vulkan:win32u_vkImportSemaphoreWin32HandleKHR d3d12 fence from other process.
err:vulkan:vkImportSemaphoreWin32HandleKHR Exception 0xc0000005 in Unix call.
```
On x64 the D3D12 fence lives in the game process, so the import works. On x86 it comes from the separate
64-bit helper, and Wine (Proton 11.0-20260917b) doesn't implement cross-process D3D12 fence import. The
Feeder's `host_creates` ownership flip only covers D3D11 clients. This needs a Wine change, or a Feeder change
where the game exports the semaphores
([#121 comment](https://github.com/jlrouzies-fr/DLSS5-Feeder/issues/121#issuecomment-5795634939)).

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
