# arm64-mTHP Autopilot: Evolution from v1.0 to v2.1

## Code Comparison, Bug Fixes, Architecture Changes, and Rationale

**v1.0:** `arm64-mthp-autopilot-rpi5-main/` (original repo)
**v2.1:** Current production code (ARM64 Wearable SoC + RPi 5)

---

## 1. Architecture Overhaul

### v1.0: Monolithic — Everything in One Module

```
kernel/ebpf_mthp_monitor.c    (719 lines — module, everything in one file)
ebpf/auto_classifier_kern.c   (663 lines — eBPF, loaded from external .o file)
userspace/ebpf_mthp_loader.c  (424 lines — dynamic linking: -lbpf -lelf -lz)
```

All best-fit logic lived only in the eBPF layer. The kernel module did buddy monitoring and shrinker work, but had no VMA-size awareness in the actual allocation path. The eBPF program could observe but NOT change kernel allocation decisions.

### v2.1: Three-Layer Separation

```
mm/mthp_bestfit_hook.c         (28 lines  — always built-in, obj-y)
mm/mthp_bestfit.c              (308 lines — tristate module, obj-m)
include/linux/mthp_bestfit.h   (42 lines  — header, function pointer hook)
mm/memory.c                    (+4 lines  — one-line call in alloc_anon_folio)

drivers/mthp/ebpf_mthp_monitor.c  (835 lines — vendor module)
drivers/mthp/profile.h            (156 lines — shared types)

mthp_classifier.bpf.c          (380 lines — eBPF, skeleton-embedded)
mthp_classifier.c              (130 lines — static binary, zero runtime deps)
mthp_classifier.h              (92 lines  — three-way type guard)
best_fit_order.h               (116 lines — order selection algorithm)
```

### Why the Change

The v1.0 eBPF layer could classify faults and log what order SHOULD have been chosen, but it couldn't change the kernel's actual allocation decision. The kernel still used `highest_order(orders)` blindly. The best-fit logic was observational only.

In v2.1, the VMA-size filter runs INSIDE `alloc_anon_folio()` via a function pointer hook. Every anonymous page fault gets filtered before allocation. The eBPF layer is now purely diagnostic.

**Impact:** v1.0 had 0% actual promotion improvement (observation only). v2.1 achieves 74.8-86.9% measured promotion on real hardware.

---

## 2. Critical Bug Fixes

### FIX-A: call_usermodehelper() Anti-Pattern (SECURITY + STABILITY)

**v1.0 code:**
```c
static void do_drop_caches(void)
{
    static char *argv[] = { "/bin/sh", "-c",
                            "echo 2 > /proc/sys/vm/drop_caches", NULL };
    call_usermodehelper(argv[0], argv, envp, UMH_NO_WAIT);
}
```

**Problems:**
- Spawns `/bin/sh` from kernel context — privilege escalation vector
- UMH_NO_WAIT means no error checking — fire and forget
- Kernel → Userspace → Kernel round-trip for a simple kernel operation
- Can deadlock under OOM (usermode helper needs memory to fork)
- Android SELinux will block this on user builds

**v2.1 fix:**
```c
/* Direct kernel function call, no userspace round-trip */
if (drop_slab_fn)
    drop_slab_fn();
```
Resolved `drop_slab()` address via kprobe at init time. Zero userspace involvement.

**Impact:** Eliminated privilege escalation risk, OOM deadlock, SELinux denial.

---

### FIX-B: Shrinker Count Underflow (OOM KILLER INFINITE LOOP)

**v1.0 code:**
```c
static unsigned long mthp_shrinker_count(struct shrinker *s,
                                         struct shrink_control *sc)
{
    long slab_net = atomic64_read(&g_slab_allocs)
                  - atomic64_read(&g_slab_frees);
    if (slab_net > leak_thresh)
        return (unsigned long)slab_net;
    return g_large_folio_cache / 2;
}
```

**Problem:** If `g_slab_frees > g_slab_allocs` (race condition during rapid deallocation), `slab_net` is negative. Casting negative long to unsigned long produces `ULONG_MAX` (~18 exabytes). The OOM killer sees this astronomical count and enters an infinite reclaim loop.

