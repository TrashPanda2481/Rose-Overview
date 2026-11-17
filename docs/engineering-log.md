# Engineering log

A reference of defects diagnosed and resolved during development. Each entry gives the symptom, how it was isolated, the root cause, the fix, and how the fix was verified. Toolchain and environment issues are grouped in a table after the defects. Open findings from the most recent audit are summarized by category at the end.

Several of these defects were latent for some time before they surfaced. The common reason is that the development QEMU setup does not deliver hardware timer interrupts (see [validation.md](validation.md#the-qemu-interrupt-limitation)), so code paths that depend on real preemption ran for the first time under VirtualBox.

---

## Found only under VirtualBox

### Scheduler panic on the first real timer tick

- **Symptom:** QEMU booted cleanly through the scheduler self-test every time. VirtualBox panicked right after the first timer interrupt: `attempt to calculate the remainder with a divisor of zero` in the scheduler.
- **Diagnosis:** the panic site was the round-robin step in `tick()`, which computes the next task index modulo the task count. The timer handler calls `tick()` on every interrupt.
- **Cause:** `kernel_main` enabled interrupts (PIC and PIT initialization) before initializing the scheduler, which registers task 0. QEMU never delivered the interrupt, so the window was never exercised. VirtualBox delivered ticks as soon as interrupts were enabled; they landed on an empty task list, and the fifth reached the modulo.
- **Fix:** initialize the scheduler immediately before the PIC, so task 0 always exists before interrupts are enabled. `tick()` also returns early on an empty queue, since it is reachable from an interrupt whose timing the kernel does not control.
- **Result:** VirtualBox boots through the scheduler self-test with the same result line as QEMU, with real ticks completing the timer self-test in 510 ms.

### Timer self-test reported a working timer as absent

- **Symptom:** under VirtualBox, the first-tick log line fired and the tick counter advanced, but the timer self-test still reported no hardware interrupt within its wait and fell back to a software interrupt.
- **Diagnosis:** the self-test bounds its wait with an RDTSC cycle budget, so the elapsed real time depends on the host clock rate.
- **Cause:** the budget was computed from an assumed minimum clock rate (500 MHz). Real elapsed time is budget divided by actual rate, so on a multi-GHz host the "about 2 seconds" wait ended in a few hundred milliseconds, before 50 ticks at 100 Hz.
- **Fix:** compute the budget from an assumed maximum clock rate (8 GHz), so the wait is at least the intended duration on any realistic host.
- **Result:** VirtualBox reports `50 ticks elapsed via hardware irq0, 510ms since boot`. The timer itself had been correct; only the test's bound was wrong.

### CR3 reverted to the kernel address space under real preemption

- **Symptom:** QEMU ran the ring-3 self-test cleanly. VirtualBox completed nine syscalls, then page-faulted on the next ring-3 instruction (not-present, user, instruction fetch) at an address inside the program's own code page, which had been executing moments earlier.
- **Diagnosis:** the faulting address was exactly the instruction after the ninth syscall's return. Everything before it, on the same page, had executed. That pointed at the active page tables changing between two instructions, not at a mapping error.
- **Cause:** the self-test switched CR3 into the program's address space directly, bypassing the scheduler. Task 0's recorded address space still named the kernel's. On real preemption, the scheduler rotated through other tasks and, on returning to task 0, reloaded CR3 from the stale record. The program resumed under page tables where its code was not mapped. Adding four syscalls to the program had pushed its run time past the first 50 ms timeslice on VirtualBox.
- **Fix:** a scheduler call that updates the current task's recorded address space and CR3 together, with interrupts disabled across both.
- **Result:** VirtualBox runs the full ring-3 sequence with real ticks interleaved and no fault.

### A configured second task faulted on its first instruction

- **Symptom:** VirtualBox completed the full ring-3 self-test, then faulted at exactly the second task's entry point, on its first instruction fetch.
- **Diagnosis:** this was the first run in which real preemption ever switched into the task created by the Configure syscall. A code comment claimed the task's address space was the same one its code had been mapped into.
- **Cause:** the comment was wrong. Retype always creates a new, empty AddressSpace. The second task's code and stack were mapped into the caller's address space, and Configure scheduled the task in the new, empty one.
- **Fix:** add the `SYS_FRAME_MAP` syscall, so a holder of a Frame capability and an AddressSpace capability can map the frame into that address space, and map the second task's code and stack into the address space it actually runs in.
- **Result:** the original fault is gone in VirtualBox; the second task executes and makes its own syscall. A different fault then appeared one instruction later (next entry).

### Fault on the instruction after a ring-3 syscall returned

- **Symptom:** after the previous fix, the second task's first syscall completed, and the very next instruction fetch faulted (not-present, user, instruction fetch), on the same code page it had just executed from.
- **Diagnosis:** byte-counting the program placed the faulting address on the first instruction after the kernel's `iretq`. The address space handle was ruled out (Configure and Frame-Map reconstruct it identically), and a nested interrupt inside the syscall handler was ruled out (the gate is an interrupt gate, so interrupts are off). The first program had already survived many real preemptions alone, so ring-3 residency by itself was not the trigger. What was new was a second ring-3 task.
- **Cause:** `TSS.RSP0`, the stack the CPU loads on every ring-3 to ring-0 transition, was set once at boot and shared by all tasks. A task preempted inside the kernel leaves its saved state, including the hardware-pushed interrupt frame, on that stack. Once two ring-3 tasks alternated, their kernel entries overwrote each other's saved state.
- **Fix:** each task carries its own kernel entry stack, and the scheduler reloads `TSS.RSP0` on every switch-in alongside CR3, with the same interrupts-off update discipline.
- **Result:** VirtualBox runs both ring-3 tasks under real preemption without a fault. This was the first confirmed run of two ring-3 tasks coexisting.

---

## Boot, memory, and scheduling

### Double fault right after loading kernel-owned page tables

- **Symptom:** boot never reached the "page tables loaded" line; the serial log showed a double fault a few instructions after `mov cr3`.
- **Diagnosis:** the faulting instruction was a stack access shortly after the switch, and the first page fault could not be delivered cleanly, which pointed at the stack itself being unmapped.
- **Cause:** the new tables mapped the direct map only for usable memory. Limine's boot stack, still in use at the moment of the switch, lives in bootloader-reclaimable memory.
- **Fix:** map bootloader-reclaimable memory through the direct map as well. This changes what is mapped, not what the frame allocator hands out.
- **Result:** boot proceeds past the switch on QEMU (BIOS and UEFI) and VirtualBox.

### Spawned tasks faulted at address zero on first switch-in

- **Symptom:** the first scheduler self-test panicked with a page fault at `rip=0` immediately after the first `yield_now()`.
- **Cause:** a new task's stack is seeded with a synthetic saved-register frame so the context switch "returns" into a trampoline. The frame was written in reverse, so the trampoline address was popped into a register and a zero word was used as the return address.
- **Fix:** write the frame in declaration order, so the trampoline is the last value popped, by `ret`.
- **Result:** tasks start and interleave as expected.

### Scheduler self-test hung after the first task finished

- **Symptom:** after the previous fix, boot hung silently after the timer self-test.
- **Cause:** test tasks that finished their work spun without yielding. Round-robin always advances to the next task, so once a finished task received the CPU, it never gave it back, freezing every other task, including the one waiting for both to finish.
- **Fix:** finished tasks keep calling `yield_now()`. The scheduler still has no notion of a finished or skippable task; that is a known gap.
- **Result:** the self-test completes (`task_a=10 task_b=10 switches=31`).

### Page fault copying the ring-3 program into its own code page

- **Symptom:** the first user-mode self-test faulted before reaching ring 3, on a supervisor write to the user code page.
- **Cause:** the code page was mapped without the writable bit on the assumption that ring 0 bypasses page permissions. That holds for the user/supervisor bit without SMAP, but not for the writable bit: `CR0.WP` blocks supervisor writes to read-only pages.
- **Fix:** map writable, copy, then remap without write before entering ring 3. Later superseded by writing program bytes through the frame's direct-map alias.
- **Result:** the program reaches ring 3 and completes on QEMU and VirtualBox.

---

## Security audit, 2026-08-09

Found by code review in multiple independent passes over the whole kernel tree, weighted toward memory safety, capability and privilege correctness, and interrupt safety. None of these had a runtime symptom; no test exercised the vulnerable paths. All were fixed in one commit and verified as non-regressing on QEMU (BIOS and UEFI) and VirtualBox. The new checks themselves are not exercised by any test, since no test attempts the attacks; the result is "no regression", not "attack shown blocked at runtime". The later typed boundary (below) moves the most serious class to compile time.

### Frame-Map: ring-3 privilege escalation

- **Cause:** three independent gaps on the Frame-Map path, any one of them sufficient. (1) No check that the target virtual address was outside the upper half whose page-table entries every address space shares with the kernel by reference, so a mapping there altered the kernel's own mappings. (2) The page-table walker did not check the huge-page bit before treating an entry as a pointer to a lower-level table; much of the direct map uses 2 MiB pages, so the walker could treat live memory as a page table and write an entry into it. (3) Neither Frame-Map nor Configure rejected the kernel's own AddressSpace as a target.
- **Fix:** all three closed independently: reject addresses in the shared range; reject the kernel's AddressSpace in both Frame-Map and Configure; assert in the walker that an entry treated as a table pointer is not a huge-page leaf.
- **Follow-up (2026-11-05):** the inline checks were replaced by the typed mapping boundary described in [architecture.md](architecture.md#frame-map): the user mapper accepts only types whose constructors are the checks, so a call site that omits a check does not compile. Two compile-fail test builds guard the boundary.

### Frame-Map: W^X bypass

- **Cause:** the writable and no-execute flags were independent, so a caller could request a writable, executable page.
- **Fix:** any writable mapping is forced no-execute regardless of the flags requested.

### Scheduler lock taken with interrupts enabled

- **Cause:** several paths took the non-reentrant scheduler spinlock without disabling interrupts. A timer tick inside the critical section would call `tick()`, which takes the same lock from inside an interrupt gate with interrupts off, and spin forever on a single core.
- **Fix:** save-and-restore of the interrupt flag around those critical sections. A plain `cli`/`sti` pair would have been wrong: the same functions run inside the syscall interrupt gate, where an unconditional `sti` would re-enable interrupts mid-handler.
- **Result:** no hang across VirtualBox boots with real ticks. This is a no-regression result; the exact race was never forced to occur.

### Configure committed before verifying

- **Cause:** Configure irreversibly consumed the caller's CSpace object before checking that the target thread was configurable. A failing call still destroyed the caller's object, against the kernel's verify-before-commit rule.
- **Fix:** check the thread's state before consuming anything; the later check next to the commit stays as the authoritative one.

### Retype ignored rights on the Untyped capability

- **Cause:** Retype checked the capability's type but not its rights, so narrowing an Untyped capability to no rights did not stop it being used to create objects.
- **Fix:** Retype requires the Write right on the Untyped capability, matching every other typed capability argument.

---

## Toolchain and environment

| Problem | Cause | Resolution |
|---|---|---|
| Custom JSON target spec required an unstable cargo flag | recent nightly cargo gates raw target specs | use the built-in `x86_64-unknown-none` target; move freestanding flags into `.cargo/config.toml` |
| `+soft-float` rejected as a target feature | it is a target-level property on this target | drop it; disabling SSE is sufficient |
| Kernel asserted on the Limine base revision at first boot | the Limine binary release was older than the protocol revision the crate requests | use the v11.x binary release |
| Limine reported no 64-bit support in VirtualBox | a 32-bit guest type masks the long-mode CPUID bit | use a 64-bit guest type with PAE/NX enabled |
| VirtualBox raw-file serial capture failed with `VERR_INVALID_NAME` | the target drive was not writable by the VirtualBox backend | write the log to a local drive |
| QEMU could not load OVMF via `-bios` | split 4 MiB CODE/VARS images need `-drive if=pflash` | two pflash drives, VARS copied to a scratch file |
| Hardware timer interrupts never reached the CPU under QEMU | specific to the development QEMU setup (see [validation.md](validation.md#the-qemu-interrupt-limitation)) | bounded self-test with a software-interrupt fallback; preemption verified in VirtualBox |

---

## Open findings

A full code audit on 2026-11-16 recorded 14 findings. All are open. They are listed by category only; each is tracked privately with its cause and a proposed fix.

| Category | Count |
|---|---|
| IPC liveness | 2 |
| Capability derivation and revocation | 2 |
| Syscall argument and caller validation | 5 |
| Resource bounds and cleanup | 2 |
| Fault containment | 1 |
| Interrupt discipline | 1 |
| Documentation drift | 1 |

One further open item predates the audit: fatal-fault reporting goes only to the serial port, so a wedged UART can hang the kernel before it halts.

Several of these are also prerequisites for the adaptive layer (bounded registries for the footprint bound, IPC that cannot block indefinitely for the human control path, a single capability installation path for the consent broker); see [adaptive.md](adaptive.md#what-the-kernel-must-provide-first).
