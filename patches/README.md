### **This directory contains the Linux kernel patchset that fixes a memory reclaim issue in `vrealloc()` when shrinking vmalloc-backed allocations.**

---

### 1. Main Fix Patch

> **`0001-mm-vmalloc-free-unused-pages-when-shrinking-vrealloc.patch`**

This patch fixes the `vrealloc()` shrink path in `mm/vmalloc.c`.

### Problem

Before this fix:

- `vrealloc()` updated `requested_size`
- Unused memory was zeroed
- But physical pages were **not released**
- Page table mappings remained valid
- TLB entries were not invalidated

As a result, shrinking a vmalloc allocation did not actually reclaim RAM.

### Solution

The patch now:

- Unmaps the unused virtual range using `vunmap_range()`
- Flushes stale TLB mappings
- Frees unused physical pages back to the buddy allocator
- Updates `vm->nr_pages` to reflect the reduced allocation size

### Impact

**Shrinking large vmalloc allocations now immediately returns unused pages
to the system instead of keeping them reserved until `vfree()`.**

---

### 2. Validation / Selftest

> **`0002-selftests-mm-add-test-for-vrealloc-shrink-page-freeing.patch`**

This patch adds a regression selftest under:

`tools/testing/selftests/mm/`

### Test Workflow

The selftest verifies that pages are truly reclaimed after shrink:

1. Allocate 10MB using `vmalloc()`
2. Touch all pages to force physical allocation
3. Shrink allocation to 2MB using `vrealloc()`
4. Validate that physical pages were released

### Expected Result

- Without the fix:
  - Test fails
  - No physical pages are reclaimed

- With the fix:
  - Test passes
  - ~2048 pages are returned to the system

---

## Testing Environment

Validated on:

- Linux Kernel `7.0.0`
- QEMU virtual machine