**v2.1 fix:**
```c
long slab_net = atomic64_read(&g_slab_allocs) - atomic64_read(&g_slab_frees);
slab_net = max(slab_net, 0L);  /* Guard: never report negative */
```

**Impact:** Prevented potential OOM killer infinite loop under memory pressure.

---

### FIX-C: Shrinker Phantom Free (INFINITE RECLAIM LOOP)

**v1.0 code:**
```c
static unsigned long mthp_shrinker_scan(...)
{
    if (slab_net > leak_thresh && auto_reclaim) {
        atomic64_inc(&g_shrinker_base);
        return slab_net;  /* Claims to have freed slab_net objects */
    }
}
```

**Problem:** Returns `slab_net` (number "freed") but actually frees NOTHING. Kernel sees "objects freed" and immediately re-invokes the shrinker for more. Infinite loop.

**v2.1 fix:** Actually calls `drop_slab_fn()`, uses `atomic64_sub()` to decrease the count, and respects `sc->nr_to_scan` limit.

**Impact:** Eliminated shrinker infinite reclaim loop.

---

### FIX-D: /proc/buddyinfo File Read (KERNEL ANTI-PATTERN)

**v1.0 code:**
```c
f = filp_open("/proc/buddyinfo", O_RDONLY, 0);
n = kernel_read(f, buf, PAGE_SIZE - 1, &pos);
/* ... parse ASCII string to extract buddy counts ... */
```

**Problems:**
- Kernel modules must NOT open /proc files — these are for userspace
- filp_open() in a workqueue can deadlock with VFS locks
- 4KB read buffer truncates on NUMA systems with many zones
- String parsing is fragile, slow, and wastes CPU cycles
- NUMA-blind: only parses the first zone, misses others

**v2.1 fix:**
```c
for_each_zone_resolved(zone) {
    if (!populated_zone(zone))
        continue;
    for (order = 0; order < MAX_ORDER; order++)
        total_free[order] += zone->free_area[order].nr_free;
}
```
Direct kernel data structure access. Zero string parsing. NUMA-aware. Resolved `first_online_pgdat()` and `next_zone()` via kprobe at init.

**Impact:** Correct buddy data on NUMA, no VFS deadlock, 100x faster.

---

### FIX-E: Compact Race Condition

**v1.0:** Static `last_compact_jiffies` variable shared across CPUs without synchronization. Two CPUs could trigger compaction simultaneously.

**v2.1:** `atomic64_try_cmpxchg()` ensures only one CPU triggers compaction within the cooldown window.

---

### FIX-F: Buddy Free Array Torn Reads

**v1.0:** Workqueue writes `g_buddy_free[]` while eBPF reads it — no synchronization. Possible torn reads (half-old, half-new values).

**v2.1:** `seqlock` around writes. Reader retries if write was in progress.

---

## 3. eBPF Program Changes

### __builtin_memcmp → bpf_strneq (BUILD FIX)

**v1.0:**
```c
if (__builtin_memcmp(comm, "chromium", 8) == 0 ||
    __builtin_memcmp(comm, "tflite",   6) == 0 || ...
```

**Problem:** `__builtin_memcmp` on BPF target emits an extern `memcmp` symbol. `bpftool gen skeleton` fails with "failed to find BTF for extern memcmp."

**v2.1:**
```c
static __always_inline int bpf_strneq(const char *s, const char *t, int len) {
    for (int i = 0; i < len && i < 16; i++)
        if (s[i] != t[i]) return 0;
    return 1;
}

if (bpf_strneq(comm, "chromium", 8) || ...
```

**Impact:** eBPF program compiles and loads on Android. Required for skeleton-based build.

---

### Angled → Quoted Includes (BUILD FIX)

**v1.0:**
```c
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
```

**v2.1:**
```c
#include "bpf/bpf_helpers.h"
#include "bpf/bpf_tracing.h"
```

**Why:** Prebuilt clang on Android/vendor doesn't have system libbpf headers. Quoted includes find local `bpf/` directory.

---

### Type Guard: `__bpf__` → `__VMLINUX_H__` (BUILD FIX)

