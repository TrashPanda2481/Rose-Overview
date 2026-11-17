# Architecture

The kernel as built at the root task handoff (Phase 1, increment 1), plus the parts of the v0.1 specification that are not yet implemented, marked as such. Scope for v0.1: capabilities, CSpace, register-only IPC, address spaces, boot handoff, scheduler. Drivers, storage, networking, graphics, and SMP are later cores.

---

## Boot sequence

1. Limine loads the kernel (built for the stock `x86_64-unknown-none` target, with SSE disabled, static relocation, kernel code model, and a linker script) and answers the kernel's protocol requests: memory map, higher-half direct map (HHDM) offset, kernel physical and virtual base, framebuffer. The kernel checks the protocol base revision before anything else.
2. `kernel_main` initializes COM1 and prints `rose: hello, hardware`. COM1 also backs the panic handler.
3. Physical memory: the memory map is parsed and logged, and a free-list frame allocator is built over usable regions.
4. GDT, IDT, and TSS replace Limine's. All 32 exception vectors have hand-written assembly stubs (save registers, call a common Rust handler with a known frame, restore, `iretq`); the `x86-interrupt` ABI is not used. Double fault runs on an IST stack.
5. Kernel-owned page tables replace the bootloader's (see Memory). The framebuffer is mapped and a text console starts mirroring serial output.
6. Kernel heap.
7. Scheduler initialization (task 0 registered), then PIC remap and PIT start. The scheduler is initialized before interrupts are enabled; the reverse order panicked on the first real timer tick (see the engineering log).
8. Self-tests: timer, scheduler, scheduled address-space isolation, CSpace, CSpace deep revoke, PCI enumeration. These run on every boot and print greppable lines.
9. Root task handoff. `kernel_main` then idles (`yield_now` and `hlt` in a loop) and creates nothing further.

A build-time feature replaces step 9 with the older hand-assembled ring-3 self-test, which exercises CSpace, Untyped, Frame-Map, Configure, and IPC error paths the root program does not. It is kept as a regression artifact and is off by default.

---

## Memory

**Frame allocator.** A free list of 4 KiB frames built from the usable regions of the memory map. Each free frame stores the next-free pointer in its first 8 bytes, accessed through the HHDM, so there is no separate bookkeeping array. Protected by a small spinlock.

**Kernel page tables.** Four-level x86_64 tables owned by the kernel. The kernel image is mapped per section from linker-provided bounds: text R+X, rodata R, data and bss R+W+NX. The HHDM is mapped for usable, bootloader-reclaimable, and framebuffer memory, with NX. `EFER.NXE` is enabled before the first kernel-owned table is loaded, so a virtualizer without PAE/NX cannot run Rose.

**Heap.** A fixed 1 MiB region, eagerly mapped, with a first-fit free-list allocator backing `extern crate alloc`. No coalescing in v0.1.

**AddressSpace.** A handle to a PML4. `new_address_space()` allocates a fresh PML4 whose upper half (entries 256 to 511: HHDM and kernel image) is shared with the kernel's tables and marked supervisor-only, and whose lower half is empty. Ring 3 cannot read or write the shared half. This is the conventional pre-KPTI layout, not the stricter goal in the specification that kernel memory be structurally absent from component page tables; it is a known v0.1 limitation to revisit if Meltdown-class concerns become relevant. There is no AddressSpace teardown yet.

**Page faults (specified, not built).** The kernel will not resolve faults; it will convert a fault into an IPC to a pager capability that the component configured, and the pager decides whether to map, kill, or otherwise respond. Today a fault in ring 3 is logged and halts. The open question is one pager per component versus one shared root pager; the default is a shared root pager until the component model is real.

---

## Capabilities

A capability is an unforgeable, kernel-mediated reference to an object plus rights. Userspace never sees raw pointers or object IDs, only slot indices into its own table.

```text
Capability {
    object_ref:  KernelObjectId
    object_type: Untyped | Endpoint | Notification | Thread | AddressSpace |
                 PageTable | Frame | IrqHandler | Reply | CSpace
    rights:      Read | Write | Grant | Map | Send | Receive   (bitfield)
    badge:       u64 (sender-set tag; not yet used by IPC)
}
```

`KernelObjectId` reuses an identity the object already has where one exists: a Frame's physical address, an AddressSpace's PML4 physical address. CSpace, Thread, Endpoint, and Reply objects are indices into per-type kernel registries.

**Object types**

| Type | Represents | Operations | Built |
|---|---|---|---|
| Untyped | raw physical memory | Retype | yes |
| Frame | one physical page | Map, Unmap | Map only |
| AddressSpace | a component's virtual memory root | install Frame | yes |
| Thread | schedulable unit | Configure, Resume, Suspend, SetPriority | Configure only |
| CSpace | a component's capability table | Insert, Delete | yes |
| Endpoint | synchronous IPC rendezvous | Send, Receive, Call | yes |
| Reply | one-shot reply capability from a Call | Reply | yes |
| Notification | binary asynchronous signal | Signal, Wait, Poll | no |
| IrqHandler | binds an IRQ line to a Notification | Ack, Bind | no |
| PageTable | one page-table level | install into AddressSpace | deliberately deferred |

