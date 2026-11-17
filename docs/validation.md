# Validation

Two virtualizers with different jobs. QEMU runs on every build and is scriptable. Oracle VM VirtualBox is a manual second opinion with a different firmware implementation and hardware-virtualized execution, and it gates every promotion to `stable`. There is no real-hardware target yet; one will be added once both virtualizers are reliably green and there is a driver set narrow enough to justify it.

---

## QEMU smoke test

`tools/smoke-test.sh`:

1. Builds the kernel and the hybrid BIOS/UEFI ISO.
2. Boots the ISO headless under `qemu-system-x86_64` on the BIOS path, 256 MiB of RAM, serial output to a file.
3. Polls the serial log for up to 50 seconds and exits early once both success lines are present.
4. Passes only if both lines are found, each checked separately:

```text
rose: hello, hardware
rose: root task: handoff ok
```

The first prints before paging is set up and is the earliest proof of life. The second prints only after the root task has read BootInfo, retyped a frame, mapped it, and written and read it back. They are checked as two separate matches because a single alternation pattern passes when only one line is present, which would hide a boot that stops halfway.

Under QEMU the root-task line takes about 20 seconds to appear, because the timer self-test waits out its full cycle budget before falling back (next section). Under VirtualBox it is near-instant.

The UEFI path runs separately with `tools/run-uefi.sh` (OVMF as two pflash drives, CODE read-only and VARS copied to a scratch file).

---

## The QEMU interrupt limitation

In the development QEMU setup (observed first with QEMU 10.2.1 under TCG software emulation), the PIT's IRQ0 never reaches the CPU. This was isolated as follows:

- PIC remap (IRQ0 to 7 on vectors 32 to 39) and PIT programming (mode 3, divisor 11931 for 100 Hz) were verified byte for byte in the disassembly.
- At runtime, `RFLAGS.IF` was 1 and the PIC mask register was `0xfe` (only IRQ0 unmasked). The PIC's interrupt request register read `0x01` (IRQ0 pending) and stayed there.
- A software `int 0x20` from the same code path dispatched immediately, so the IDT stub, dispatch, EOI, and tick counter were correct.
- QEMU's `-d int` trace showed no interrupt servicing during the wait. The behavior was identical on BIOS and UEFI and with TCG selected explicitly.
- The same ISO under VirtualBox took the interrupt immediately.

It was not pursued further as a QEMU issue. Consequences:

