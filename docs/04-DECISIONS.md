# 04 — Decisions (ADR Log)

Format: Context → Decision → Alternatives considered → Consequences.
Status is one of `Decided`, `Open`, or `Superseded`.

---

### ADR-0001 — Capability-based microkernel over hybrid or monolithic
**Status:** Decided

**Context:** Kernel design paradigm is the single largest lever on both
tech debt and performance ceiling for this project.

**Decision:** Capability-based microkernel, seL4-lineage: minimal kernel
providing IPC/scheduling/MMU-control mechanism only; all policy (drivers,
filesystems, network stack) in userspace.

**Alternatives considered:**
- *Monolithic modular (Linux-style)* — highest raw syscall performance
  (direct function calls, no IPC hop for driver access), but a single
  faulting driver can take the whole kernel down, and the TCB is
  effectively "everything," which works against the "eliminate bug
  classes" directive.
- *Hybrid (XNU/NT-style)* — a reasonable middle ground, but it keeps
  drivers privileged for performance reasons, which reintroduces exactly
  the "driver crash is a kernel crash" bug class this project wants gone.

**Consequences:** IPC cost becomes a real, measured design constraint
(mitigated by the fast-path design in `03-SYSCALL-ABI.md`), and driver
performance depends on the shared-ring async layer for I/O-heavy cases
rather than direct in-kernel calls. In exchange, driver isolation,
minimal TCB, and capability-mediated authority come for free from the
architecture rather than needing separate enforcement.

---

### ADR-0002 — Implementation language: C
**Status:** Decided

**Context:** Rust would provide compile-time memory/data-race safety;
C provides maximum low-level control with more manual discipline required.

**Decision:** C, freestanding, no libc in the kernel.

**Alternatives considered:** Rust (rejected for this project, not
rejected in general — it's a legitimate, arguably safer default for new
kernels); Zig; a mixed Rust/C split.

**Consequences (the honest one):** the kernel itself does **not** get
compile-time memory or data-race safety. This is a real cost, not a
rounding error, and it means the mitigations in `00-PHILOSOPHY.md` —
minimal TCB, no kernel heap, static analysis as a build gate, host-testable
core logic, eventual formal verification of critical invariants — are
load-bearing, not optional polish. seL4 shows this combination is provable
safe in principle (via Isabelle/HOL proof, not via the language), but that
proof effort is a multi-year undertaking historically done by a dedicated
research team — treat "formally verify everything" as an aspirational
stretch goal, not an assumed safety net for this project's timeline.

---

### ADR-0003 — Target hardware: x86-64 only, UEFI boot
**Status:** Decided

**Context:** Multi-architecture support (ARM64, RISC-V) broadens hardware
coverage but forks every low-level doc from day one.

**Decision:** x86-64 only. UEFI firmware only, no legacy BIOS/CSM.

**Alternatives considered:** x86-64 + ARM64 (e.g. Raspberry Pi target);
adding RISC-V for its cleaner/more orthogonal architecture at the cost of
firmware/tooling immaturity.

**Consequences:** No hardware-abstraction-layer seam is built prematurely
(per the "no HAL until a second real target exists" rule in
`00-PHILOSOPHY.md`) — every doc can speak in concrete x86-64 terms
(specific MSRs, specific instructions) instead of hedged generic language,
which is exactly the level of detail this project wants. Revisit only if
a second real hardware target is actually acquired.

---

### ADR-0004 — x2APIC only; no persistent xAPIC/PIC support
**Status:** Decided

**Context:** xAPIC (MMIO-based, 8-bit APIC ID, present on essentially all
x86-64 hardware) vs. x2APIC (MSR-based, no MMIO serialization stalls,
32-bit APIC ID, present on hardware from roughly the last 15 years).

**Decision:** Detect x2APIC via `CPUID.01H:ECX[21]` at boot and require it.
Legacy 8259 PICs are defensively masked at boot (see `02-BOOT.md`) but are
never a supported runtime interrupt path. No persistent xAPIC fallback
code path is maintained.

**Alternatives considered:** Support both, falling back to xAPIC on older
hardware — rejected as exactly the kind of "maintain two paths
indefinitely" pattern `00-PHILOSOPHY.md` argues against; the hardware
population that lacks x2APIC is both old and out of scope for a project
explicitly targeting "best performance."

**Consequences:** Genuinely ancient hardware (pre-~2012) cannot boot this
OS. Every core-count/IPI/timer design downstream can assume MSR-based APIC
access without a compatibility branch.

---

