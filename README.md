# Rose

Rose is a capability-based microkernel for x86_64, written from scratch in Rust and booted by Limine. It is not derived from Unix, Linux, or NT. The kernel enforces capabilities and address-space isolation and owns CPU and memory allocation; drivers, services, and policy are meant to run as userspace components that hold capabilities. It is the planned replacement for the Linux kernel under [Meridian OS](https://github.com/TrashPanda2481/Meridian-OS-Overview), reached through a deliberately narrow Linux compatibility shim that is designed to shrink to zero. An adaptive layer (system observation plus consent-gated automation) is specified to run entirely as unprivileged components, with the kernel enforcing what that layer may do and what it may cost. That layer is designed, not built.

This repository is the project's public technical overview: what is built, what is specified, the defects diagnosed and resolved, and the plan. The source tree is private for now.

| | |
|---|---|
| Language | Rust, `no_std`, built-in `x86_64-unknown-none` target, nightly toolchain pinned to `nightly-2026-08-08`; hand-written assembly for exception stubs, the syscall gate, and the context switch |
| Size | about 7,800 lines in the kernel crate (including boot self-tests), 180 in the shared `abi` crate |
| Boot chain | Limine (v11.x binary release, vendored), Limine boot protocol, hybrid BIOS/UEFI ISO |
| Machine model | x86_64, single core, 8259 PIC + 8254 PIT at 100 Hz, COM1 serial plus a framebuffer text console |
| Test targets | QEMU BIOS (automated smoke test) and QEMU UEFI (OVMF); Oracle VM VirtualBox (legacy BIOS) as the manual gate for `stable`; no real hardware yet |
| Current phase | Phase 0 (kernel primitives) mostly done. Phase 1 (runtime component model) in progress: Untyped/Retype, Endpoint IPC, and root task handoff increment 1 are merged and on `stable`; the Loader is next |
| Adaptive layer | Specified, no code |
| Branches | `main` (active work) and `stable` (promoted only after QEMU and VirtualBox both pass); last promotion 2026-11-06 |

---

## Contents

1. [What Rose is](#1-what-rose-is)
2. [What is different from a plain capability microkernel](#2-what-is-different-from-a-plain-capability-microkernel)
3. [The kernel today](#3-the-kernel-today)
4. [Boot output](#4-boot-output)
5. [Where it runs](#5-where-it-runs)
6. [Engineering and troubleshooting](#6-engineering-and-troubleshooting)
7. [Documentation map](#7-documentation-map)
8. [Status](#8-status)

---

## 1. What Rose is

Rose is built in layers with separate obligations:

- **Kernel (reflex).** Mechanism only: capability and rights checks, address spaces, the scheduler, synchronous IPC, interrupt and fault handling. It decides what is permitted, not what should happen.
- **Root task.** The single privileged component the kernel creates at boot. It receives the initial authority (an Untyped capability over memory, a capability to its own AddressSpace, and a read-only BootInfo page) and creates every later component by retyping that memory. The kernel creates nothing after the handoff.
- **Components.** Drivers, servers, and applications as separate address spaces holding capabilities. Phase 1 (in progress) makes these creatable at runtime.
- **Adaptive layer (specified).** Sensing (Stem) and reasoning plus the user interface (Compass) as ordinary components, holding only capabilities the person granted, inside a footprint the kernel enforces. A build without them is still a complete system. How Compass on Meridian maps onto Rose is in the Meridian overview's [rose.md](https://github.com/TrashPanda2481/Meridian-OS-Overview/blob/main/docs/rose.md).

Design rules that the code and specs follow:

- Capabilities instead of Unix permissions. No ambient authority, no superuser. Every capability traces back to the boot handoff.
- Userspace is a component graph, not a process tree. There are no process, file, or socket kernel objects.
- A small syscall surface. Every syscall today is an operation on a capability the caller holds, plus one boot-reporting call used by the smoke test.
- No POSIX layer in the native model. Linux compatibility, when it exists, is one userspace personality server with a shrinking scope.
- Bespoke where a component defines the system's identity or exposes architectural surface (capability model, IPC, component graph, the adaptive substrate). Standard where hardware or protocol dictates the design (the boot chain uses Limine rather than a custom loader).

Full detail: [architecture.md](docs/architecture.md).

---

## 2. What is different from a plain capability microkernel

The kernel object model follows the seL4 line closely: Untyped memory retyped into typed kernel objects, a per-component capability table with a derivation tree for revocation, synchronous endpoints with one-shot reply capabilities. That part is not claimed as new. The differences are below, split by whether they exist in code.

**In the kernel today**

- **A typed user-mapping boundary.** User mappings go through three newtypes whose only constructors are the checks themselves: `UserVaddr` (4 KiB aligned, outside the kernel-shared half), `UserAddressSpace` (not the kernel's own), and `UserMapFlags` (user bit set, writable implies no-execute). The mapper reachable from syscalls accepts only these types, so a call site that skips a check does not compile. Two compile-fail test builds (one passes a raw address, one forges the type from a sibling module) must fail to build; a clean build means the boundary has a hole. This replaced the inline checks added after an audit found a privilege-escalation path on that syscall ([engineering-log.md](docs/engineering-log.md#frame-map-ring-3-privilege-escalation)).
- **W^X from the first kernel-owned page table.** The kernel image is mapped per section (text R+X, rodata R, data and bss R+W+NX); heap and direct-map pages carry NX. Any writable user mapping is forced no-execute regardless of the flags requested.
- **Verify before commit, as a written kernel rule.** Code that is about to commit state on information it has not checked performs a read-only check first. Thread configuration peeks the thread's state before claiming it; PCI enumeration reads a slot's vendor ID before touching any other register. The rule is used as an audit criterion, and audit findings are filed against it.
- **A two-virtualizer promotion gate.** QEMU runs on every build; VirtualBox, a different firmware and a hardware-virtualized CPU with real timer interrupts, gates every promotion to `stable`. Five defects so far appeared only under VirtualBox ([validation.md](docs/validation.md)).

**Specified, not built** ([adaptive.md](docs/adaptive.md))

- **An untrainable floor.** A fixed list of kernel behaviors that no policy, learned rule, or registered fast-path program may modify: capability checks, isolation and W^X, fault containment, the capability-minting path, revocation, the adaptive layer's own footprint cap, hardware-safety cutoffs, and a reserved minimum for the human control path.
- **One mint path for delegated authority.** A consent broker is the only component that mints capabilities for the adaptive layer, and only on a user's yes. Each automation is a capability in the derivation tree, so one revoke removes it and everything derived from it.
- **Consent modes as a consolidation gradient.** "Handle once" mints an ephemeral capability; "remember" stores a rule that re-mints on the same trigger; a rare, bounded subset may later be registered as a kernel fast-path rule through a verifier that holds the floor as its limit (an eBPF-shaped gate, scoped by capabilities).
- **A kernel-enforced footprint for the observer.** The adaptive layer runs from a bounded Untyped allocation, so the component watching for waste cannot exceed a cap that it cannot raise.
- **Linux compatibility confined to one component.** The transition plan puts all Linux-shaped code in a single personality server whose scope must shrink each phase ([roadmap.md](docs/roadmap.md)).

---

## 3. The kernel today

| Area | State | Notes |
|---|---|---|
| Boot, COM1 serial, framebuffer text console | stable | Limine protocol requests, base revision check |
| Physical frame allocator | stable | free list over 4 KiB frames, addressed through the higher-half direct map |
| GDT, IDT, TSS | stable | all 32 exception vectors, double fault on an IST stack |
| Kernel page tables, 1 MiB kernel heap | stable | W^X per section; first-fit heap backing `alloc` |
| PIC and PIT timer | stable | IRQ0 on vector 32 at 100 Hz |
| Scheduler | stable | one round-robin queue, 50 ms timeslice, per-task CR3 and kernel entry stack |
| AddressSpace | stable | one PML4 per component; kernel half shared, supervisor-only |
| Capabilities and CSpace | stable | 64-slot table, copy, mint, move, revoke with full cascade |
| Untyped and Retype | stable | Frame, CSpace, AddressSpace, Thread, Endpoint |
| Frame-Map, Thread Configure | stable | through the typed mapping boundary |
| Endpoint IPC | stable | Send, Receive, Call, Reply; register-only, no badges, no capability transfer yet |
| PCI enumeration | stable | legacy configuration mechanism #1, read-only |
| Root task handoff, increment 1 | stable | root program still embedded in the kernel image |
| Loader and boot archive | next | Phase 1 |
| Notification, IrqHandler, PageTable objects; Unmap; priorities | not built | |
| SMP, drivers outside the kernel, storage, networking | not started | Phase 2 and later |

Object model, syscall interface, and scope cuts: [architecture.md](docs/architecture.md).

---

## 4. Boot output

Rose has no graphical interface. This is a full boot of the current `stable` build under Oracle VM VirtualBox, from the serial log, abridged (the memory map and PCI listing are trimmed). Every boot runs the in-kernel self-tests before handing off to the root task; the last two lines come from the handoff.

```text
rose: boot ok
rose: base revision supported
rose: hello, hardware
rose: memory map: 17 entries
...
rose: usable memory: 522752 KiB
rose: hhdm offset=0xffff800000000000
rose: frame allocator: 130688 total frames, 130688 free
rose: gdt/idt loaded
rose: idt self-test: resumed after breakpoint, ok
rose: page tables loaded, kernel-owned
rose: address space self-test: isolation confirmed
rose: heap loaded, 1048568 bytes free
rose: pic remapped, irq0 unmasked
rose: timer self-test: 50 ticks elapsed via hardware irq0, 510ms since boot
rose: scheduler self-test: task_a=10 task_b=10 switches=31
rose: scheduled address space self-test: task_c read 0x1111111122222222 (match), task_d read 0x3333333344444444 (match), isolation across scheduling confirmed
rose: cspace self-test: move cleared source slot ok, copy/mint object_ref match, mint narrowed rights+badge ok, revoke cascade cleared moved+minted+root ok, overall confirmed
rose: cspace deep-revoke self-test: chain depth 63, deep enough ok, tail present pre-revoke ok, whole chain cleared ok, overall confirmed
rose: pci self-test: enumerating
...
rose: pci self-test: 10 device(s) found
rose: root task: handoff built, entering ring 3 at 0x0000000000400000
rose: root task: handoff ok (bootinfo read, untyped retype+map+rw confirmed)
```

`rose: hello, hardware` and `rose: root task: handoff ok` are the two lines the smoke test requires.

---

## 5. Where it runs

| Target | Firmware | Role | Result |
|---|---|---|---|
| QEMU | SeaBIOS (legacy BIOS) | automated smoke test on every build | passes; hardware IRQ0 is not delivered in the development QEMU setup, so preemption-dependent paths are confirmed elsewhere |
| QEMU | OVMF (UEFI) | manual run of the UEFI path | last run on the IPC Call/Reply increment (2026-08), passed; not re-run for the framebuffer console or root task increments |
| Oracle VM VirtualBox | legacy BIOS | manual gate for every promotion to `stable` | passes on the current `stable` |
| Real hardware | | | not yet |

Smoke test, VirtualBox VM settings, and the QEMU interrupt limitation: [validation.md](docs/validation.md).

---

## 6. Engineering and troubleshooting

[engineering-log.md](docs/engineering-log.md) is a reference of defects with symptom, diagnosis, cause, fix, and result. Representative entries:

| Symptom | Cause | Fix |
|---|---|---|
| Panic on the first real timer tick, VirtualBox only | scheduler initialized after interrupts were enabled; `tick()` divided by an empty task count | initialize the scheduler before the PIC; guard `tick()` against an empty queue |
| Ring-3 program faults mid-run under real preemption, VirtualBox only | a manual CR3 switch was not recorded in the task, so the scheduler restored the stale address space | one call that updates the task record and CR3 together, with interrupts off |
| A configured second task faults on its first instruction | its code and stack were mapped into a different address space from the one it ran in | add the Frame-Map syscall and map into the task's own address space |
| Fault on the instruction after a ring-3 syscall returns, two ring-3 tasks only | every task trapped through one shared kernel entry stack (`TSS.RSP0`) | per-task kernel entry stacks, reloaded on every switch |
| A ring-3 task could map memory over kernel text | three missing checks on the Frame-Map path (found by audit) | three independent checks, then a typed boundary that makes them compile-time |
| Double fault right after the switch to kernel-owned page tables | the bootloader's stack lived in memory the new tables did not map | map bootloader-reclaimable memory through the direct map |

---

## 7. Documentation map

| Document | Contents |
|---|---|
| [architecture.md](docs/architecture.md) | boot sequence, memory, capabilities and the derivation tree, Untyped/Retype, Frame-Map, threads, IPC, scheduler, syscall interface, root task handoff, the `abi` crate, verify before commit |
| [adaptive.md](docs/adaptive.md) | the specified adaptive layer: layers, untrainable floor, consent broker, learning gradient, reflex gate, what the kernel must provide first, related work |
| [engineering-log.md](docs/engineering-log.md) | fixed defects with symptom, diagnosis, cause, fix, result; toolchain and environment issues; open findings summary |
| [validation.md](docs/validation.md) | QEMU smoke test, VirtualBox promotion gate, the QEMU interrupt limitation, merge discipline, promotion history |
| [roadmap.md](docs/roadmap.md) | the transition plan off the Linux kernel, Phase 0 through 5, open design questions |
| [Meridian OS: rose.md](https://github.com/TrashPanda2481/Meridian-OS-Overview/blob/main/docs/rose.md) | in the Meridian overview: how Compass maps onto Rose, and a worked example of one adaptive decision |

---

## 8. Status

Phase 1 in progress.

**Confirmed working:** boot on QEMU (BIOS and UEFI) and VirtualBox; kernel-owned page tables with W^X; frame allocator and heap; hardware timer interrupts and preemptive round-robin scheduling across separate address spaces; capability tables with copy, mint, move, and cascading revoke; Untyped retype into five object types; Frame-Map; ring-3 threads; synchronous IPC including Call and Reply under real preemption; the root task handoff, where the root reads BootInfo, retypes a frame from its Untyped capability, maps it into its own address space, and writes and reads it back.

**Open:** a code audit on 2026-11-16 recorded 14 findings, all open, in IPC liveness, capability derivation, syscall argument and caller validation, resource bounds, fault containment, interrupt discipline, and documentation drift; they are summarized by category in [engineering-log.md](docs/engineering-log.md#open-findings). Fatal-fault reporting depends on the serial port. No real-hardware target yet. The adaptive layer has no code.

**Next:** the Loader (the root task loads and starts a component from a boot archive bundled at build time). Other follow-ups named in the specs: contiguous-region Untyped, IPC badges, capability transfer over IPC, and Notification objects. Detail in [roadmap.md](docs/roadmap.md).

---

*The boot log is from a VirtualBox run of the `stable` build; the PCI devices it lists are VirtualBox's emulated hardware.*