- The timer self-test bounds its wait with an RDTSC cycle budget and falls back to a software interrupt, so boot completes under QEMU.
- Anything that depends on real preemption (a blocked Send or Call resuming, two ring-3 tasks alternating, races against the timer) cannot be observed under QEMU and is confirmed in VirtualBox. Five defects so far surfaced only there ([engineering-log.md](engineering-log.md#found-only-under-virtualbox)).

---

## VirtualBox promotion gate

The same `rose.iso` is attached to the optical drive and the serial port is redirected to a raw file, so the check is the same two-line check as QEMU.

| Setting | Value | Reason |
|---|---|---|
| Type / version | Other, 64-bit | a 32-bit guest type hides long mode from the guest |
| Base memory | 256 MB minimum | matches QEMU |
| Chipset | PIIX3 | |
| Firmware | BIOS (EFI off) | VirtualBox's EFI is unreliable booting from CD; its legacy BIOS is still a different implementation from SeaBIOS, which is the point of the second opinion. UEFI is tested under QEMU with OVMF. |
| Processors | 1 | the kernel is single-core |
| PAE/NX | on, required | the kernel enables `EFER.NXE` and sets NX on every data, heap, and direct-map page; without PAE/NX the NX bit is reserved and paging faults at boot |
| Nested VT-x/AMD-V | off | |
| Paravirtualization | none | |
| Nested paging | on | |
| Graphics | VBoxVGA, 16 MB, 3D off | framebuffer console |
| Serial port 1 | COM1 (0x3F8, IRQ 4), raw file | log capture |

---

## Promotion checklist

A merge from `main` to `stable` requires all of:

1. The kernel builds clean, with zero warnings.
2. The QEMU smoke test passes.
3. A VirtualBox boot shows both smoke-test lines in the serial log.
4. The merge is recorded in the merge log with what it proves.

There are two branches. `main` is active work and may be broken between commits. `stable` receives only merges from `main` that pass the checklist and is never worked on directly.

## Merge discipline

A script in the repository mechanizes the checklist:

- Each change is built on its own branch in its own git worktree off `main`.
- The gate runs on the merged tree, not the branch alone. A failing gate leaves the target branch exactly as it was.
- Claims are checked rather than trusted: the commits must exist, the worktree must be clean, and a gate pass must be recorded for the exact head commit.
- Steps a machine cannot run are attested explicitly. On a machine with QEMU and `xorriso` the smoke test runs as part of the gate; the VirtualBox step is always attested by a person. Promotion to `stable` is refused without the attestations.
- The script never pushes.

---

## Tests that run on every boot

Each prints a result line to serial (and to the framebuffer console once it exists):

- Frame allocator: allocate, free, and confirm the freed frame is returned first.
- IDT: trigger a breakpoint and resume.
- Paging: map, write, read back, unmap.
- Address spaces: a page present in one AddressSpace is absent in another; switch CR3, write, read back, switch back.
- Heap: a `Box` and a growing `Vec` through the real `alloc` API; bytes return to the free list on drop.
- Timer: 50 hardware ticks (software fallback under QEMU).
- Scheduler: two cooperative tasks, deterministic regardless of timer delivery.
- Scheduled isolation: two tasks in separate AddressSpaces map the same virtual address to different frames and each reads only its own pattern across preemptions.
- CSpace: move, copy, mint with narrowed rights and a badge, and cascading revoke; a separate test revokes a 63-deep derivation chain in one call.
- PCI: enumerate the bus through legacy configuration mechanism #1. Under VirtualBox this finds devices QEMU does not emulate, a cross-check that real configuration space is being read.
- Root task: the handoff line described above.

An opt-in build replaces the root task with the older ring-3 self-test, which covers the CSpace, Untyped, Frame-Map, Configure, and IPC error paths the root program does not.

## Compile-fail tests

Two feature-gated builds must fail to compile. One passes a raw integer to the user mapper where a checked `UserVaddr` is required; the other constructs `UserVaddr` directly from outside its module. A clean build of either means the typed mapping boundary has a hole ([architecture.md](architecture.md#frame-map)).

---

## Promotion history

| Date | Promoted | Verified on |
|---|---|---|
| 2026-11-06 | Root task handoff, increment 1 | QEMU BIOS smoke test, VirtualBox |
| 2026-08-30 | Framebuffer console; also closed VirtualBox confirmation of IPC Call/Reply under real preemption | QEMU BIOS, VirtualBox (screen and serial log) |
| 2026-08-09 | Endpoint IPC increment 1 (Send, Receive) | QEMU BIOS and UEFI, VirtualBox (full round trip) |
| 2026-08-09 | Security audit fixes | QEMU BIOS and UEFI, VirtualBox |
| 2026-08-08 | PCI enumeration, Frame-Map, per-task kernel entry stacks | QEMU BIOS and UEFI, VirtualBox |
| 2026-08-01 | Retype extended to AddressSpace and Thread | QEMU BIOS, VirtualBox |
| 2026-08-01 | Capabilities and CSpace, CSpace syscalls, Untyped/Retype, the task 0 address-space fix | QEMU BIOS and UEFI, VirtualBox |
| 2026-07-25 | PIC remap and PIT timer | QEMU, VirtualBox (first confirmation of hardware IRQ0 delivery) |
| 2026-07-25 | GDT/IDT, kernel-owned page tables, kernel heap | QEMU BIOS and UEFI, VirtualBox |
| 2026-07-25 | Memory map parsing, frame allocator | QEMU BIOS and UEFI, VirtualBox |
| 2026-07-25 | First boot to the smoke-test line | QEMU BIOS and UEFI, VirtualBox |

## Not covered

- Real hardware.
- UEFI under VirtualBox (deliberately; see the firmware row above).
- The UEFI path has not been re-run since the IPC Call/Reply increment, so the framebuffer console and root task increments are confirmed on BIOS only.
- The checks added by the 2026-08-09 audit are confirmed not to regress normal boots; no test attempts the attacks they block.
- SMP.
