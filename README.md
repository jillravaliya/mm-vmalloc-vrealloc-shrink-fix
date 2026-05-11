# Eight Megabytes That Never Came Back

![Mainline Linux](https://img.shields.io/badge/MAINLINE_LINUX_7.0.0-0a0a0a?style=for-the-badge&logo=linux&logoColor=F5C842)
![Subsystem](https://img.shields.io/badge/mm%2Fvmalloc.c-0a0a0a?style=for-the-badge&logo=gnu&logoColor=aaaaaa)
![Bug](https://img.shields.io/badge/VREALLOC_SHRINK_BUG-0a0a0a?style=for-the-badge&logo=hackaday&logoColor=3DD68C)
![LKML](https://img.shields.io/badge/LORE.KERNEL.ORG-0a0a0a?style=for-the-badge&logo=mail.ru&logoColor=5B9CF2)
![syzbot](https://img.shields.io/badge/SYZBOT_TESTED-0a0a0a?style=for-the-badge&logo=googlecloud&logoColor=F26B5B)

> **vmalloc allocated 10MB. The caller asked to shrink to 2MB. The kernel zeroed the tail, updated a bookkeeping field, returned success — and never freed a single physical page. 8MB of RAM: silently gone. No crash. No warning. Just a TODO comment sitting right there in mainline, left by the developer who already knew.**

---

## What Happened

A TODO comment inside `mm/vmalloc.c` was found, investigated, fixed, and submitted to the Linux kernel mailing list.

```c
/*
 * TODO: Shrink the vm_area, i.e. unmap and free unused pages.
 */
if (size <= old_size) {
    memset((void *)p + size, 0, old_size - size);
    vm->requested_size = size;
    return (void *)p;    ← returns here. pages never freed.
}
```

- `vrealloc()` is the kernel's vmalloc resize function
- The shrink path zeroed unused memory and updated `requested_size`
- The physical pages behind those addresses were **never unmapped, never freed, never returned to the buddy allocator**
- The TODO was left by the developer who wrote the function — the problem was already known

> Shrink a 10MB vmalloc allocation to 2MB and 8MB of RAM disappears until `vfree()` is eventually called on the whole thing.

---

## Root Cause

> `vrealloc()`'s shrink path handled the bookkeeping but skipped the actual work: unmapping the page table entries for the tail region, flushing the TLB, and returning the physical pages to the buddy allocator. RAM stayed allocated with no owner.

Every vmalloc allocation is backed by three things the kernel tracks simultaneously:

```
1. Virtual address range    — which kernel virtual addresses are claimed
2. Physical pages           — actual RAM frames borrowed from the buddy allocator
3. Page table mappings      — MMU translations: virtual page → physical page
```

When `vfree()` frees an allocation it cleans up all three. When `vrealloc()` shrinks one, it was cleaning up **none of them** for the unused tail:

```
vmalloc(10MB) previously called:
├── 2560 physical pages allocated
└── 2560 page table entries written

vrealloc(p, 2MB) — shrink path:

What happened:       memset tail → update requested_size → return ✓
What was skipped:    page table entries 2MB–10MB     → STILL MAPPED  ✗
                     physical pages 512–2559          → STILL ALLOCATED ✗
                     TLB entries for 2MB–10MB         → STILL CACHED ✗
```

**2048 physical pages — 8MB of RAM — locked up doing nothing.**

The buddy allocator had no idea those pages were available. Other kernel code could not use them. They sat until `vfree()` was called on the entire allocation.

---

### Why the TLB Matters

The TLB is the CPU's hardware cache of recent virtual-to-physical address translations. After unmapping page table entries, the TLB still holds the cached translations. Any access to those virtual addresses before a TLB flush finds a stale entry pointing at physical pages that may already belong to someone else — a use-after-free at the hardware level.

`vunmap_range()` — which already exists in `vmalloc.c` — handles both: clears the page table entries and flushes the TLB. The fix calls it for the tail range.

> Order is critical: unmap before free. Freeing physical pages first creates a window where page tables still claim those addresses are valid.

---

## The Fix Applied

**21 lines added to the shrink path in `mm/vmalloc.c`:**

```c
/* Shrink the vm_area: unmap and free unused pages. */
if (size < alloced_size) {
    unsigned long new_nr_pages = PAGE_ALIGN(size) >> PAGE_SHIFT;
    unsigned long i;

    /* Unmap unused virtual range and flush TLB. */
    vunmap_range((unsigned long)p + PAGE_ALIGN(size),
                 (unsigned long)p + alloced_size);

    /* Free unused physical pages back to buddy allocator. */
    for (i = new_nr_pages; i < vm->nr_pages; i++) {
        mod_lruvec_page_state(vm->pages[i], NR_VMALLOC, -1);
        __free_page(vm->pages[i]);
        vm->pages[i] = NULL;
    }

    vm->nr_pages = new_nr_pages;
}
```

**Verified with a kernel module running in QEMU:**

```
free pages before vmalloc(10MB):   83,477
free pages after  vmalloc(10MB):   83,477 − 2560 = 80,917
free pages after  vrealloc(2MB):   80,917 + 2538 = 83,455

PASS — 2538 pages returned to system
```

> On the unpatched kernel, free pages after `vrealloc()` showed zero change.

---

## What Came Back

Two responses arrived within hours of submitting to the mailing list.

**Uladzislau Rezki — the developer who wrote `vrealloc()` and left the TODO:**

```
There is already work to address this:
https://lore.kernel.org/all/20260428-vmalloc-shrink-v12-0-3c18c9172eb1@zohomail.in/
```

**syzbot — Google's automated kernel fuzzer:**

```
kernel BUG in __vunmap_range_noflush

Triggered via: BPF verifier → bpf_patch_insn_data → vrealloc()
vunmap_range() called with invalid args in BPF-specific allocation patterns
```

> The fix had the right core idea. The production version needed 12 iterations across 5 months to handle locking, `VM_USERMAP` flag checks, kmemleak tracking, `GFP_NOFS`/`GFP_NOIO` context skipping, and a `vread_iter()` fix for `/proc/kcore`. This was iteration 1.

---

## Patches on lore.kernel.org

```
https://lore.kernel.org/linux-mm/?q=jillravaliya

[PATCH 1/2] mm/vmalloc: free unused pages when shrinking vrealloc() allocation
[PATCH 2/2] selftests/mm: add test for vrealloc() shrink page freeing
             2026-05-07 11:48 UTC
```

---

## Affected Code

- Any kernel caller that uses `vrealloc()` to shrink a vmalloc allocation
- BPF verifier (`kernel/bpf/fixups.c`) — resizes instruction arrays via `vrealloc()`
- Any long-running workload where vmalloc regions are repeatedly grown and shrunk
- Systems where `VmallocUsed` in `/proc/meminfo` grows unbounded over time

---

## Connect With Me

I'm actively looking for internship opportunities in Linux kernel development, systems programming, and memory management.

- **Email:** jillravaliya@gmail.com
- **LinkedIn:** [linkedin.com/in/jill-ravaliya-684a98264](https://linkedin.com/in/jill-ravaliya-684a98264)
- **GitHub:** [github.com/jillravaliya](https://github.com/jillravaliya)
- **Kernel patches:** [lore.kernel.org/linux-mm/?q=jillravaliya](https://lore.kernel.org/linux-mm/?q=jillravaliya)

---

> *Eight megabytes that never came back. A TODO left by someone who knew. The fix was 21 lines — and getting it fully right took someone else 12 iterations across 5 months. The core insight was correct. The production gaps were real. Both things are true.*
