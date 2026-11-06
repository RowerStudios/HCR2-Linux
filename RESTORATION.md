# HCR2-Linux — Restoration Guide

The development machine was reset/wiped. The `mcpelauncher-hcr` launcher source tree and `/tmp/hcr2` artifacts were lost. This document explains how to rebuild the working state from what survived.

## What survived (on this machine)

| Item | Location | Status |
|------|----------|--------|
| Docs + patches | `/home/rower/Data/HCR2-Linux/` | ✅ Git repo, pushed to GitHub (`RowerStudios/HCR2-Linux`) |
| Reconstructed launcher source | `/home/rower/Data/mcpelauncher-hcr/` | ✅ All 5 repos cloned + HCR2 patches applied + committed |
| Extracted HCR2 APK (game data) | `/home/rower/Data/mcpelauncher-hcr/game_data/` | ✅ Copy of `/home/rower/Data/hcr2_data/` |
| App config (jniMode: cocos2dx) | `game_data/app_config.json` | ✅ |

## What was lost

- The original `mcpelauncher-hcr` working tree (never pushed to GitHub, only on the reset machine)
- `/tmp/hcr2/` artifacts: `libcocos.mem`, `libcocos.got`, `renderstack.txt`, per-run logs, device logcat
- Any local build artifacts

These are regenerable from the running port; the patches + source tree are what matter.

## Reconstructed launcher tree

`/home/rower/Data/mcpelauncher-hcr/` contains 5 git repos, each with HCR2 patches applied and committed:

| Repo | Path | Patches applied |
|------|------|-----------------|
| mcpelauncher-client | `client/` | `mcpelauncher-client.patch` (cocos2dx jniMode, app_config, startCocos2dGame, etc.) |
| mcpelauncher-core | `core/` | `mcpelauncher-core.patch` (fake_opensl, GLES1 symbols) |
| libc-shim | `libc-shim/` | `libc-shim.patch` (misc shim symbols, pthreads fixes) |
| mcpelauncher-linker | `linker/` | `mcpelauncher-linker.patch` (compat.h) |
| game-window | `game-window/` | `game-window.patch` (game_window.h, window_sdl3) |

Patches source: `/home/rower/Data/HCR2-Linux/patches/` (also on GitHub).

## How to build

The mcpelauncher project uses CMake. Each repo builds independently. Typical dependencies: CMake, SDL2, OpenGL/GLES, libpng, zlib, etc.

Build order: `core` → `libc-shim` → `linker` → `game-window` → `client` (plus `mcpelauncher-common` as a dependency of client).

Example:
```bash
cd /home/rower/Data/mcpelauncher-hcr/core
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install

# repeat for libc-shim, linker, game-window, then client
```

Also clone `mcpelauncher-common` (dependency of client) if not already available.

## How to run

The launcher reads `app_config.json` from the game directory. Use `--game-dir` to point at the extracted APK:

```bash
cd /home/rower/Data/mcpelauncher-hcr/client
./build/mcpelauncher-client --game-dir /home/rower/Data/mcpelauncher-hcr/game_data/
```

`game_data/app_config.json` sets:
- `jniMode: "cocos2dx"` → uses the Cocos2d-x render path
- `nativeLibrary: "libcocos2dcpp.so"`
- `preloadLibraries: ["libcocos2dcpp.so", "libfingersoft_hcr2.so"]`
- `minecraftPatches: false` → skips Minecraft-specific patches
- `package: "com.fingersoft.hcr2"`

Expected result (last known good state, run76): the engine boots, `nativeRender` drives frames continuously, `eglSwapBuffers` spins until the process is killed (the port's own 45 s timer). No crash, no frame-6 stall (`pthread_detach` GOT shim returns 0).

## Remaining work (from PROGRESS.md)

1. Register `onFinishedAssetLoader()` on the fake `Hcr2Activity` (`src/jni/cocos2dx_activity.h` / `.cpp`) — gate out of the loading stage
2. Build the mouse/keyboard input path (engine's `nativeOnKeyEvent`/`nativeOnTouchEvent` JNI surface)
3. Polish: vsync, window lifecycle, audio (OpenSL shim already at 48000/240)

## Key engineering facts (from PROGRESS.md)

- Engine base = `nativeRender − 0x1a51370`, dumped to `libcocos.mem`
- GOT slot `0x21f79f8` → `pthread_detach` (shimmed to return 0)
- Render thread drives the asset loader; `onFinishedAssetLoader()` fires from inside `nativeRender` ~11s after launch on device
- OpenSL ES handles are ptr-to-ptr (`&o->vfPtr`); IID compared as raw 8-byte value
- Device logcat and APK capture were on the reset machine; regenerate via `capture_hcr2.sh` when a device is available
