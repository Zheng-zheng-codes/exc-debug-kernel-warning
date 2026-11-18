# Reproducing WARNING in exc_debug_kernel from the state snapshot

This directory contains the verified QEMU/KVM state snapshot, compressed with
`xz -9` and split into 90 MiB parts. The archive contains:

- `disk.qcow2` — OS disk with the internal QEMU snapshot `kernsight-state`
- `trixie-base.img` — exact base image required by `disk.qcow2`
- `bzImage` — v7.4-rc3 kernel image
- `scratch-*.qcow2`, `logwrites.qcow2` — state devices
- `repro.prog` — single crashing syzkaller testcase
- `id_rsa`, `id_rsa.pub` — test-only SSH key for this guest
- `run-qemu.sh`, `reproduce.sh` — launch and one-command reproduction

The uncompressed bundle is about 6.9 GiB. The archive is about 908 MiB.

## Reassemble and run

```sh
cd snapshot
sha256sum -c SHA256SUMS

cat exc-debug-state-v7.4-rc3.tar.xz.part-* > /tmp/exc-debug-state-v7.4-rc3.tar.xz
sha256sum /tmp/exc-debug-state-v7.4-rc3.tar.xz
# expected:
# 711dab9e0378bfbe629f17ee09c1ed75a7591403b1d2161c29b23a15c1b8ccc2

xz -t /tmp/exc-debug-state-v7.4-rc3.tar.xz
mkdir -p /tmp/exc-debug-state
tar -xJf /tmp/exc-debug-state-v7.4-rc3.tar.xz -C /tmp/exc-debug-state
cd /tmp/exc-debug-state
./reproduce.sh
```

Expected:

```
REPRODUCED: WARNING in exc_debug_kernel
[...] WARNING: arch/x86/kernel/traps.c:1286 at exc_debug_kernel+0x1c5/0x230 [...]
```

`reproduce.sh` starts QEMU with `-loadvm kernsight-state`, waits for SSH, then
runs the testcase in the resumed guest with:

```sh
syz-execprog -executor=/syz-executor -procs=8 -repeat=0 -sandbox=none /root/repro.prog
```

`-repeat=0` is intentional. The warning is a race; the program repeats until
the warning fires. The script polls the serial log and exits as soon as
`traps.c:1286` appears. This method reproduced the warning on two consecutive
clean-snapshot runs.

## Requirements

- x86_64 Linux host with KVM enabled (`/dev/kvm`)
- QEMU; tested with QEMU 6.2.0
- `ssh`, `scp`, `sha256sum`, `xz`, `tar`
- at least 8 GiB free RAM and 12 GiB free disk

If port 2222 is busy:

```sh
SSH_PORT=2345 ./reproduce.sh
```

The bundle uses `-accel kvm -cpu host`; it is not expected to work under TCG.
The SSH key is a test-only key for this disposable guest.
