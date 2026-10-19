# 🩸 Byepervisor Reborn — PS5 Hypervisor Takedown, Reverse-Engineered

> **A full write-up of the PS5 proof-of-concept ELF that pierces Sony's AMD-SVM hypervisor, clears the `XOTEXT` bit on kernel pages, survives a sleep/resume cycle, and dumps `kernel.text` + `hvcore` to `/data/`.**

[![License: MIT (docs)](https://img.shields.io/badge/License-MIT_(docs)-blue.svg)](./LICENSE)
![Platform](https://img.shields.io/badge/Target-PlayStation%205-003791?logo=playstation&logoColor=white)
![Arch](https://img.shields.io/badge/arch-x86__64-informational)
![Format](https://img.shields.io/badge/format-ELF-black)
![Research](https://img.shields.io/badge/scope-research%20only-red)

---

## Why read this

Modern consoles don't trust their own kernel. The PS5 ships an **AMD-SVM nested-paging hypervisor** that owns the real page tables and marks the kernel's `.text` as **execute-only** (`XOTEXT`) — even root in the kernel can't read it back. On paper, kernel RW ≠ kernel code dumping, and HV dumping is "impossible".

Byepervisor Reborn says *impossible, sure*. And then it:

1. **Walks the nested page tables by hand** to find the PDE/PTE for every `.text` page.
2. **Clears bit 58** (the HV's `XOTEXT` marker) and **sets bit 1** (WRITE). Kernel code is now readable *and* writable.
3. **Stages a tiny gadget chain** (`khook`) inside the HV's own suspend/resume path.
4. Calls `sceSystemStateMgrEnterStandby()` — the console goes to sleep.
5. On wake, the HV ticks through the planted chain, drops a cookie `0xCAFE133741414141`, and the hypervisor is **neutered**.
6. The payload loops until it sees the cookie, writes `/data/<fw>_kernel_<ktext>.bin` and `/data/<fw>_hv_<pa>.bin`, and prints:

   > `Byepervisor Reborn: Welcome back, hypervisor is defeated! Cookie? 0x...`

This repo documents the mechanism, bit by bit. No new exploit chain, no DRM bypass — the write-up itself is the artifact.

## ✨ What you get in this repo

| | |
|---|---|
| 📄 `README.md` | this page |
| 📘 `docs/01-main-flow.md` | annotated decompilation of `main()` |
| 🔓 `docs/02-xotext-defeat.md` | the page-table walk + the XOTEXT bit |
| 🪝 `docs/03-resume-hook.md` | `install_resume_hook()` disassembly |
| ⛓️ `docs/04-khook-chain.md` | the gadget-chain framework (`kh_*`) |
| 💾 `docs/05-dumps.md` | on-disk dump format + file layout |
| 🗒️ `strings/all_strings.txt` | all 543 static strings |
| 🧪 `binary/byepervisor_reborn.elf` | the artifact itself (+ `SHA256SUMS`) |

## 🧬 At a glance

```
┌────────────────── userland ELF ──────────────────┐
│                                                   │
│  main()                                           │
│   ├─ fingerprint fw + SoC (Pro vs Regular)        │
│   ├─ bump authid → 0x4800000000000007 (debug)     │
│   ├─ for every kernel .text page                  │
│   │     clear XOTEXT  (bit 58)                    │
│   │     set   WRITE   (bit 1)                     │
│   ├─ stage break_hv shellcode @ kdlsym(9)+0x1000  │
│   ├─ install_resume_hook()  ──┐                   │
│   ├─ sleep(3)                 │ KHOOKED\x01       │
│   └─ EnterStandby()           │ persistence       │
│                               ▼                   │
└───────────────────────┬───────────────────────────┘
                        │ (physical wake)
                        ▼
    ┌───── HV resume path (hvcore) ─────┐
    │ runs the planted kh_chain:        │
    │  • memcpy break_hv → scratch       │
    │  • patch HV ptr slots              │
    │  • drop cookie into user_data      │
    └──────────────┬─────────────────────┘
                   │
                   ▼
         userland polls kread64(ksym9+0x440)
         sees cookie → dumps kernel + HV to /data/
```

Full breakdown: start with [`docs/01-main-flow.md`](docs/01-main-flow.md).

## 🏃 How to run it (research only)

> **This is a kernel-level PS5 payload. Do not run it on a console you depend on. It can brick, soft-lock, or corrupt `/data/`.** Read first, run never, unless you own the hardware and understand what every page-table write is doing.

### Prerequisites

* A **PS5** on a vulnerable firmware (`kdlsym(0x6)` must resolve — the ELF prints the firmware it detected; if it says `Firmware is not supported`, you can't).
* A prior **kernel R/W primitive** — this ELF is a *payload*, not an entry chain. The typical setup is the WebKit → kernel exploit chain shipped with [`ps5-payload-dev/pacbrew-pkg`](https://github.com/ps5-payload-dev/pacbrew-pkg) / [`ps5-payload-sdk`](https://github.com/ps5-payload-dev/sdk).
* A **payload launcher** such as [`ps5-payload-elfldr`](https://github.com/ps5-payload-dev/elfldr), running on port `9021`.
* About **100 MB free in `/data/`** for the kernel + HV dumps.

### Steps

```bash
# 1. Verify the hash
shasum -a 256 -c binary/SHA256SUMS

# 2. On the PS5, run your webkit->kernel chain so elfldr is listening.
#    From a laptop on the same LAN:
nc -q 0 <PS5_IP> 9021 < binary/byepervisor_reborn.elf
```

The payload prints to the console's `klog` (socat / `nc <PS5_IP> 3232` depending on your setup) and shows on-screen notifications:

```
[+] Byepervisor reborn v1
[+] Detected firmware: 5.00
[+] Regular model detected, HV loader PA = 0x62800000
[+] Disabling xotext + enabling write...
[+] Installing hook...
[+] Hook installed, enter sleep and resume!
Byepervisor Reborn: Entering rest mode in 3 secs
```

The console will enter **Rest Mode**. Wake it with the controller/power button. After a few seconds, you'll see:

```
Byepervisor Reborn: Welcome back, hypervisor is defeated! Cookie? 0xCAFE133741414141
[+] Uninstalling hook
```

The dumps land in `/data/`:

```
/data/5.00_kernel_ffffffff82000000.bin
/data/5.00_hv_62800000.bin
```

Pull them with FTP or SMB (whichever payload you already have running).

### Building from source

Source isn't published in this repo — only the compiled ELF is included for hash verification. If you want to build an equivalent payload yourself:

* Grab the [`ps5-payload-sdk`](https://github.com/ps5-payload-dev/sdk).
* Study the write-ups in [`docs/`](docs/).
* The missing pieces are the `kh_*` library (khook) and the `break_hv` shellcode blob — publicly tracked by the research community.

## 🧠 How it's analyzed

Static only. Hopper Disassembler driving the `rea` MCP server for machine-readable dumps:

```bash
# procedure list, strings, pseudo-code, xrefs
# everything in docs/ comes from these three primitives:
#   procedure_pseudo_code
#   search_strings
#   search_procedures
```

Zero dynamic analysis — no console was pwned to produce these docs. Everything you read is derived from the bytes in `binary/byepervisor_reborn.elf` and matched against the public PS5-security literature.

## 📎 References

* [SpecterDev — PS5 kernel research](https://github.com/Cryptogenic)
* [`ps5-payload-dev/sdk`](https://github.com/ps5-payload-dev/sdk) — payload runtime + headers
* [`ps5-payload-dev/elfldr`](https://github.com/ps5-payload-dev/elfldr) — ELF loader
* [AMD64 APM Vol. 2, ch. 15](https://www.amd.com/system/files/TechDocs/24593.pdf) — SVM nested paging
* flat_z, Chendo, znullptr — PS5 HV research threads

## 🤝 Contributing

Pull requests welcome, especially:

* Corrections to decompile annotations.
* Diagrams (SVG preferred) of the kh_chain structure.
* Firmware-version → `kdlsym` index map, where it has been made public.
* Translations.

Please keep the repo **documentation-only**. No exploit sources, no firmware blobs, no dumps.

## ⚖️ Legal / Ethics

Everything in this repo is **after-the-fact documentation** of a publicly circulated PoC. Nothing here is new, nothing here is a weapon. Written for:

* Platform-security researchers who want to understand modern console hypervisors.
* OS-dev students who've never touched nested paging in anger.
* People exercising their right to study the hardware they own.

Not for: piracy, cheat development, DRM bypass. Those use-cases are both out of scope and not something this repo helps with.

The included ELF is third-party; its author retains the copyright. Documentation (`docs/`, `README.md`, `strings/`) is MIT.

---

<sup>If this helped you understand how a hypervisor actually enforces XOM, star the repo — it's the only metric that tells me to write more of these.</sup>
