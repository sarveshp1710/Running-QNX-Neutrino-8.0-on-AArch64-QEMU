# Running QNX 8.0 AArch64le on QEMU (QEMU 8)
## Errors Faced, Root Causes, and Fixes

This document explains the **actual problems encountered** while emulating  
**QNX Neutrino RTOS 8.0 (AArch64le)** on **QEMU (version 8)** and how they were
resolved by analyzing QNX startup internals and Momentics-generated targets.

This is **not a full installation guide**.  
It focuses on **debugging, root causes, and solutions**.

---

## Environment

- Host OS: Ubuntu Linux  
- QNX: SDP 8.0  
- Emulator: QEMU 8 (`qemu-system-aarch64`)  
- Target architecture: AArch64le  
- Image base: `com.qnx.qnx800.quickstart.qemu`

x86_64 QNX works out-of-the-box.  
All issues documented here are **AArch64 QEMU–specific**.

---

## Error 1: `startup-qemu-virt` Not Found

### Symptom

Running:

```sh
mkqnximage --type=qemu --arch=aarch64le
```

fails with:

```
output/build/ifs.build:23 Host file 'startup-qemu-virt' not available
```

### Root Cause

The base QNX QEMU quickstart image includes **only x86 startup files**.

For AArch64 QEMU:
- `startup-qemu-virt` is **not included by default**
- It comes from separate target packages

### Fix

Install the following from **QNX Software Center**:

- `com.qnx.qnx800.target.qemuvirt`
- `com.qnx.qnx800.target.driver.virtio.devb`

After installation:
- `startup-qemu-virt` becomes available
- `mkqnximage` proceeds past IFS generation

---

## Error 2: slogger / SLM Boot Failure on ARM QEMU

### Symptom

The system boots but fails with errors like:

```
slog2_api: cannot connect to slogger2
Unable to start "uname" (2)
slm: unexpected 'SLM:command'
Unable to access "/dev/ser1"
```

### Root Cause

- **SLM (System Log Manager) starts too early**
- `/dev/slog2` is not yet created by `slogger2`
- ARM QEMU has slower, non-deterministic device startup
- This creates a **race condition**

This issue does **not** appear on:
- x86_64 QEMU
- VMware
- Real hardware

---

## Why Momentics Targets Do Not Fail
I took Momentics Target as reference because,
Momentics AArch64 QEMU targets:

- Disable SLM by default
- Assume minimal early logging
- Avoid dependencies on `/dev/slog2` during early boot

Manual images enable SLM, which breaks ARM QEMU timing.

---

## Fix: Disable SLM for AArch64 QEMU

Edit the following files:

```
~/qnx800/images/qemu/qemu/local/options
~/qnx800/images/qemu/qemu/output/options
```

Change:

```sh
OPT_SLM='yes'
```

to:

```sh
OPT_SLM='no'
```

Then rebuild and run:

```sh
mkqnximage --clean --build
mkqnximage --run
```

Result:
- Successful AArch64 boot
- `slogger2` initializes correctly
- No early startup failure

---

## Error 3: No GUI / Screen on AArch64 QEMU

### Observation

After successful boot:
- No Screen
- No `/dev/screen`
- No GUI splash or console art

### Explanation

This is **based on my observation, I may be wrong**.

QNX Screen:
- Works on x86_64 (VMware / QEMU)
- Works on real ARM boards
- **Does NOT work on ARM QEMU**

ARM QEMU runs **headless by design**.  
There is no Screen Board Support package for ARM QEMU.

---

## Summary of Fixes

| Issue | Cause | Fix |
|-----|-----|-----|
| `startup-qemu-virt` missing | Target packages not installed | Install `qemuvirt` + virtio devb |
| slogger / SLM failure | Early startup race | Disable SLM |
| No GUI on ARM QEMU | Unsupported | Expected behavior |

---

## Status

- x86_64 QNX with GUI: Working  
- AArch64 QNX on QEMU (headless): Working  
- AArch64 GUI on QEMU: Not supported  (My conclusion)

---

*Thank you*