There is no process, file, or socket object. PageTable is deferred on purpose: `map_page` manages intermediate levels internally, and nothing yet needs a table level as a nameable object.

**CSpace.** Each component has one capability table, addressed by local integers (`CPtr`). It is a fixed 64-slot table; slot 0 is the null capability. A component cannot name a slot in another component's table. Operations:

- Copy: new slot, same object, equal or lower rights.
- Mint: new slot, new badge and/or lower rights.
- Move: transfer out of the source slot. Same-CSpace only in v0.1; cross-CSpace transfer needs IPC capability transfer, which is not built.
- Revoke: destroy a capability and everything derived from it.

**Derivation tree.** Copy, mint, and retype record the new capability as a child of its source. Revoke walks the tree and clears every descendant (full cascade, seL4 style). A boot self-test builds a 63-deep derivation chain and confirms one revoke clears all of it. Whether a capability may instead be sealed to survive its parent's revocation (EROS style, for persistent references) is an open question; full cascade is the v0.1 default.

---

## Untyped and Retype

An Untyped object is a pool of physical frames. `retype` takes the next frame from the pool and produces one object: Frame, CSpace, AddressSpace, Thread, or Endpoint. The new capability is installed as a child of the Untyped capability, so revoking the Untyped cascades into everything retyped from it.

The syscall checks everything that can fail before calling `retype` (both slots resolve, the source is Untyped with the Write right, the type is retypeable, the destination slot is empty), because a frame taken from the pool has no path back if the install is then rejected.

Default rights on new capabilities: Frame gets Read, Write, Map; AddressSpace gets Map; CSpace and Thread get Grant (provisional until per-operation thread rights are defined).

Scope cuts in v0.1:

- The pool is frames pulled one at a time from the allocator, not one contiguous physical region. Contiguous-region Untyped is a follow-up to the root task handoff.
- Revoke destroys capabilities but does not return frames to the allocator. There is no reclamation.
- CSpace and Thread structures live on the kernel heap or in a registry, not in the frame charged to them; the frame is bookkeeping. An AddressSpace charges one bookkeeping frame plus the PML4 frame allocated directly.

---

## Frame-Map

`SYS_FRAME_MAP` maps a Frame into an AddressSpace at a caller-chosen virtual address. It requires Map on the Frame (and Write if a writable mapping is requested) and Map on the AddressSpace. Flags are an abstract two-bit field (writable, no-execute), not raw PTE bits.

The checks are enforced by types rather than by inline guards. `paging` exposes:

- `UserVaddr`: 4 KiB aligned and outside the kernel-shared half. Its field is private to `paging`; the checked parser is the only constructor.
- `UserAddressSpace`: an AddressSpace proven not to be the kernel's own.
- `UserMapFlags`: user bit set; any writable mapping is also no-execute.

`map_user_page` accepts only these types. The raw `map_page` remains as the kernel-internal primitive. A raw integer address, the kernel's address space, or a writable-and-executable flag set cannot reach the user mapper because it does not type-check. Two compile-fail builds guard this: one passes a raw `u64` where a `UserVaddr` is required (must fail with a type mismatch), and one constructs `UserVaddr` directly from another module (must fail on the private field). Either building cleanly is a regression. As a further structural check, the page-table walker asserts that an entry it is about to treat as a pointer to a lower table is not a huge-page leaf.

Unmap is not implemented.

---

## Threads

A retyped Thread is inert until `SYS_THREAD_CONFIGURE` binds it to an AddressSpace, a CSpace, an entry point, and a stack top, and places it on the run queue in ring 3. Configure follows verify before commit: it reads the thread's state without changing it (`peek_thread_unconfigured`), and only then consumes the caller's CSpace object and claims the thread. Resume, Suspend, and SetPriority do not exist yet; there is no task exit.

---

## IPC

**Specified message (v0.1).** Register-only, no shared buffers or memory grants:

```text
Message {
    label:     u32   (opcode, interpreted by the receiver)
    length:    u8    (0..MAX_MSG_WORDS)
    cap_count: u8    (0..MAX_MSG_CAPS)
    data:      [u64; MAX_MSG_WORDS]
    caps:      [CPtr; MAX_MSG_CAPS]
}
```

The default limits are 4 words and 2 capabilities, small on purpose so bulk transfer goes through a real grant mechanism later. The exact values are an open question.

**Built.** An Endpoint object with four operations, each blocking:

- Send: blocks until a receiver takes the message.
- Receive: blocks until a Send or Call arrives.
- Call: Send plus a wait on a kernel-created one-shot Reply object.
- Reply: answers one Call through the Reply capability, which the kernel installs in the receiver's CSpace when the delivered message came from a Call.

The implemented message is a subset of the specified one: a label and two data words. Badges (which minted capability the sender used) and capability transfer are not built. Interrupts and asynchronous events are specified to use Notification objects (Signal, Wait), with no separate interrupt-handler concept; Notification is not built.

