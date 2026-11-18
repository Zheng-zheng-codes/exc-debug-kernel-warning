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
- Independent manual snapshot restore on 2026-11-18 reproduced the same
  `WARNING: arch/x86/kernel/traps.c:1286 at exc_debug_kernel`
- Fresh-VM replay of the full 324-entry recipe did **not** reproduce it in
  our tests (8/1/1 procs and 600 s runs)
- Evidence: `reproducer/repro.outcomes.jsonl`, `reproducer/repro.stats`,
  `reproducer/repro.report`, `logs/snapshot-restore-serial.txt`

## How to reproduce

The reliable method is the same state snapshot used for verification:
restore the exported QEMU state and then run the single testcase. The 4.5 GiB
state snapshot is not included in this repository because of its size; it is
available from the reporter on request.

```sh
# 1. restore the exported state, for example:
qemu-system-x86_64 ... -drive file=state/disk_qcow2-0.qcow2,if=ide,format=qcow2 \
    ... -loadvm kernsight-state ...

# 2. in the resumed guest, as root:
syz-execprog -executor=/syz-executor -repeat=1 -procs=1 \
    -sandbox=none /root/repro.prog
```

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

## Notes

- Crash title: `WARNING in exc_debug_kernel`
- Subsystem: x86 debug exception / io_uring setup
- The kernel WARN is `WARN_ON_ONCE(dr6 & DR_STEP)` in the #DB handler; the
  comment in traps.c explicitly mentions QEMU GDB stub leakage as a known
  source of this condition.
