# HCR2-Linux

Porting **Hill Climb Racing 2** (Android, Cocos2d-x) to run natively on Linux — driving the engine through the `mcpelauncher-hcr` fork's Android shim (`libmcpelauncher`), no emulator. The twist: instead of translating touch, we're going straight for native keyboard/mouse input.

This repo logs the journey. The launcher itself lives at `/home/rower/mcpelauncher-hcr`; `PROGRESS.md` is the running tracker.

## How we're doing it

1. **Extract & analyze** the real device APK (`com.fingersoft.hcr2`), reverse the packed `libcocos2dcpp.so` JNI surface (`org.cocos2dx.lib.*` + `com.fingersoft.hcr2.AppActivity`) from dex.
2. **Bypass** the Fingersoft anti-tamper lib (`libfingersoft_hcr2.so`) so the Cocos engine actually loads its assets.
3. **Run** the engine inside mcpelauncher's C++ JNI fake environment (FakeJNI/FakeAssetManager) with `jniMode:"cocos2dx"`.
4. **Reverse** the in-memory engine to find choke points (frame loop, thread dispatch, asset loader), hook them, and route direct mouse/keyboard to the engine.

## Status: roughly 55% of the way

| Area | Progress |
|------|----------|
| Infrastructure (build, JNI shim, asset path, loader hook) | **~90%** |
| Game boot up to first real frame | **~45%** |
| Native (mouse/keyboard) input | **0%** |

### What's working
- mcpelauncher-hcr builds and runs; engine `.so` loads, JNI natives bind, Cocos thread starts.
- Real device capture pipeline (`capture_hcr2.sh`), dex decompile (jadx/cfr), packed-`.so` decoding.
- Anti-tamper lib bypassed → asset loading truly begins (`SetApkPath` now feeds the real `base.apk`; frame 4 does real work, 65ms → 193ms).
- `SIGUSR2` render-thread stack dump gives a full FP-chain unwind of a live engine render frame.

### Current blocker — the "frame-6 stall"
The render thread **deadlocks inside `pthread_detach`'s join path**:

- Call chain: `nativeRender` → `0x1c4a25e` → `0x1c47927` → `0x1c65ce7` → `0x1c62f84` → `0x1c6328c` → `0xce33b7` → `0xcd9225` → `0x1400129` → `0x19852f0` (sync-submit) → `0x1fb32e1` (`pthread_detach`) → futex wait in libc (`__GI___futex_abstimed_wait64`).
- The "spin" we kept sampling was really this futex ret-site. The `futex` returns `EINVAL` immediately (misaligned address), so the thread loops with high CPU.
- The handle fed to `pthread_detach` is an **engine-internal object, not a real glibc pthread**, so the join never progresses.

**Next step:** shim `pthread_detach` (GOT slot `0x21f79f8`) to identify that object, get the loader to complete, register `onFinishedAssetLoader()`, then start on the input path.

## Repo contents
- `PROGRESS.md` — full engineering log: offsets, GOT, thread census, next moves.
- Key artifacts are kept out of this repo (size/licensing); they're regenerated on demand via `capture_hcr2.sh`.

## Working set (on disk)
- `/tmp/hcr2/device_logcat.log` — device timeline (audio info, `onFinishedAssetLoader`, frame timing)
- `/tmp/hcr2/libcocos.mem` + `libcocos.got` — in-memory engine dump + GOT (base = `nativeRender − 0x1a51370`)
- `/tmp/hcr2/renderstack.txt` — live render-thread FP backtrace (stall evidence)
- `/tmp/hcr2/runNN.log` — per-run launcher logs
- `/tmp/hcr2/hcr2_src/.../org/cocos2dx/lib/`, `hcr2_src5/.../AppActivity.java` — JNI surface

## License / scope
Engineering log for a porting project; no copyrighted assets shipped. HCR2 belongs to Fingersoft.