### ADR-0005 — SYSCALL/SYSRET as sole kernel entry; async I/O as a separate, userspace-visible ring layer
**Status:** Decided

**Context:** Initial framing conflated "SYSCALL/SYSRET vs. io_uring" as if
they were competing choices; they operate at different layers (kernel
entry mechanism vs. async I/O batching model).

**Decision:** SYSCALL/SYSRET is the only kernel entry mechanism (no
`INT 0x80`, no `SYSENTER`/`SYSEXIT` path). A shared submission/completion
ring, `io_uring`-style, is layered on top for I/O-heavy clients, using
`Notification` objects for signaling and SYSCALL only for ring setup and
optional block-waits — never per-operation in the steady state.

**Alternatives considered:** N/A — this ADR exists to record the
correction itself, so the wrong framing doesn't quietly resurface later.

**Consequences:** Full design detail in `03-SYSCALL-ABI.md`.

---

### ADR-0006 — No kernel heap; static/untyped-object memory model
**Status:** Decided

**Context:** A conventional `kmalloc`-style kernel heap is the path of
least resistance but reopens "unbounded kernel allocation" and "kernel
OOM" as live bug categories.

**Decision:** All kernel objects are created via `Retype` on `Untyped`
capability objects, per `01-ARCHITECTURE.md`. No dynamic kernel heap exists.

**Alternatives considered:** A bounded/pooled kernel heap with hard caps —
rejected as still requiring a policy decision (the caps) that's better
made explicitly in userspace via untyped-region grants than implicitly
via a global kernel constant.

**Consequences:** All memory allocation *policy* moves to the root task
and downstream memory-manager process. Kernel accounting becomes "how much
Untyped capacity has been retyped," a much simpler invariant to reason
about than general heap fragmentation/exhaustion.

---

### ADR-0007 — No BIOS/CSM, no 32-bit userspace, no 16-bit paths outside the AP trampoline
**Status:** Decided

**Context:** Supporting legacy boot and 32-bit compatibility mode
broadens compatibility but each is a permanent second code path.

**Decision:** UEFI-only boot; no 32-bit compatibility-mode userspace
(`IA32_CSTAR` left unprogrammed); the only 16-bit real-mode code in the
system is the transient AP trampoline described in `02-BOOT.md`, which
exists only because the x86-64 architecture itself requires it for SMP
bring-up, not by choice.

**Alternatives considered:** Supporting 32-bit userspace for
compatibility with existing toolchains/binaries — not applicable here
since there's no existing binary ecosystem to be compatible with; this
is a from-scratch OS.

**Consequences:** Simpler GDT/TSS/syscall-ABI story throughout, at the
(irrelevant, for a from-scratch OS) cost of not running any existing
32-bit binaries.

---

### ADR-0008 — Bootloader: Limine, not hand-rolled
**Status:** Decided

**Context:** A hand-rolled UEFI bootloader is more educational and more
"yours," but re-derives a lot of well-trodden GOP/ACPI/memory-map/
`ExitBootServices` handling that's easy to get subtly wrong on unfamiliar
real hardware.

**Decision:** Use the Limine boot protocol. It already provides exactly
the boot-info structure `02-BOOT.md` describes (memory map, framebuffer,
RSDP pointer, higher-half kernel mapping), is purpose-built for hobby/
research kernels, and has been exercised across far more real UEFI
implementations than a from-scratch bootloader would be before shipping.

**Alternatives considered:** Hand-rolled bootloader — rejected per
`00-PHILOSOPHY.md`'s "boring technology for anything not the point of the
exercise": firmware-quirk-wrangling teaches nothing that transfers to the
capability-kernel work that's the actual point of this project, and is
exactly the kind of debugging most likely to stall a hobby project before
the interesting part starts.

**Consequences:** Unblocks `docs/06-ROADMAP.md` milestone M1. Kernel entry
point and boot-info parsing should target the Limine boot protocol
directly; see `docs/11-GETTING-STARTED.md` for toolchain setup.

---

### ADR-0009 — Network identity: design for unlinkability, not undetectability
**Status:** Decided

