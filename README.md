# WARNING in exc_debug_kernel (arch/x86/kernel/traps.c:1286, v7.4-rc3)

Syzkaller-found x86 debug-exception warning on Linux v7.4-rc3, reproduced
and verified by the KernSight state-primed pipeline using the exported QEMU
state snapshot (`snapshot_restore`). The warning is state-dependent: replaying
the same 324-entry recipe on a fresh VM did not reproduce it in our tests.

## Summary

The x86 #DB handler warns because `DR_STEP` is set when the kernel did not
expect single-step:

```
dr6 & DR_STEP
WARNING: arch/x86/kernel/traps.c:1286 at exc_debug_kernel+0x1c5/0x230
arch/x86/kernel/traps.c:1286, CPU#3: syz.1.18/71843
```

Relevant source:

```c
if (WARN_ON_ONCE(dr6 & DR_STEP))
    regs->flags &= ~X86_EFLAGS_TF;
```

Call trace:

```
<#DB>
 asm_exc_debug+0x1e/0x40
  exc_debug_kernel+0x1c5/0x230
   _copy_from_user+0x97/0xd0
    copy_from_user
     io_uring_setup+0xb3/0x21b0
      __x64_sys_io_uring_setup
       do_syscall_64
```

The warning fires while copying the `io_uring_setup` parameters from user
space. The source comment notes that QEMU's GDB stub can leak #DB into the
guest; this may be an emulation/debug-register artifact rather than a
kernel defect. We report it because it is user-triggerable and reproducible
in the syzkaller VM.

## Verification

- Pipeline attempt 1-2: budget exhausted
- Pipeline attempt 3: `confirmed=true`, `status=verified`
- Time: `2026-11-17T16:49:38Z`
- Method: restore the exported pre-crash QEMU state (`state/`,
  `-loadvm kernsight-state`), then run the testcase
- 323 replay programs were used to construct the pre-crash state
- Candidate 1 reproduced the warning from the frozen state
- Independent manual snapshot restores on 2026-11-18 reproduced the same
  `WARNING: arch/x86/kernel/traps.c:1286 at exc_debug_kernel`
- The reliable trigger is `syz-execprog -procs=8 -repeat=0` (repeat until
  the race fires); two consecutive clean-snapshot runs reproduced the warning
- Fresh-VM replay of the full 324-entry recipe did **not** reproduce it in
  our tests (8/1/1 procs and 600 s runs)
- Evidence: `reproducer/repro.outcomes.jsonl`, `reproducer/repro.stats`,
  `reproducer/repro.report`, `logs/snapshot-restore-serial.txt`

## How to reproduce

The verified state snapshot is included in this repository as a compressed,
split archive under `snapshot/`. It contains the QEMU state (`kernsight-state`,
4.5 GiB overlay), the exact base image, the kernel image, scratch disks, SSH
test key, `repro.prog`, and ready-to-run `run-qemu.sh` / `reproduce.sh`.

```sh
cd snapshot
sha256sum -c SHA256SUMS

cat exc-debug-state-v7.4-rc3.tar.xz.part-* > /tmp/exc-debug-state-v7.4-rc3.tar.xz
sha256sum /tmp/exc-debug-state-v7.4-rc3.tar.xz
# 711dab9e0378bfbe629f17ee09c1ed75a7591403b1d2161c29b23a15c1b8ccc2

xz -t /tmp/exc-debug-state-v7.4-rc3.tar.xz
mkdir -p /tmp/exc-debug-state
tar -xJf /tmp/exc-debug-state-v7.4-rc3.tar.xz -C /tmp/exc-debug-state
cd /tmp/exc-debug-state
./reproduce.sh
```

Expected output:

```
REPRODUCED: WARNING in exc_debug_kernel
[ ... ] WARNING: arch/x86/kernel/traps.c:1286 at exc_debug_kernel+0x1c5/0x230 ...
```

`reproduce.sh` starts QEMU with `-loadvm kernsight-state`, waits for the
guest, and runs:

```sh
syz-execprog -executor=/syz-executor -procs=8 -repeat=0 \
    -sandbox=none /root/repro.prog
```

`-repeat=0` is intentional: the warning is a race, so the program repeats
until the warning fires. The script polls the serial log and exits as soon as
`traps.c:1286` appears. Two consecutive clean-snapshot runs reproduced it.

Requirements: x86_64 Linux host with KVM, QEMU (tested with QEMU 6.2.0),
`ssh`/`scp`, at least 8 GiB free RAM, and about 12 GiB free disk. Override the
host SSH port with `SSH_PORT=... ./reproduce.sh` if 2222 is busy.

The state snapshot already contains the post-replay, pre-crash kernel state;
only `repro.prog` is needed after restore. `repro.kernsight.prog` is the
self-contained 324-entry recipe for reconstruction on a fresh VM, but that
path was not reliable for this state-dependent bug.

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc3 with CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y,
CONFIG_DEBUG_INFO=y, booted with `oops=panic` and `panic_on_warn=1`. See
`environment.txt` and `kernel.config`.

## Files

- `description`, `environment.txt`, `kernel.config`
- `reproducer/repro.kernsight.prog` — 324-entry state recipe
- `reproducer/repro.priming.prog`, `reproducer/repro.prog`
- `reproducer/repro.report`, `repro.stats`, `repro.outcomes.jsonl`,
  `repro.context.json`
- `logs/log-0.txt`, `logs/report-0.txt`, `logs/machineInfo-0.txt`,
  `logs/title-stat.json`, `logs/snapshot-restore-serial.txt`
- `snapshot/exc-debug-state-v7.4-rc3.tar.xz.part-*` — compressed state bundle
- `snapshot/SHA256SUMS` — part checksums

## Notes

- Crash title: `WARNING in exc_debug_kernel`
- Subsystem: x86 debug exception / io_uring setup
- The kernel WARN is `WARN_ON_ONCE(dr6 & DR_STEP)` in the #DB handler; the
  comment in traps.c explicitly mentions QEMU GDB stub leakage as a known
  source of this condition.
