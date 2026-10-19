# 00 — Glossary

Quick definitions for the terms used across the docs.

| Term | Meaning |
|---|---|
| **HV / hypervisor** | The AMD-SVM thin hypervisor Sony runs underneath the PS5 kernel. Owns the nested page tables and enforces `XOTEXT`. |
| **hvcore** | The main hypervisor binary blob, loaded by the HV loader. On Regular PS5 it lives at PA `0x62800000`; on Pro at `0x8E0800000`. |
| **XOTEXT** | "Execute-Only TEXT" — AMD-reserved PTE bit 58 that the HV repurposes to make kernel `.text` unreadable even to a root kernel reader. |
| **NX** | Standard AMD64 PTE bit 63; set means "no execute". |
| **kread / kwrite** | Primitives supplied by the earlier exploit stage: arbitrary read/write at a kernel VA. |
| **kdlsym(i)** | Index into a table of resolved kernel/HV symbols for the detected firmware. |
| **kdldmap(pa)** | Returns a kernel VA that direct-maps the given physical address. |
| **ktext(off)** | Returns kernel `.text` base + `off`. |
| **rawoff(id)** | HV raw-offset table lookup. |
| **ucred** | FreeBSD "user credentials" struct; `authid` 0x4800000000000007 is the PS5 debug/root authid. |
| **ksym9 / ksym8** | Shorthand for `kdlsym(0x9)` and `kdlsym(0x8)` — kernel scratch and kernel data base. |
| **khook** | The in-ELF mini-framework (`kh_init`, `kh_chain_*`, `kh_install`, `kh_trigger`) that builds the resume gadget chain. |
| **KHOOKED\\x01** | Marker literal `0x4B484F4F4B454401` written at `ksym9+0x600` so a re-run can detect a surviving hook. |
| **Cookie** | `0xCAFE133741414141` — written by the HV-side chain into user_data; userland polls for it to know the HV has been compromised. |
| **Rest Mode / Standby** | PS5 S3-like state entered via `sceSystemStateMgrEnterStandby()`. The HV runs its resume path here, which is where the planted chain fires. |
| **Pro vs Regular** | Detected by `sceKernelGetMainSocId()` == `0x840FD0` ⇒ Pro. The HV base PA differs by 0x800000. |
| **SPRX** | Sony's signed PRX module format (`.sprx`). The ELF loads many on startup for libkernel/libSceSysmodule bindings. |
