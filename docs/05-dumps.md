# 05 — On-disk artifacts

After a successful run the ELF writes two files into `/data/`:

```
/data/<maj>.<min>_kernel_<ktext_va>.bin
/data/<maj>.<min>_hv_<hv_pa>.bin
```

* `<maj>.<min>` — firmware major/minor derived from `kernel_get_fw_version()`:
  ```
  maj = (fw >> 24) & 0xff
  min = (fw >> 16) & 0xff
  ```
* `<ktext_va>` — the resolved kernel `.text` base (as a hex `uint64_t`, `%lx`).
* `<hv_pa>` — the model-specific HV loader physical address:
  * Regular: `0x62800000`
  * Pro:     `0x8E0800000`

## `dump_kernel_text`

Walks from `ktext(0)` forward, snprintf-formatted output ("`\r    0x%lx / 0x%lx`") shows chunked progress. Chunk size and total size come from `kernel_get_size` (debug string: `[+] finding kernel size...`, `[+] kernel size: 0x%lx (%lu MB), dumping from 0x%lx in 0x%x chunks -> %s`). Sanity-check: `[!] invalid kernel size 0x%lx` on a zero/malformed answer.

## `dump_hv_image`

Reads 0x2000000 bytes (32 MiB) starting at `g_hv_base_pa` via `kdldmap`. Chunks are the same.
Debug line: `[+] dumping HV from pa 0x%lx (ka 0x%lx, max 0x%lx) in 0x%x chunks -> %s`.
Guarded by `if (hv_pa == 0) [!] HV pa is 0, skipping HV dump`.

## File-system access

Standard POSIX via libkernel: `open`, `read`, `lseek`, `close`, `malloc`/`free`, with userland `kwrite`-backed reader on the kernel side. Errors format through:

```
<118>[%s] %s
<118>[%s] %s: %s
```

(syslog PRI `<118>` = user.info)