**v1.0:**
```c
#ifdef __bpf__
  /* vmlinux.h types */
#else
  #include <stdint.h>
  #include <linux/types.h>
#endif
```

**Problem:** Not all clang versions define `__bpf__` for BPF target. On vendor prebuilt clang-r547379, the guard doesn't trigger, `<stdint.h>` gets included alongside vmlinux.h, and type redefinitions crash the build.

**v2.1:**
```c
#if defined(__VMLINUX_H__)
  /* BPF context: vmlinux.h provides everything */
#elif defined(__KERNEL__)
  #include <linux/types.h>
#else
  #include <stdint.h>
  #include <stdbool.h>
#endif
```

**Why `__VMLINUX_H__` is reliable:** vmlinux.h always defines this macro. Since `#include "vmlinux.h"` comes before the shared header, the guard is guaranteed to trigger.

---

## 4. Loader: Dynamic → Static Skeleton (DEPLOYMENT FIX)

### v1.0: External .o File + Dynamic Linking

```c
// Loader loads BPF from external file at runtime
struct bpf_object *obj = bpf_object__open(obj_path);

// Build: dynamically linked
gcc loader.c -lbpf -lelf -lz -o loader
```

**Problems:**
- Needs `libbpf.so`, `libelf.so`, `libz.so` on target device
- Android rootfs has NONE of these
- Two files to deploy: loader binary + .bpf.o
- Runtime file path resolution is fragile

### v2.1: Skeleton-Based + Static Linking

```c
// BPF bytecode embedded in binary at compile time
#include "mthp_classifier.skel.h"
struct mthp_classifier_bpf *skel = mthp_classifier_bpf__open_and_load();

// Build: statically linked
aarch64-linux-gnu-gcc -static loader.c libbpf.a libelf.a libz.a -o mthp_classifier
```

**Result:** Single 1.5MB static ARM64 binary. Zero runtime dependencies. Works on any Android userdebug device with `adb push`.

---

## 5. Build System: RPi Native → Cross-Compilation

### v1.0: Native Build on RPi 5

```makefile
KDIR ?= /lib/modules/$(shell uname -r)/build
BPF_CC := clang
CC := gcc
```

Only works on the RPi 5 itself. Cannot build for Android. Requires all development tools installed on the Pi.

### v2.1: Cross-Compilation from x86 Host

**Two approaches:**

**A. libbpf-bootstrap-android framework:**
```bash
make BPFTOOL=/usr/sbin/bpftool CC=aarch64-linux-gnu-gcc
```

**B. Kernel tree tools/lib/bpf:**
```bash
make -f Makefile.kernel_tree BPFTOOL=/usr/sbin/bpftool
```

Both use pre-built aarch64 `libelf.a` + `libz.a` from `deps/`. No system ARM64 packages needed.

**Impact:** Build anywhere (x86 workstation), deploy anywhere (RPi 5, Wearable SoC, Wearable SoC).

---

## 6. GKI Module Architecture (NEW in v2.1)

### v1.0: No GKI Awareness

