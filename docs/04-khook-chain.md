# 04 — The khook framework

Byepervisor Reborn ships its own minimal gadget-chain engine (`kh_*` functions). It is not SROP, not JOP; it's a tiny state machine the HV happens to tick through on resume.

## State object (`kh_ctx_t`)

Fields observed in call-order:

| Field | Setter | Content |
|-------|--------|---------|
| rw backends | `kh_init` | `kread`, `kwrite` function pointers |
| kernel .text base | `kh_init` | `ktext(0)` |
| ksym9 | `kh_init` | state scratch / staging base |
| hook slot id | `kh_init` | fixed `0x6` |
| PT bases | `kh_set_tables` | `kdlsym(0xA)` (PML4), `kdlsym(0xB)` (direct map) |
| core ctx | `kh_set_core` | caller-supplied |
| hook target | `kh_set_hook_target` | `kdlsym(0x6)` — HV address executed on resume |
| resume rsi | `kh_set_resume_rsi` | scratch `kdldmap(0x3FFFF000)` |
| resume rdx | `kh_set_resume_rdx` | CR3 for the kernel address space |
| user_data | `kh_user_data_addr` | returns the scratch slot for cookie I/O |

## Chain API

```
kh_chain_init(&c, buf, cap)                  // reset
kh_chain_store(&c, dst_va, qword)            // 8-byte poke
kh_chain_g(&c, gadget_id)                    // generic gadget selector
kh_chain_q(&c, qword)                        // qword argument to prior gadget
kh_chain_memcpy(&c, dst_va, src_va, len)     // bulk copy
```

Gadget IDs used:

| id | effect (inferred) |
|----|-------------------|
| 0x3 | pop/store arg into user_data+8 |
| 0x6 | enter hook target with provided CR3 |
| 0xD | sync / tlb-flush / barrier |

## Chain layout dropped by `install_resume_hook`

```
store ud + 0,   0xCAFE133741414141     ; cookie seed, user_data[0]
g 0x6 ; q cr3                           ; enter HV with kernel CR3
g 0x3 ; q ud + 8                        ; stash gadget return at ud+8
g 0xD                                   ; barrier
memcpy(dldmap(0x3FFFF000), ksym9+0x1000, 0xF00)
store dldmap(0x3FFFFF00), kdlsym(0x7)
store dldmap(0x3FFFFF08), kdldmap(0)
store dldmap(0x3FFFFF10), ud
store dldmap(0x3FFFFF18), g_hv_base_pa  (- 0x800000 on Pro)
store dldmap(0x3FFFFF20), g_hv_base_pa + rawoff(0xC)
store dldmap(0x3FFFFF28), g_hv_base_pa + rawoff(0xD)
```

The final `kh_install` serializes this into whatever on-chip queue the hypervisor's resume handler already consults, then `kh_trigger` arms it.

## Markers summary

| Const | ASCII | Role |
|---|---|---|
| `0x4B484F4F4B454401` | `KHOOKED\x01` | persistence flag at `ksym9+0x600` |
| `0xCAFE133741414141` | — | cookie seed; "defeated" when readback changes |

## uninstall_hook

Not re-dumped here but is a thin inverse: zeroes the persistence marker and `kh_remove`s the chain. See the string `[+] Uninstalling hook` cross-ref.
