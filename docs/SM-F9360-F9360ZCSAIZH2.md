# SM-F9360 / F9360ZCSAIZH2 — q4q (Galaxy Z Fold4, 2026-08 firmware)

Firmware `F9360ZCSAIZH2`, kernel `5.10.236-android12-9-2755199-abF9360ZCSAIZH2`,
Android 16 (SDK 36), Snapdragon 8+ Gen 1 (SM8450), locked bootloader, no
fastboot flashing. Build chain driven entirely from a temporary root obtained
through CVE-2026-43499 plus a KernelSU late-load.

## 1. Relation to the ZF1 profile

The ZH2 Image is the same length as the ZF1 Image (41 421 312 B) and the two
`/proc/config.gz` blobs are byte-identical, so every structure offset and the
`task_struct` layout are shared with
[`SM-F9360-F9360ZCSAIZF1.md`](SM-F9360-F9360ZCSAIZF1.md).

Only three data symbols move, all by `+0x380`:

| symbol | ZF1 | ZH2 |
| --- | --- | --- |
| `ashmem_fops` | `0x02078cb8` | `0x02079038` |
| `kmalloc_caches` | `0x020baea0` | `0x020bb220` |
| `anon_pipe_buf_ops` | `0x01efc028` | `0x01efc3a8` |

Everything else the payload uses — `ashmem_misc.fops`, the ashmem/configfs
dispatch functions, `init_task`, `root_task_group`, `selinux_state.enforcing`,
and the KASLR slide helpers — is unchanged. `src/targets/q4q-F9360ZCSAIZH2/`
therefore mirrors the ZF1 profile with those three values (plus the build
fingerprint) substituted.

## 2. KernelSU side: rebase onto v3.3.1

The ZF1 module was built from KernelSU v3.3.0 and reports `uapi_version = 2`.
KernelSU Manager v3.3.1 expects `KERNEL_SU_UAPI_VERSION = 5` and keeps showing
the red "kernel update required" banner no matter what `versionCode` the module
reports — the banner is a UAPI mismatch, not a version mismatch.

`kernelsu/android12-5.10_kernelsu-F9360ZCSAIZH2-c12-331.ko` is KernelSU v3.3.1
kernel sources with the Samsung KDP/RKP/DEFEX adaptation ported on top:

- `compat/samsung_kdp.c`, `compat/samsung_defex.c`, `include/ksu_samsung_kdp.h`
  carried over verbatim;
- `core/init.c` gains the KDP/DEFEX init and exit calls after the symbol
  resolver;
- `policy/app_profile.c` installs credentials through
  `ksu_samsung_kdp_commit_creds()` (falling back to `out_abort_creds` on
  failure) and calls `ksu_samsung_defex_sync_current()`;
- `hook/syscall_hook_manager.c` gains the Samsung sucompat kprobes
  (`execve`, `newfstatat`, `faccessat`, `statx`, `faccessat2`) and the
  `setresuid` kretprobe;
- upstream v3.3.1 code is kept: `execveat` handling, `webview_zygote_umount`,
  `module_blacklist`, `app_profile_init`. The v3.3.0-based Samsung diff also
  contains deletions that merely move those files back to older upstream
  revisions; copying them wholesale drops 3.3.1 behaviour.

### 2.1 The two offsets that panic the device if a rebuild gets them wrong

| field | device value | what a wrong build does |
| --- | --- | --- |
| `task_struct->cred` | `0x780` | the DEFEX kprobe loads `-1` and dereferences `0x3` (Oops) |
| `task_struct->stack` | `0x38` | the sucompat kprobe computes `task_pt_regs()` from the wrong stack pointer (Oops on the next `newfstatat`) |

`stack` is the trap that bites rebuilds: with `CONFIG_SHADOW_CALL_STACK` off,
`thread_info` loses `scs_base`/`scs_sp` (16 bytes) and `stack` compiles as
`0x28`. That option silently disappears when `make olddefconfig` runs without
`CC=clang` — kconfig probes with gcc, sets `CC_IS_CLANG=n`, and disables
`SHADOW_CALL_STACK`. Use the device `/proc/config.gz`, run `olddefconfig` and
`prepare` with the cross `CC`, and confirm the option landed in
`include/generated/autoconf.h` before building. Both offsets are worth
static-gating on every rebuild:

```sh
llvm-objdump -d kernelsu.ko   # ksu_samsung_defex_pre_handler  -> ldr x8, [x8, #0x780]
llvm-objdump -d kernelsu.ko   # samsung_sucompat_*_pre_handler -> ldr x9, [x8, #0x38]
```

## 3. Building the late-load binary (`ksud-*`)

Root My Galaxy downloads the `ksud-*` artifact and runs `ksud late-load`, which
loads the module embedded as `assets <kmi>_kernelsu.ko`. To rebuild one:

```sh
# 1. place the module under the KMI name ksud looks up
cp kernelsu.ko KernelSU/userspace/ksud/bin/aarch64/android12-5.10_kernelsu.ko

# 2. cross-compile ksud (rust target aarch64-linux-android)
export CC_aarch64_linux_android=$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android33-clang
export AR_aarch64_linux_android=$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ar
export BINDGEN_EXTRA_CLANG_ARGS_aarch64_linux_android="--sysroot=$SYSROOT -I$SYSROOT/usr/include -I$SYSROOT/usr/include/aarch64-linux-android --target=aarch64-linux-android33"
export LIBCLANG_PATH=<dir containing libclang.so/.dylib>
cargo build --release --target aarch64-linux-android -p ksud

# 3. result embeds the module; `strings ksud | grep android12-5.10_kernelsu.ko`
#    confirms the asset name, and `ksud late-load --help` lists the CLI flags
#    the root helper relies on (--kmi, --package-name, --allow-shell).
```

The KMI string for both q4q firmwares is `android12-5.10`
(`5.10.236-android12-…`), so the asset name is identical for ZF1 and ZH2 and
only the embedded bytes differ per profile.

## 4. What is verified on hardware

Verified 2026-10-29 on `SM-F9360`, `F9360ZCSAIZH2`, through the payload's
shell-domain chain (temporary root, then module late load):

- `symbols patched=219 missing=0 skipped=1` → `KernelSU loaded OK`;
- `/proc/modules` shows `kernelsu … Live`;
- `su -c id` → `uid=0(root) … context=u:r:ksu:s0`;
- Manager v3.3.1 reports `版本: 32680-5`, state "working [jailbreak mode]", and
  the "kernel update required" banner is gone;
- survives repeated `execve`/`stat` traffic (the pre-fix build died 0.8 s after
  init in `samsung_sucompat_newfstatat_pre_handler`).

Not yet verified: the app-domain (untrusted_app) execution path of
`cve-2026-43499-app.so` for either q4q firmware. The artifacts are built from
the same sources that produced the device-tested shell payload, at API 35
(device SDK 36); the app-domain chain still needs a hardware run through Root
My Galaxy before the profiles can be called device-tested end to end.

## 5. Artifacts

| file | notes |
| --- | --- |
| `artifacts/q4q-F9360ZCSAIZH2/cve-2026-43499-app.so` | app payload, 134 000 B |
| `kernelsu/android12-5.10_kernelsu-F9360ZCSAIZH2-c12-331.ko` | audit copy of the module |
| `kernelsu/ksud-q4q-F9360ZCSAIZH2-kdp` | late-load binary embedding that module |
