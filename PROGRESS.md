# HCR2-Linux — Engineering Log

Running soap-opera of getting Hill Climb Racing 2's Cocos2d-x engine to boot natively on Linux inside the `mcpelauncher-hcr` fork's Android-compat shim, then wiring real keyboard/mouse input.

All offsets are relative to the **in-memory** engine `libcocos2dcpp.so` base (`nativeRender − 0x1a51370`), dumped to `/tmp/hcr2/libcocos.mem`.

---

## Timeline

### 1. Device ground truth
Captured a real Android device running HCR2 (`capture_hcr2.sh`): full logcat + APK copies (`base.apk` 210 MB, `split_config.arm64_v8a.apk` 29 MB).

Key device facts (from `device_logcat.log`):
- `nativeSetAudioDeviceInfo: sampleRate 48000, bufferSizeInFrames 240` (OpenSL ES) right after lib load.
- `AppActivity.onFinishedAssetLoader(SourceFile:192)` called from **inside nativeRender** ~11 s after launch — i.e. the render thread itself is where the asset loader finishes and the callback fires. This is a loading-stage frame.
- Every real frame: `nativeRender` returns quickly (13 ms class); the loader-start frame is the long one. Wine of the code: the engine does a chunked asset build inside one render callback, then must spin its own thread dispatcher.
- `JniHelper: Failed to find static method id of setPlayerGDPRStatus` — benign, engine keeps going.

### 2. Clean-room JNI surface
Decompiled dex (jadx → `hcr2_src5/.../AppActivity.java`, 2558 lines; `hcr2_src/.../org/cocos2dx/lib/`) and smali cross-check. The natives the engine needs:
- `Cocos2dxRenderer.nativeOnDrawFrame()` — per-frame, called on render thread.
- `Cocos2dxActivity.*` misc (`ativeGetFrameSync`, GFX hook, etc.).
- `onFinishedAssetLoader()` — the load-complete signal, read on the Cocos thread; registering this on the fake Java side is the gate to frame 6+.

### 3. Anti-tamper bypass
Fingersoft's `libfingersoft_hcr2.so` checks the APK/loader integrity. Shmed it so `libcocos2dcpp.so` stops aborting early. After this the engine starts doing real work (frames go 16 ms / 0 ms / 0 ms / **193 ms** / 0 ms … then stall).

### 4. Frame-6 stall — the wall
Every run stops at frame ~6 forever. Watchdog + `SIGUSR2` render-thread dump gives the live picture:

Thread census (sample):
```
render  wchan=0  syscall=[running]  cpu(ut+st)=281 195
main    wchan=futex_do_wait
...
```

Full FP-chain unwind of the render thread (`renderstack.txt`), newest first:
```
0x1a51386  nativeRender
0x1c4a25e
0x1c47927
0x1c65ce7
0x1c62f84
0x1c6328c
0xce33b7
0xcd9225
0x1400129
0x19852f0   sync-submit (fills the dispatcher with a mirrored task)
0x1fb32e1   pthread_detach  ← STUCK
    → __GI___futex_abstimed_wait64  (libc+0x94b97, futex EINVAL loop)
```

Key reverse-engineering results:
- `0x1fb32d0` (callers via PLT `0x201e900`): looks like a fence-wait that we proved is a **detach-thread-handle helper**:
  ```
  push rbx
  mov  rbx, rdi
  mov  rdi, [rdi]        ; this->field0 = the wait token
  test rdi, rdi
  jz   err            → EINVAL
  call pthread_detach (via GOT 0x21f79f8 → libc+0x99430)
  test eax, eax
  jnz  err            → log + int3 (trap)
  mov  [rbx], 0
  ret
  ```
