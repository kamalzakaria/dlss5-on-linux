# FINAL FANTASY VIII Remastered — modding on Linux

Full Demaster + Mcindus "Lunatic Pandora" texture stack running under Proton, with a working
depth buffer for ReShade's depth-aware effects.

**The game is 32-bit (x86) OpenGL 3.3** (GLFW/GLEW). `FFVIII_EFIGS.dll` is the engine;
`FFVIII.exe` is a 595 KB stub. That one fact drives most of what follows.

## Order of operations

Each step is inert until the previous one is done, and two of them fail *silently*.

### 1. Demaster (Maki) — the base

[FF8_demaster](https://github.com/MaKiPL/FF8_demaster): `dinput8.dll` proxy + `ff8_demaster.dll`,
and it replaces `FFVIII_LAUNCHER.exe` with its own (back up the stock .NET one).

```
WINEDLLOVERRIDES="dinput8=n,b;opengl32=n,b" %command%
```

**GitHub's newest release is 1.3.3, but the texture packs need 1.3.4+.** The newer build ships
inside the *LunarCry* download, in its `DEMASTER Update/` folder — copy `ff8_demaster.dll` and
`demaster.conf` from there (5.1 MB DLL vs 2.9 MB).

### 2. Export the archives — nothing works before this

Run `ffviii_demaster_manager.exe` inside the game's Proton prefix and let it unpack. Until
`DEMASTER_EXP/` exists you get:

```
There is no export directory, so it looks like you didn't export the files
from zzz files. Not applying patch
```

Result: ~2.7 GB, 14,323 files. A healthy log afterwards shows `Applying FIELD_BACKGROUND PATCH`,
`directIO_fopenReroute: DEMASTER_EXP\…` and so on.

Bonus: the game's GLSL shaders become loose editable files in `DEMASTER_EXP/shaders`
(`main.frag`, `main_2xsal_color_depth.frag`, `main_fxaa2.frag` …).

### 3. Mcindus texture packs

All merge into `DEMASTER_EXP/textures`. **Order matters where they share directories:**

```
Angelwing (field backgrounds) / BattlefieldPack / HorizonPack / FieldModelPack
  -> Rebirth Flame  ->  LunarCry (+ its hashOutput/)  ->  Lionheart
```

- **SeeDReborn** (UI) is independent. With it installed use Rebirth Flame **SR**; without it, **V**.
- Naming trap: *"Angelwing ULTIMA_v2 Tomb Update"* is an **increment on v1.1**, not a v2 base —
  v2 art is released zone by zone on top of v1.1.
- LunarCry's bundled `FMT and Rebirth Flame First!!!.txt` is stale; its release post lists no
  prerequisites, and the packs cover disjoint texture sets anyway (enemies vs player characters).

### 4. ReShade — **must be 32-bit**

A 64-bit ReShade in this game fails **silently**: Wine cannot load it into a 32-bit process, falls
back to its builtin `opengl32`, and you get no log, no overlay, no error. Run the official installer
inside the prefix, target **`FFVIII.exe`**, API **OpenGL**.

Diagnose bitness from the live process:

```sh
grep -ioE '[^ ]*opengl32[^ ]*' /proc/$(pgrep -x FFVIII.exe)/maps
# i386-windows  => the game is 32-bit
```

### 5. Depth buffer — works, with two adjustments

ReShade → Add-ons → **Generic Depth** lists several buffers. Pick by **draw calls**:

| Buffer | Res | Draw calls | |
|---|---|---|---|
| `0x008d41…` | 1920×1440 | **160** | ← the scene |
| `0x008218…` | 1920×1080 | 1 | renders black |
| `0x000de1…` | 1280×960 | 4 | |

Then set `RESHADE_DEPTH_INPUT_IS_UPSIDE_DOWN=1` (OpenGL Y-flip) and leave `IS_REVERSED=0`.
Verify with **DisplayDepth**. This unlocks MXAO, DOF, SSR and the rest of the depth-aware effects —
on a game that composites 3D characters over pre-rendered art, ambient occlusion is the biggest win.

## Not possible (yet)

- **DLSS 5** — see [README](README.md). Every link works except the OpenGL↔D3D12 fence import.
- **lsfg-vk frame generation** — it hooks Vulkan; an OpenGL game never presents through Vulkan.
- **Analog / 360° movement** — the 8-way quantisation lives inside `FFVIII_EFIGS.dll`'s field code,
  so no controller remapping or Steam Input config can help. It needs a Demaster-style binary patch
  that nobody has written. (FF7 has such a mod; FF8 does not.)

## Expected framerate

**30 fps on field screens, 15 in battles and FMVs — by design.** The engine ties rendering to a
fixed timestep and Square Enix cannot change it without the source. Seeing ~31 fps is correct,
not a performance problem.
