# Roadmap

The path from "Rose boots under a Debian-based Meridian" to "Rose is Meridian's kernel and nothing Linux-derived is load-bearing". Dates are not committed. Current state per feature is in [architecture.md](architecture.md) and [validation.md](validation.md); this document tracks the plan's shape.

---

## Decision: a hybrid shim

Three readings of "replace the Debian kernel" were considered:

| Option | Outcome |
|---|---|
| Full Linux syscall compatibility (fork/execve, signals, mmap, VFS, cgroups, namespaces, abstract Unix sockets) | Rejected. It amounts to reimplementing most of Linux's kernel interface, and matching Linux semantics that closely makes the system Linux-shaped. |
| Cold native replacement: nothing from Debian's userland runs unmodified from the start | Rejected as a starting point. It is the right end state, but nothing is usable until the whole native stack exists, and there is no incremental proof along the way. |
| Hybrid: a minimal, deliberately narrow compatibility shim to bootstrap the transition, with everything meant to be permanent written native from the start | Chosen. The shim exists to shrink, not to grow toward parity. |

---

## Phases

### Phase 0: kernel primitives

**Mostly done.** Frame allocator, GDT/IDT/TSS, paging, heap, timer, scheduler, address spaces, capabilities and CSpace, CSpace syscalls. Primitives named in the v0.1 specification that are not built yet include Notification and IrqHandler objects, Unmap, thread priorities and SetPriority, task exit, AddressSpace teardown, and frame reclamation on revoke.

### Phase 1: runtime component model

**In progress.** Nothing past this point works until components can be created at runtime and talk to each other.

| Step | State |
|---|---|
| Untyped/Retype: turn raw memory into typed kernel objects at runtime | done, on `stable` |
| Endpoint IPC: register-only synchronous messaging (Send, Receive, Call, Reply) | done, on `stable`; no badges or capability transfer yet |
| Root task, increment 1: the kernel creates one privileged root task, hands it an Untyped capability and BootInfo, and creates nothing after | done, on `stable`; the root program is still embedded in the kernel image |
| Loader: the root task parses and maps a binary (Rose's own format, not necessarily ELF) and starts its thread, from a boot archive bundled at build time; there is no filesystem yet | next |

Follow-ups named in the specifications: contiguous-region Untyped (root task increment 1b), IPC badges (IPC increment 3), cross-CSpace capability transfer over IPC, and the root task creating a child component and talking to it over IPC.

### Phase 2: minimal native runtime

- A Rose-native runtime crate (extending `abi`, or a new crate): syscall wrappers, no libc, no POSIX assumptions.
- The console and serial driver move out of the kernel into a userspace component holding IrqHandler and Frame capabilities. This is the first end-to-end proof that driver isolation works.
- A storage driver (virtio-blk under QEMU and VirtualBox) and a minimal native filesystem server, so native components can load from disk rather than only from the boot archive.

### Phase 3: the shim

One userspace component: a personality server translating a narrow slice of Linux syscalls into calls against the native filesystem, loader, and IPC primitives from Phases 1 and 2.

Scope: enough for a shell, coreutils, and a build toolchain. Out of scope unless a concrete transition step needs it: cgroups, namespaces, abstract Unix sockets, the deeper systemd and D-Bus surface.

This is the only place Linux-shaped code exists in Rose. Growth past what the next phase needs is treated as drift back toward full compatibility, not progress.

### Phase 4: replace Meridian's stack outward

Lowest boot dependency first: init and service supervisor, then shell, then display, then applications.

Rule: each time a native replacement lands for something the shim covered, that part of the shim is deleted, not kept "just in case". The shim's line count should fall every phase.

### Phase 5: shim at zero

A fully native Meridian userland on Rose, with no Linux compatibility code. Interoperability with other systems (common filesystem formats, common network protocols, possibly a foreign-binary translation layer) can exist afterward as an optional add-on, not as something the system depends on.

---

## The adaptive layer

The adaptive layer cannot be built before the component model: it needs components to observe and components to act through. Its kernel prerequisites (an observation surface, an observe right, a single consent-gated mint path, Notification objects, capability transfer, badges, bounded registries, scheduler reservations) are listed in [adaptive.md](adaptive.md#what-the-kernel-must-provide-first). The per-component accounting hooks in particular are cheap to add while the Phase 1 paths are being built and expensive to retrofit afterward.

---

## Open design questions

1. Revocation: full cascade (current) or capabilities that can be sealed to survive a parent's revocation. On the consent broker's critical path.
2. IPC message limits: data words and capabilities per message (defaults 4 and 2).
3. Paging policy: one pager per component or one shared root pager.
4. Address-space layout: whether to move from the shared supervisor-only upper half to a KPTI-style or seL4-style layout in which kernel memory is absent from component page tables.

## Heuristic for later components

Build original where the component defines Rose's identity or exposes architectural surface. Conform to the standard where hardware or protocol dictates the design. Drivers, storage, networking, and graphics default to standard-conformant unless there is a specific reason Rose needs to differ.