---

## Scheduler

v0.1 as built: one round-robin run queue, a fixed 5-tick timeslice (50 ms at 100 Hz), preemption from the PIT interrupt and voluntary `yield_now`, and a context switch written as naked assembly (save callee-saved registers, swap stack pointers, `ret`). On every switch-in the scheduler reloads CR3 for the incoming task's AddressSpace and `TSS.RSP0` for its private kernel entry stack. The scheduler lock is taken with interrupts saved and disabled, then restored, because the same paths run both in normal context and inside the syscall interrupt gate.

Specified but not built: priorities 0 to 15 set at Thread configuration and changeable by the holder of the Thread capability, one timeslice per priority level, and no starvation handling (a saturated high-priority thread can starve lower ones; a known gap, deferred behind IPC correctness). Floor item 8 in the adaptive specification (a reserved minimum for the human control path) depends on scheduler reservations that do not exist yet.

---

## Syscall interface

Entry is `int 0x80` through a DPL 3 interrupt gate. The syscall number is in `rax`, arguments in `rdi`, `rsi`, `rdx`, `r10` (`r10` rather than `rcx`, so the convention carries over to a later `syscall`/`sysret` entry). The result is returned in `rax`: zero or positive is success, negative is an error code whose magnitude identifies the error, with a separate offset per syscall family so codes do not collide.

| Number | Syscall |
|---|---|
| 0x1000 to 0x1003 | CSpace Copy, Mint, Move, Revoke |
| 0x1004 | Untyped Retype |
| 0x1005 | Thread Configure |
| 0x1006 | Frame Map |
| 0x1007 to 0x100A | Endpoint Send, Receive, Call, Reply |
| 0x2000 | Root report (boot verification only) |

Every call in the 0x1000 range operates on capabilities in the caller's own CSpace. The 0x2000 call carries no capability operation: the root program passes three values that the kernel checks and logs for the smoke test.

---

## Root task handoff (Phase 1, increment 1)

The kernel builds one root task and starts it:

| Resource | Mapping or rights |
|---|---|
| Fresh AddressSpace | code page R+X+U at 0x400000; stack R+W+U+NX at 0x600000; BootInfo R+U+NX at 0x800000 |
| Root CSpace | holds exactly two capabilities |
| Untyped capability | a bounded pool of 1024 frames; Read, Write, Map |
| AddressSpace capability | the root's own; Map only |
| BootInfo page | plain data, not a capability |

Each right is the minimum the root program needs: Write on the Untyped because Retype requires it, Map on its own AddressSpace so it can map what it retypes, nothing else.

`BootInfo` is a `repr(C)` structure in the `abi` crate with a fixed layout read by offset: a magic value, usable memory in bytes, the HHDM offset, and framebuffer address, width, height, pitch, and bits per pixel. The raw memory map is deferred until the Loader needs it.

The root program is still embedded in the kernel image. It reads the BootInfo magic, retypes one Frame from its Untyped capability, maps it into its own address space writable and no-execute, writes and reads back a 64-bit pattern, and reports. The kernel verifies the magic and the readback and prints `rose: root task: handoff ok`. Any failure short of that faults, so the line prints only on the fully working path.

Deferred from this increment, each to a named later step: loading an external binary from a boot archive (the Loader), contiguous-region Untyped, a real console device capability (device capabilities are Phase 2), reclamation, and the root creating a child component and talking to it over IPC.

The specified invariant: the handoff is the only place authority comes from nothing. Every other capability in the system must trace back to it.

---

## The `abi` crate

One `no_std`, dependency-free crate shared by the kernel and future userspace: `ObjectType`, `Rights`, `Capability`, `CPtr`, `KernelObjectId`, and `BootInfo`. Object types, syscall numbers, and message labels are defined once and never duplicated by hand on both sides of the boundary. Breaking changes to the model bump the specification version.

---

## Verify before commit

A standing rule for kernel code: nothing acts on unconfirmed information. Wherever the kernel is about to commit state on something it has not checked, a read-only verification comes first, and the commit proceeds only if it passes. Unverified data is treated as untrusted, not assumed correct.

Current instances: Configure peeks the thread's state before claiming it; Retype validates every argument before taking a frame; PCI enumeration reads a slot's vendor ID before reading any other register. The rule is applied in audits: a 2026-08 finding (Configure consumed a caller's CSpace object before checking the thread) and a 2026-11 finding (a CSpace operation takes the source capability before finishing validation) were both filed as violations of it. The adaptive specification applies the same rule to component identity: a component's declared manifest is a claim, checked against its actual capabilities before it is believed.

---

## Open design questions

1. Revocation: full cascade (current) or sealed capabilities that survive a parent's revocation. This now blocks the consent broker design ([adaptive.md](adaptive.md)).
2. Message limits: `MAX_MSG_WORDS` and `MAX_MSG_CAPS`.
3. Pager: per component or one shared root pager.
