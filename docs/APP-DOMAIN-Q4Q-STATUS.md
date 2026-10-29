# q4q app-domain status (SM-F9360 / F9360ZCSAIZF1 + ZH2)

Status as of 2026-10-29: the **delivery half of the app chain works**, the
**app-domain exploit does not complete yet**. Do not wire q4q into
`support/targets-v3.json` until the remaining stage below is fixed — the payload
currently panics the kernel, which reboots the user's device.

## 1. What was verified working (real device, app process)

Root My Galaxy (0.2.65) driven against a local copy of this repo:

```
[+] 支持配置: q4q-F9360ZCSAIZH2
[*] 正在下载 exploit   → [*] exploit 已验证
[*] 正在下载 KernelSU  → [*] KernelSU 已验证
[app] loading verified payload=…/payloads/q4q-F9360ZCSAIZH2/cve-2026-43499-app.so
[+] startup context uid=10293 attr=u:r:untrusted_app:s0:c37,c257,c512,c768
[+] startup limits NoNewPrivs=0 Seccomp=2 Seccomp_filters=1
[+] build config label=q4q-F9360ZCSAIZH2-app-physical-p0-oracle
[+] p0 profile phys_offset=0x80000000 kernel_phys_load=0xa8000000 delta=0x28000000
```

So profile resolution, artifact download, size/hash verification, the app's
root-helper hand-off and the payload load into `untrusted_app` all behave. Both
q4q profiles (ZF1 and ZH2) load and run identically.

## 2. Blocker #1 (fixed): `SLIDE_KSNITCH_APPENDED_FUTEXES` was half the signal

With the inherited value `1024` the pipe stage never produced a candidate:

```
only found 0 collisions -> cannot continue
[!] pipe KernelSnitch collision finding failed
ksnitch timing diag approx_time=14 threshold=140 min=3 max=110 mean=4 n=98302
```

The timing side channel itself was healthy (98 302 probes, span 3–110), but the
piled-up futex bucket never exceeded the 10× threshold. Every other target in
this tree uses `2048`:

```
a15-A155NKSS6BYH1  a36xq-A366WVLS3AYG1  dm1q  dm2q  dm3q …  = 2048
q4q-F9360ZCSAIZF1 / -ZH2 (before)          = 1024
```

With `2048` q4q matches the a36xq 5.10 profile and the wall disappears:

```
[*] KernelSnitch profile mode=1 fast=0 appended=2048 repeat=64 average=8
[*] p0 pipe oracle prepared base=ffffff89a4558000 pipes=240 gate_slots=1
```

## 3. Blocker #2 (open): allocator corruption in the page stage

Immediately after the oracle is prepared, the payload's next `clone()` dies in
the buddy allocator — the physical-write/reclaim stage has corrupted a freelist:

```
Unable to handle kernel paging request at virtual address ffffffc050293c68
Internal error: Oops: 96000007 [#1] PREEMPT SMP
pc : __list_del_entry_valid+0x48/0xcc
lr : move_freepages_block+0x1dc/0x24c
Call trace: move_freepages_block / steal_suitable_fallback / rmqueue /
            get_page_from_freelist / __alloc_pages_nodemask / allocate_slab /
            dup_mmap / copy_mm / copy_process / kernel_clone / __arm64_sys_clone
Kernel panic - not syncing: Oops: Fatal exception
```

Raw excerpt: [`q4q-app-panic-20261029.txt`](q4q-app-panic-20261029.txt).

### Ruled out

| hypothesis | measurement |
| --- | --- |
| CPU-count geometry wrong in app domain | `sysconf(_SC_NPROCESSORS_ONLN)=8`, `_CONF=8`, `/sys/.../online=0-7`; `top-app`/`foreground` cpusets are `0-7`. Only `sched_getaffinity` is narrower (6–7 of 8), which the payload does not use. |
| Samsung cpuset starving the exploit | app `Cpus_allowed_list 0-5,7` vs shell `0-7` — a difference, but the payload's `cpu_count` comes from `sysconf`, so lane count is unaffected. |
| `mm_struct` slab geometry | `/proc/slabinfo`: `mm_struct … 960 34 8` → object 0x3c0 and 8 pages/slab confirm `MM_STRUCT_SZ 960`, `MM_ORDER 3`. |
| Machine load | runs started at loadavg 2.3–2.5, the same window in which the shell-domain chain succeeds on this device. |
| Profile mismatch (ZF1 vs ZH2) | both profiles fail at the same stage with identical p0 profile values. |

## 4. What still differs from the shell chain

`src/common.h` gives the app build 2 attempts where the shell build gets 12/72:

| knob | shell | app |
| --- | --- | --- |
| `PIPE_MAX_ATTEMPTS` | 12 | 1 |
| `FOPS_KERNEL_PAGE_SETUP_ATTEMPTS` | 72 | 2 |
| `SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS` | 12 | 2 |

The app path therefore cannot brute-force the page layout, and each failing
geometry write lands in the buddy allocator instead of being retried. Candidate
next steps, in the order worth trying on hardware: instrument the reclaim
geometry (`pipe=240`, `gate_slots=1`) against the assumptions in `src/pipe.c`,
compare with a36xq's `SLIDE_PSELECT_TIMEOUT_NSEC` (100 ms there, 500 ms here),
and only then touch `KSNITCH_COLLISIONS`.

## 5. Shell-domain chain remains the supported path

The shell chain (payload binary + `ksud late-load`, KernelSU v3.3.1 / UAPI 5)
is device-verified on this exact firmware — see
[`SM-F9360-F9360ZCSAIZH2.md`](SM-F9360-F9360ZCSAIZH2.md). That is what
`kernelsu/ksud-q4q-*-kdp` and the module in `kernelsu/` are for.
