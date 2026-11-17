# Adaptive layer

**Status: written design, no code.** Building it waits on the Phase 1 component model ([roadmap.md](roadmap.md)). The specification fixes where the adaptive layer sits, what authority it can hold, and the safety boundary around it. Each adaptive behavior is a closed loop with five parts (signal, state, decision, actuator, feedback). That decomposition is settled; the concrete definition of each part is not written yet, and the current directions below are proposals, not decisions.

The adaptive layer is the native successor to Compass, the guidance system on [Meridian OS](https://github.com/TrashPanda2481/Meridian-OS-Overview) today. How Compass's mechanisms on Linux map onto Rose is covered in the Meridian overview's [rose.md](https://github.com/TrashPanda2481/Meridian-OS-Overview/blob/main/docs/rose.md).

---

## Placement: no reasoning in the kernel

The kernel is reflex. It enforces capabilities and isolation and owns CPU and memory allocation; it decides what is permitted and leaves what should happen to the layers above it. The adaptive layer is a set of ordinary userspace components holding capabilities.

Reasons the reasoner stays out of the kernel:

- A reasoner in the kernel enlarges the trusted computing base by the size of the reasoner.
- The layer's resource consumption must be bounded by something it cannot influence. The kernel can enforce a bound on a component; it cannot enforce one on itself.
- Three roles stay separate: a component proposes, the kernel authorizes, the person grants.

The kernel's part is substrate, present in every build: a safe observation surface, a single path by which a human yes becomes authority, the footprint bound, and a gated facility for registering fast-path rules. The reasoning components (Stem, Compass, the learner) are a tenant. A build without them is a complete, working system.

---

## Layers

| Layer | Component | Role | Authority |
|---|---|---|---|
| Sensing | Stem | keeps baselines, detects deviation from them, pushes a report when something crosses a threshold; interprets nothing | scoped observe capabilities only |
| Reflex | kernel | enforcement; an untrainable floor plus a modulated tier that can be biased within bounds | non-optional |
| Modulation | resource manager | biases allocation inside ranges the kernel owns and can override; acts on its own within limits granted once | bounded, granted once |
| Intent | Compass and the person | interprets Stem reports, forms a likely cause, proposes an action, obtains consent | only what the consent broker mints |

The resource manager is its own component, separate from Compass, so the fast loop runs even in a build with no interface layer. Hard-safety responses stay autonomous because asking is too slow.

Two rules connect the layers:

- **Intent flows down, veto flows up.** Influence moves from the person toward the reflex layer; safety authority moves the other way. The untrainable floor overrides any intent, policy, or learned behavior.
- **The observer has a kernel-enforced footprint.** The cap on what Stem, Compass, and the learner may consume is enforced by the kernel and cannot be raised by the components it bounds.

---

## The untrainable floor

Kernel behaviors that stay pure reflex: consulted by nothing, modifiable by no one, never a target for consolidation from above. The floor is both the system's safety guarantee and the ceiling on anything the adaptive layer may install. Ratified 2026-10-17, before any code.

| # | Floor item | Kernel mechanism today |
|---|---|---|
| 1 | Capability and rights enforcement. Every invocation is checked against the capabilities held; no learned shortcut or fast path skips the check. | rights checks in syscall dispatch |
| 2 | Address-space isolation and W^X. | per-component page tables, forced NX on writable mappings |
| 3 | Fault containment. A fault terminates the offending component, not the system; policy never defers or suppresses a fault. | IDT fault path (today a fault halts; per-component containment needs the component model) |
| 4 | The authority-granting path. Only the consent broker, blessed by the root task, mints capabilities for the adaptive layer; the layer can never grant itself authority. | the capability install paths in Retype and IPC; not yet routed through one mediated path |
| 5 | Revocation always completes. The adaptive layer cannot block, slow, or train away revocation of its own authority. | CSpace derivation tree, full cascade |
| 6 | The footprint bound on the adaptive layer itself. | bounded Untyped (not yet enforceable, see below) |
| 7 | Hardware-safety cutoffs, such as an overtemperature halt, fire autonomously and cannot be suppressed. | not implemented; no hardware-safety sensing exists |
| 8 | The human control path is never starved. The path by which the person observes, consents, and revokes holds a guaranteed minimum of CPU and memory. It is the dual of item 6. | not implemented; needs scheduler reservations |

Off the floor (influenceable, and consolidatable under consent): allocation strategy (priority bias, which non-essential work to suspend, prefetch and cache hints, scheduling bias within safe ranges), what to surface and when, which suggestions to make, and preference adaptation.

---

## The consent broker

Authority enters the adaptive layer through one component. A human yes is the only thing that makes it mint a capability for Compass or the resource manager. Every interaction has the same shape: what was observed, the likely cause, the proposed action, what to expect if it runs, then a wait for an answer. For example: CPU temperature is above its baseline, a background program not tied to current work is the main load, suspending it should lower the temperature; proceed?

The answer selects one of three modes:

| Mode | Result |
|---|---|
| Handle once | an ephemeral capability for this action, gone afterward |
| Remember | a stored automation rule that re-mints the capability when the same trigger recurs |
| Never ask | a suppression entry |

Each automation is therefore a delegated, revocable capability, granted one situation at a time and removed with one cut of the derivation tree. Badges (not yet built) are meant to carry provenance on minted capabilities, so any delegated autonomy can be traced to the consent that created it.

The three modes presume a decided revocation semantics (full cascade or sealed capabilities, an open kernel question), so that decision is on the broker's critical path.

---

## Learning

**Training signal.** Only the person's answers: accept, decline, or do it differently. Nothing is sent off the machine and nothing is aggregated across users. A fresh install knows nothing about its user, asks more, and assumes less.

**Consolidation gradient.** A response moves down the layer stack as trust accrues, matching the three consent modes:

1. Manual: Compass asks every time.
2. Modulation: after approval and an explicit "remember", it runs on its own within bounds.
3. Reflex: in rare, tightly bounded cases, it is registered as a kernel fast-path rule through the reflex gate. Never into the floor.

**Grounding.** How the layer identifies what a component is, in trust order:

1. The capability and IPC graph. A component's purpose is legible from the capabilities it holds and the endpoints it talks to, which is hard to fake.
2. The component's declared manifest, treated as a claim and checked against the graph before it is believed.
3. Labels the person gives. Every consent dialogue is one labeled example.

**The learner.** Small, local, and interpretable: rules and thresholds plus a bounded case memory. Every suggestion traces to a rule or a past decision. No opaque model sits in the authority path.

---

## Decision loops

Each adaptive behavior is a closed loop with five parts. The decomposition is settled; the right-hand column is the current direction for each part and is proposed, not decided.

| Part | Question | Current direction (proposed) |
|---|---|---|
| Signal | what Stem emits | per-component counters and sensor readings; a running mean and variance as the baseline; a sustained threshold crossing as a deviation |
| State | the picture decisions are made against | the capability and endpoint graph, each node annotated with its baselines |
| Decision | whether a deviation matters and what to do | rules matching graph patterns plus deviations, and a case memory of past situations, actions, and verdicts (case-based reasoning) |
| Actuator | what the layer can do | a fixed, enumerated set: suspend, resume, priority bias, revoke, restart, cache hints; each is a kernel operation that requires a capability |
| Feedback | how a decision is judged | the user's verdict; whether a measured outcome may demote a remembered rule that fails to move its signal is open |

Of the proposed actuators, only revoke exists in the kernel today; thread suspend, resume, and priorities are not built. A worked example of one decision end to end (a sustained high CPU temperature, with each step that depends on unbuilt kernel or component work marked) is in the Meridian overview's [rose.md](https://github.com/TrashPanda2481/Meridian-OS-Overview/blob/main/docs/rose.md#example-one-decision-end-to-end).

---

## The reflex-registration gate

The kernel can acquire new fast-path behavior without changing its own code. It exposes a bounded, verified, capability-gated facility for registering rules of the form condition plus pre-authorized action. A registered rule acts without consultation, and the verifier holds floor items 1 through 8 as its hard limit. The shape is eBPF's (small programs installed from userspace, checked by a verifier, run in a kernel fast path), scoped by capabilities rather than by a global privilege. What a rule may express, and how the floor is proven as the verifier's limit, are open.

---

## Scope of the first version

System health only: is the machine behaving normally. That grounds cleanly in signals, baselines, and the capability graph. Inferring what the person is working on from indirect signals is an open problem and is deferred to a later, opt-in stage.

---

## What the kernel must provide first

Updated after the 2026-11-16 kernel code audit, which added items and moved one open question onto the critical path:

- **The component model** (Phase 1). There is nothing to observe or act through until components can be created at runtime.
- **An observation surface.** Per-component accounting in the kernel: event and object counters on Retype, Frame-Map, and the IPC path. Cheap to add while those paths are being built, expensive to retrofit.
- **An observe right** in the kernel's `Rights` set, decided when the root task first hands out capabilities. The root task is the only authority that could grant system-wide observation while withholding system-wide power.
- **The consent broker as a kernel-backed single mint path.** One mediated path, with provenance recorded at mint time, that every capability installation routes through. Capability installation is not yet routed through one path today.
- **A decision on revocation semantics** (full cascade or sealed capabilities), since all three consent modes presume one.
- **Notification objects**, for Stem's push path. The observe right governs what Stem may read; notifications are how it reports without polling.
- **Cross-CSpace capability transfer over IPC.** The broker delegates by moving a capability into another component's table, and Move is same-CSpace only in v0.1.
- **IPC badges**, to carry provenance on minted capabilities.
- **A bounded Untyped for the adaptive layer, and bounded kernel registries** to enforce it against. Registries are not yet bounded and nothing is reclaimed, so no footprint claim under floor item 6 is enforceable yet.
- **Scheduler reservations** for floor items 7 and 8, and with them the IPC half of item 8: a yes must always arrive. The audit found IPC paths that can block indefinitely ([engineering-log.md](engineering-log.md#open-findings)).
- **The reflex-registration gate**, later.

---

## Decisions settled (2026-10-17)

- The untrainable floor, with the human-control-path minimum added as item 8.
- Grounding is structural first: the capability graph leads; manifests and labels enrich it.
- The learner is interpretable only: rules plus a bounded case memory, no opaque model in the authority path.
- Resource-allocation policy is its own resource-manager component, separate from Compass.
- The fast loop acts on its own within bounds granted once.

## Open

- The encoding of the observe capability or right in the kernel's `Rights` set.
- Revocation semantics: whether a revoke always cascades down the derivation tree or capabilities can be sealed. The consent broker's three modes depend on the answer.
- The manifest schema and how a manifest is checked against the capability graph.
- The case memory's size bound and eviction policy.
- The reflex gate's verifier: what a fast-path rule may express, and how the floor is proven as its limit.
- The concrete loop definitions above, and whether measured outcomes may demote a remembered rule.

---

## Related work

None of the individual parts is new.

| Part | Prior art |
|---|---|
| Monitor, analyze, plan, execute loop | autonomic computing and the MAPE-K loop [1, 2] |
| Capability microkernel | KeyKOS [3], EROS [4], seL4 [5], Fuchsia's Zircon [6] |
| Verified rules installed into a kernel fast path | eBPF; sched_ext, merged in Linux 6.12, which runs scheduling policy as BPF programs [7] |
| Machine learning inside a kernel | LinnOS, a small neural network in the block layer predicting per-I/O flash latency [8]; LAKE, ML-driven kernel subsystems with accelerator access from kernel space [9] |
| Usage-driven resource management | Android Adaptive Battery, Windows SysMain, macOS App Nap, systemd-oomd with pressure stall information (PSI) |
| Rules plus case memory | expert systems and case-based reasoning |

Rose's kernel object model follows the capability-kernel line directly; the cascade-versus-sealed revocation question is the seL4 and EROS split. The reflex gate has the same shape as eBPF. LinnOS and LAKE place a model in the kernel's decision path; the Rose specification keeps any learned component out of the authority path and out of the kernel.

1. J. O. Kephart and D. M. Chess, "The Vision of Autonomic Computing," IEEE Computer 36(1), 2003.
2. IBM, "An Architectural Blueprint for Autonomic Computing."
3. N. Hardy, "KeyKOS Architecture," ACM SIGOPS Operating Systems Review 19(4), 1985.
4. J. S. Shapiro, J. M. Smith, D. J. Farber, "EROS: A Fast Capability System," SOSP 1999.
5. G. Klein et al., "seL4: Formal Verification of an OS Kernel," SOSP 2009. [paper](https://trustworthy.systems/publications/papers/Klein_EHACDEEKNSTW_09.abstract)
6. Zircon handles and rights. [rights](https://fuchsia.dev/docs/concepts/kernel/rights)
7. sched_ext initial pull request for Linux 6.12. [pull request](https://lkml.iu.edu/hypermail/linux/kernel/2409.1/07472.html)
8. M. Hao et al., "LinnOS: Predictability on Unpredictable Flash Storage with a Light Neural Network," OSDI 2020. [paper](https://www.usenix.org/conference/osdi20/presentation/hao)
9. H. Fingler et al., "Towards a Machine Learning-Assisted Kernel with LAKE," ASPLOS 2023. [doi](https://doi.org/10.1145/3575693.3575697)

The claimed difference is the combination: the adaptive layer starts with no authority; each piece of autonomy is a revocable capability minted from one user consent through a single broker; and the kernel enforces both that authority and the layer's resource footprint, under a floor nothing above it can modify. This is a design claim. None of it is built.