- `pthread_detach` (glibc+0x99430): reads TCB state at `pd+0x628`. State==2 → detach (fast, returns 0). State==3 → EINVAL. **Anything else → falls through to `__pthread_clockjoin_ex`** — a real blocking join (futex wait on the target's TID word). glibc's `pthread_detach` on a non-running/not-yet-fully-cleaned-up thread behaves like `pthread_join`.
- At the stall, register dump: `rdi=rbx=0x7fed6ded96b6`, `r12=0x7fed6ded908e`, and the futex uaddr `0x7fed6ded96b6` is **not 4-aligned** → every futex syscall returns `EINVAL` immediately → busy loop at `libc+0x94b97` with high CPU.

**Conclusion:** the engine's sync-submit path hands `pthread_detach` a pointer that isn't a real glibc pthread (or the thread finished with a stale/weird TCB state), so the "wait for task done" join deadlocks. The same code on bionic works because the handle is a real thread there; on our glibc host it's an engine-internal object.

### 5. (Next) un-stick the loader
- Shim/inspect `pthread_detach` (patch GOT slot `0x21f79f8`) to log the object at `[rbx]` and decide: return 0 (pretend joined) vs. find the real thread.
- Once the loader finishes its sync-submit, register `onFinishedAssetLoader()` on the fake `Hcr2Activity`; verify the game exits the loading stage.
- Then build the input path: direct mouse/keyboard → engine (no touch translation).

### 6. ✅ Boot blocker resolved: OpenSL ES fake ABI + game boots & renders
Two distinct ABI bugs in `fake_opensl.cpp` were root-caused from live crash state and fixed:

**Bug A — interface handle ABI.** The original fake stored `*pEngine = &o->objItf` (an embedded struct) and treated handles as `struct*`. Real OpenSL ES handles are a **pointer-to-pointer** (`SLObjectItf = const SLObjectItf_ *const *`; `(*handle)->Method`). The engine dereferenced the function address as a pointer table → SIGSEGV at `engine+0x1b93ad4` (`call *(%rax)`, `si_addr=(nil)`) during audio-engine init. `fake_opensl.cpp` was rewritten to the classic layout: shared static vtable instances (`g_objectVf/g_engineVf/g_playVf/...`), `ObjectImpl` stores `const *Vf` at 0x0..0x38 and `kind` at 0x40, handles are `&o-><vfPtr>`, and `objFor` resolves them via a `g_owners` map.

**Bug B — IID comparison.** The game calls `obj_GetInterface(itf, iid, out)` passing its **own baked-in** IID as a bare 8-byte value equal to the uuid's first 8 bytes (e.g. `iid=0x987d127c359fcbae` ⇔ ENGINE uuid `AE CB 9F 35 7C 12 7D 98 …`), never a readable pointer to the 16-byte uuid. The old code dereferenced `iid` as a pointer → `xor (%r11),%rcx` fault when `r11=0x987d…`. Fixed with `iidIs()`: if `iid` looks like a readable pointer, `memcmp` 16 bytes; otherwise compare the raw 8 pointer-value bytes against each uuid's leading 8 bytes.

(Attempted but not needed: exporting `SL_IID_*` as 8-byte pointer-variables instead of 16-byte arrays — the game doesn't import our IID symbols at all; `readelf --dyn-syms` shows no `SL_IID`/`slCreateEngine` imports, so `getSymbols()` IID entries are inert. Left as-is, harmless.)

**Result (run76):** game fully boots. `opensl.log` shows a single `GetInterface` handled cleanly; the log segfault branch is gone; the process renders continuously — `eglSwapBuffers` spins for the full 45 s `timeout` window to `rc=137` (our own SIGKILL timer, not a crash). Survivor: the asset loader (Frame 6) and all audio init now complete. Only benign warnings remain (`Fail to get Android API level!`, missing `splash-video.ogg`, null audio player — audio is intentionally dropped).

**Remaining gap:** no input yet — `onFinishedAssetLoader` not registered on the fake `Hcr2Activity`, and no mouse/keyboard path. The game boots to the loading/main loop but has no interactivity.


---

## Dispatcher notes (misc RE)
- Dispatcher A `0x1b57a20`: classic task queue — drain while head≠tail, `cond_wait` via `0x1b57d70`→`0x201ebf0` when empty.
- Sync-submit `0x1b58d00`: enqueues a task (`0x1b58c00`), then calls the detach-helper PLT `0x201e900` to wait.
- GOT who's-who (fresh run):
  - `0x21f79f8` → `libc+0x99430` = `pthread_detach`
  - `0x21f0480` → `engine+0x1fb32d0` (self-export used by sync-submit)

## Per-run log index
- `run54..run65` — early boot, JNI binds, thread census.
- `run69` — apkPath fix lands → frame 4 193 ms.
- `run70` — FP-chain capture of the stall; `renderstack.txt` generated.
- `run75` — user-run (rc=139): OpenSL handle ABI fixed, crash moved into `obj_GetInterface` (`iid` deref).
- `run76` — ✅ OpenSL IID deref fixed → **game boots and renders continuously** (rc=137 = own 45 s KILL timer).

---

## Tooling/artifacts (kept on the work machine)
- `capture_hcr2.sh` — device APK + logcat capture.
- `libcocos.mem` / `libcocos.got` / `renderstack.txt` / `runNN.log` — engine memory, GOT, backtraces, run logs.
- `hcr2_src*` — jadx/cfr decompile of `AppActivity.java` + `org.cocos2dx.lib`.

## Next steps (short list)
1. ✅ `pthread_detach` GOT-shim (return 0, no join) — done in jni_support.cpp.
2. ✅ Unblock the fork → loader completes → game boots & renders — done (run76).
3. Register `onFinishedAssetLoader` on the fake `Hcr2Activity`; confirm loading-stage frame transitions.
4. Build the input path: thread mouse/keyboard → engine (touch translated or direct).
4. Input: keyboard/mouse natives on the Cocos thread.
5. Polish: window, vsync, audio (OpenSL path already shimmed 48000/240).