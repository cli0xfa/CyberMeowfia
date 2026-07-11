# CyberMeowfia — K60 (Redmi K60) Adaptation

This fork contains adaptation work for running the **CVE-2026-43499 (GhostLock / IonStack Part II)** exploit on **Redmi K60 (mondrian, kernel 5.10.236)** and other ARM64 Android devices.

> **Original upstream**: [NebuSec/CyberMeowfia](https://github.com/NebuSec/CyberMeowfia)

---

## What's in this fork

### 🔧 K60 (mondrian) Target

Full device-specific exploit adaptation at `IonStack/CVE-2026-43499/exploit/src/targets/mondrian-AQ3A.250226.002/`:

| Component | File | Description |
|-----------|------|-------------|
| **target.h** | `target.h` | ~40 kernel symbol offsets for GKI 5.10.236 (KIMAGE_TEXT_BASE, ashmem, configfs, selinux, slab, etc.) |
| **fops.c** | `fops.c` | 5.10-compatible configfs `.read`/`.write` (6.x uses `.read_iter`/`.write_iter`) |
| **slide.c** | `slide.c` | KASLR bypass when delta=0 (bypasses futex PI deadlock on < 5.12) |
| **util.c** | `util.c` | KernelSnitch mm_struct leak with ARM64 timing tuning + pagemap-based identity map fallback |
| **main.c** | `main.c` | Owner-thread unlock f_pi_target for 5.10 PI chain trigger attempt |

### 🏗️ CI Workflow

`.github/workflows/build.yml` provides automated builds:

```bash
# Trigger a build for the K60 target
gh workflow run build.yml \
  -f project=mondrian-AQ3A.250226.002 \
  -f api=35

# Build for a different target (e.g., Pixel 10)
gh workflow run build.yml \
  -f project=blazer-CP2A.260605.012 \
  -f api=35

# Build kernel vmlinux for offset extraction
gh workflow run build.yml \
  -f project=blazer-CP2A.260605.012 \
  -f api=35 \
  -f kernel_tag=android12-5.10.236_r00
```

### 📦 Build Artifacts

| Artifact | Description |
|----------|-------------|
| `preload-{project}.so` | LD_PRELOAD exploit library |
| `poc` | Standalone UAF reproducer (futex PI requeue race) |
| `vmlinux-{kernel_tag}` | Compiled kernel vmlinux (for offset extraction) |

**Build Requirements:**
- Android NDK r29 (auto-downloaded in CI)
- Android API 35 (or matching device API)
- For kernel builds: Clang 12 + aarch64-linux-gnu toolchain

### 🧩 Kernel Offset Extraction

The `scripts/extract_offsets.py` script automatically extracts all needed symbols from a compiled `vmlinux`:

```bash
python3 scripts/extract_offsets.py
```

Example output:
```
KIMAGE_TEXT_BASE=0xffffffc008000000
init_task off=0x1ccce80
ashmem_fops off=0x14d0fd0
configfs_read_bin_file off=0x0040575c
...
```

### 🔬 Device Analysis

The adaptation includes full device security posture analysis:

| Check | K60 Result |
|-------|-----------|
| Kernel | 5.10.236-android12-9 (GKI) |
| SELinux | Enforcing |
| Shell capabilities | CapEff=0 (zero effective caps) |
| Bootloader | Locked |
| ASLR | KASLR disabled (delta=0) |

### ⚠️ Known Limitation

CVE-2026-43499 requires **Linux 5.12+** for the full privilege escalation chain. The 5.10 kernel lacks `rt_mutex_adjust_pi()` (added in commit `a55c072dfe22`), which is needed to trigger a second PI chain traversal after the stack UAF is set up. On < 5.12, the `rb_erase` write primitive cannot be reliably triggered from userspace.

**The exploit is fully functional on 5.12+ devices** (e.g., Xiaomi 13 at 5.15.178, Pixel 9/10 series). See PR #16 and PR #22 upstream.

---

## Quick Start

```bash
# Build for K60
gh workflow run build.yml --ref main \
  -f project=mondrian-AQ3A.250226.002 \
  -f api=35

# Push to device and test
adb push preload.so /data/local/tmp/
adb shell "LD_PRELOAD=/data/local/tmp/preload.so /system/bin/id"
```

## Device Compatibility

| Device | Kernel | Target | Status |
|--------|--------|--------|--------|
| Redmi K60 | 5.10.236 | `mondrian-AQ3A.250226.002` | UAF triggered, KASLR bypass, mm_struct leak ✅ / Full chain blocked by < 5.12 |
| Xiaomi 13 | 5.15.178 | `xiaomi13` (PR #16) | Full chain ✅ |
| Pixel 10 | 6.x | `blazer-CP2A.260605.012` | Full chain ✅ |

---

## Directory Structure

```
IonStack/CVE-2026-43499/exploit/
├── src/
│   ├── targets/
│   │   └── mondrian-AQ3A.250226.002/   ← K60 target
│   │       ├── target.h                   Kernel offset definitions
│   │       ├── fops.c                     5.10 configfs fops override
│   │       ├── slide.c                    KASLR bypass (no-PI path)
│   │       ├── util.c                     arm64-tuned KernelSnitch
│   │       └── main.c                     5.10 PI trigger attempt
│   ├── kernelsnitch/                      KernelSnitch timing library
│   ├── common.h                           Shared macros
│   ├── offset.h                           Target config include
│   ├── main.c, slide.c, util.c, fops.c, pipe.c, root.c
│   └── preload.c, su_daemon.c, su_blob.S, wallpaper_blob.S
├── scripts/
│   └── extract_offsets.py                 Kernel symbol extractor
├── Makefile
└── assets/
```

## Build System Details

The Makefile supports per-target source overrides via `$(call pick_src,file.c)`:

```make
CORE_SRCS := \
  $(call pick_src,main.c) \
  $(call pick_src,util.c) \
  $(call pick_src,slide.c) \
  $(call pick_src,fops.c) \
  $(call pick_src,pipe.c) \
  src/root.c
```

If `src/targets/{PROJECT}/main.c` exists, it's used instead of `src/main.c`. This enables clean per-device customization without modifying shared code.

---

## License

Apache License 2.0 — see [LICENSE](LICENSE)