**Context:** Initial framing ("randomize the IP so a third party can't
tell what this OS is") conflated two distinct properties — unlinkability
(traffic is visible, but sessions can't be correlated to one origin over
time) and undetectability (an observer can't tell communication is
happening at all). These call for different, and not fully overlapping,
mechanisms.

**Decision:** Design explicitly for unlinkability: MAC randomization per
network attach, IPv6 privacy addresses (RFC 4941), rotating egress via an
onion-routing overlay, TCP/IP stack fingerprint normalization (matching a
common high-population profile, not a novel one), per-application network
compartmentalization, and DNS-over-HTTPS/TLS by default. Undetectability
(traffic morphing, cover traffic) is explicitly out of scope for now —
a harder, separate problem, noted as a possible future stretch goal.

**Alternatives considered:** Attempting a literally novel/unique network
stack fingerprint as a "private" signature — rejected, since a unique
fingerprint is *more* identifiable than blending into a large existing
population, the opposite of the goal.

**Consequences:** Full design in `07-PRIVACY-AND-IDENTITY.md`, including
an explicit honest-limits section (global traffic correlation and
endpoint compromise are not solved by this layer).

---

### ADR-0010 — Storage/malware resistance: bounded blast radius + guaranteed recovery, not "unhackable"
**Status:** Decided

**Context:** "Unhackable" and "ransomware-proof" are unfalsifiable
absolute claims, achievable by no system that executes arbitrary code.

**Decision:** Replace the absolute claim with a falsifiable design target
— no single compromise has unbounded blast radius, and recovery from any
single compromise is always possible — implemented via capability-scoped
deny-by-default storage access, an immutable/verified base system with
atomic updates and verified boot, copy-on-write snapshotting of user data,
and deny-by-default execution capabilities for new binaries.

**Alternatives considered:** Signature/heuristic-based malware scanning
as the primary defense — rejected as a detection-based approach (defends
against known patterns) rather than a structural one (removes the
mechanism ransomware depends on); may still be worth adding later as a
secondary layer, not a replacement for the structural approach.

**Consequences:** Full design in `08-STORAGE-AND-INTEGRITY.md`, including
an honest-limits section (hardware side channels, social-engineered
capability grants, and supply-chain compromise are explicitly not solved
by this layer — the capability-grant UX becomes a real open design
problem rather than something this architecture quietly solves for free).

---

### ADR-0011 — Personality-server architecture; no ABI/API surface lives in the kernel
**Status:** Decided

**Context:** The initial idea ("Wine in the kernel") would have placed a
huge, historically bug-dense API translation surface directly in the most
trusted execution context, directly contradicting ADR-0001's minimal-TCB
rationale. Separately, "the ability to redefine what the OS does" was
raised as its own, seemingly unrelated request.

**Decision:** Application-facing API/ABI surfaces are implemented as
userspace **personality servers** — ordinary, capability-scoped,
independently restartable processes — never as kernel code. A Win32
personality server is the correct home for Wine-equivalent functionality.
Adding a new "definition of what the OS does" means adding a new
personality server, never a kernel change.

**Alternatives considered:** In-kernel Win32 translation layer — rejected
per ADR-0001/`00-PHILOSOPHY.md`. A single fixed native API with no
personality concept — rejected as foreclosing both the Win32-compat goal
and the "redefine the OS" goal for no architectural benefit, since the
kernel was already designed to have no opinion about application-facing
API shape.

**Consequences:** Full design in `09-PERSONALITY-SERVERS.md`, including
precedent (Windows NT's environment subsystems, L4Linux) and an honest
scope-setting note that broad Win32 compatibility is a decades-scale
undertaking realistically started narrow, not attempted broadly on day
one.

---

### ADR-0012 — UI: powerbox-style capability granting, pluggable unprivileged shell
**Status:** Decided

**Context:** Every property from ADR-0001 onward can be undone at the UI
layer if permission/authority is presented as a conventional bolted-on
dialog system rather than integrated with how capabilities actually work.

**Decision:** Adopt the powerbox pattern (CapDesk/Polaris precedent —
Stiegler, Karp, Yee, Close, Miller) for file and resource grants: the
picker action itself is the capability grant. Require the compositor to
guarantee a secure input path between the picker and the user, isolated
from untrusted application input/output — a hard requirement, not a
nice-to-have, since the powerbox model is spoofable without it (as it
historically was under the X Window System). Desktop shell/compositor is
unprivileged userspace, following the personality-server pattern from
ADR-0011.

**Alternatives considered:** Conventional static permission-grant dialogs
("Allow this app to access your files: Yes/No") — rejected as the
well-documented failure mode this design specifically avoids: a wall of
text that trains users to click through it.

**Consequences:** Full design in `10-USER-INTERFACE.md`, including an
honest-limits section citing a real usability study (DeWitt & Kuljis,
2006) showing even well-designed capability UI doesn't fully eliminate
careless grants — reduces them, doesn't solve the human element outright.