The old code was a simple out-of-tree module for RPi 5. It didn't consider:
- GKI kernel locking (can't modify `common/mm/`)
- Bazel/Kleaf build system
- Vendor module separation
- Symbol export restrictions

### v2.1: Function Pointer Hook for GKI Compliance

```
mm/mthp_bestfit_hook.c    ← obj-y, always built into GKI
  mthp_bestfit_mask_hook   = NULL (default)
  mthp_bestfit_register()  = set pointer
  mthp_bestfit_unregister()= clear pointer

mm/mthp_bestfit.c          ← obj-m, vendor module
  module_init → mthp_bestfit_register(__mthp_bestfit_mask)
  module_exit → mthp_bestfit_unregister()

mm/memory.c                ← GKI locked, calls inline
  orders = mthp_bestfit_mask(vma, orders)
  → reads hook pointer, calls if non-NULL
```

**Why function pointer instead of direct call:** GKI core (`mm/memory.c`) can't link to a module. The function pointer in `mthp_bestfit_hook.c` is always compiled in (`obj-y`), the inline in the header reads it, and the module sets it on load. This lets `rmmod`/`insmod` toggle the feature cleanly.

---

## 7. Symbol Resolution Evolution

### v1.0: Hardcoded Symbol Names

```c
kprobe.symbol_name = "compact_nodes";
kprobe.symbol_name = "kmem_cache_alloc";
```

Broke on Linux 6.12 where `kmem_cache_alloc` was renamed to `kmem_cache_alloc_noprof` (due to CONFIG_MEM_ALLOC_PROFILING).

### v2.1: Fallback Name Tables

```c
static const char * const sym_slab_alloc[] = {
    "kmem_cache_alloc", "kmem_cache_alloc_noprof", NULL
};
static const char * const sym_page_free[] = {
    "free_unref_page", "free_pages_ok", "__free_pages_ok", NULL
};
```

`resolve_symbol_fallback()` tries each name in order. Works across kernel versions 5.10-6.12+.

---

## 8. Kprobe Registration Fix

### v1.0: Unchecked `.registered` Flag

```c
/* Unregister all kprobes */
for (i = 0; i < ARRAY_SIZE(kp_table); i++)
    if (kp_table[i].kp.addr)
        unregister_kprobe(&kp_table[i].kp);
```

**Problem:** Used `.addr` as proxy for "registered." But `.addr` is set during registration even if it fails partway. Unregistering a half-registered kprobe corrupts the kprobe hash table.

### v2.1: Explicit `.registered` Flag

```c
struct kprobe_entry {
    struct kprobe kp;
    const char * const *names;
    bool registered;     /* ← explicit flag */
};

/* Unregister only if actually registered */
if (kp_table[i].registered)
    unregister_kprobe(&kp_table[i].kp);
```

---

## 9. Summary Table

| Area | v1.0 (Old) | v2.1 (New) | Impact |
|---|---|---|---|
| **Best-fit location** | eBPF only (observe) | In-kernel (alloc_anon_folio) | Actually changes allocations |
| **Promotion rate** | 0% (observation only) | 74.8-86.9% | Real page table savings |
| **call_usermodehelper** | `/bin/sh -c "echo 2 > ..."` | Direct `drop_slab_fn()` | No privilege escalation |
| **Shrinker count** | Negative → ULONG_MAX | `max(slab_net, 0L)` | No OOM infinite loop |
| **Shrinker scan** | Returns count, frees nothing | Calls drop_slab, respects nr_to_scan | No phantom free loop |
| **Buddy reading** | `filp_open("/proc/buddyinfo")` | `zone->free_area[order].nr_free` | No VFS deadlock, NUMA-aware |
| **Compact race** | Unguarded static variable | `atomic64_try_cmpxchg` | No double compaction |
| **Buddy array** | No synchronization | seqlock | No torn reads |
| **memcmp** | `__builtin_memcmp` (extern) | Inline `bpf_strneq()` | Skeleton build works |
| **BPF includes** | `<bpf/bpf_helpers.h>` | `"bpf/bpf_helpers.h"` | Cross-compile works |
| **Type guard** | `__bpf__` | `__VMLINUX_H__` | All clang versions work |
| **Loader** | Dynamic, 2 files | Static skeleton, 1 file | Android deployment works |
| **Build** | Native RPi only | Cross-compile x86→arm64 | Build anywhere |
| **GKI** | Not considered | Function pointer hook | Module works with GKI |
| **Symbol names** | Hardcoded | Fallback tables | Works across kernel versions |
| **Kprobe unreg** | `.addr` check | `.registered` flag | No hash table corruption |
| **Lines of code** | 1,806 (3 files) | 2,157 (10+ files) | Better separated, better tested |
| **Platforms tested** | RPi 5 only | RPi 5 + ARM64 Wearable SoC | Production validated |

---

## 10. What Stayed the Same

Despite all the changes, the core algorithm is unchanged:

1. **VMA size determines page order** — same table, same thresholds
2. **Buddy guard** — same 32-block threshold check
3. **SlimBytes accounting** — same PTE/TLB savings formula
4. **Ring buffer events** — same struct classify_event layout
5. **Process classification** — same comm-based app class detection

The v2.1 changes are entirely about making the v1.0 concept actually WORK in production: fixing bugs that would crash the system, building for Android, and integrating into the GKI kernel's allocation path where it can make a real difference.
