# 00 — Philosophy

## Prime directive

When a bug class is possible, the default instinct is to write a check for
it. The instinct this project uses instead: change the design so the bug
class has nowhere to live. A check is something you can forget to add, forget
to update, or get subtly wrong. A structural absence doesn't need to be
remembered.

This is not mysticism — it's a known lineage in serious OS design:

- **seL4** doesn't test for unauthorized memory access; the capability model
  makes an unreachable object *unrepresentable*, and the kernel is formally
  proven (Isabelle/HOL) to uphold that.
- **Capability security** (seL4, Fuchsia's Zircon, historic EROS/CapROS)
  removes ambient authority entirely. A "confused deputy" bug — a privileged
  component tricked into misusing its own authority on a caller's behalf —
  cannot occur if the component was never handed the authority in the first
  place.
- **Erlang/OTP supervision trees** don't prevent every crash; they make
  crashes cheap, isolated, and auto-restartable, so "one bad driver takes
  the machine down" is not a bug you fix, it's a bug that isn't there.

Every subsystem doc in this repo should be able to answer: *what bug class
does this design make impossible, as opposed to unlikely?*

## What this means concretely, here

- **No ambient authority.** A process can only affect what it holds a
  capability to. Not "shouldn't" — *cannot*, because there is no other
  addressing mode for kernel objects.
- **No kernel heap.** All kernel memory comes from `Untyped` capability
  objects, explicitly retyped by userspace policy (the root task), matching
  seL4's model. This deletes "unbounded kernel allocation" and "kernel OOM"
  as categories, at the cost of pushing allocation *policy* to userspace,
  where it belongs.
- **Drivers are userspace, IOMMU-isolated, supervised.** "A driver crash
  takes the kernel with it" doesn't exist because a driver crashing is a
  process exiting, caught by a supervisor, optionally restarted with a fresh
  capability set. See `07-DRIVER-MODEL.md` (roadmap item, not yet written).
- **One syscall ABI, generated, not hand-synced.** "Kernel and userspace
  silently disagree about a struct layout" doesn't exist because both sides
  are generated from one interface description, not maintained as parallel
  hand-written headers.
- **Real hardware from early on, not "someday."** "Works in QEMU, breaks on
  real metal" doesn't exist as a late surprise because the roadmap puts a
  real-hardware boot test in the loop starting at a specific milestone
  (M8), not after the design is already ossified around VM-only quirks.

## What to use

- **Freestanding C**, no libc dependency inside the kernel (`-ffreestanding
  -nostdlib`, your own tiny string/memory primitives).
- **Explicit, bounded control flow in the kernel**: no recursion in kernel
  code paths, no unbounded loops, no VLAs. This is a MISRA-C-adjacent
  subset, chosen for auditability, not certification-box-checking.
- **Sized integer types everywhere** (`stdint.h`): `uint64_t`, not
  `unsigned long`. Ambiguity in width is a bug waiting for a new target.
- **Static analysis as a gate, not a suggestion**: `-Wall -Wextra -Werror`,
  Clang Static Analyzer, `cppcheck`, and treat any warning in kernel code
  as a build failure.
- **Host-testable core algorithms.** Anything that isn't inherently
  hardware-coupled (capability derivation tree logic, scheduler queue math)
  should be compilable and testable on the host with ASan/UBSan, *before*
  it's trusted to run in ring 0.
- **One spec before one line of code**, per subsystem. If you can't write
  the object model down, you don't understand it well enough to implement
  it yet.

## What to avoid

- **A dynamic kernel heap growing "just for now."** If you catch yourself
  reaching for `kmalloc`-equivalent inside the kernel proper, that's a
  signal the allocation belongs in userspace policy instead.
- **Speculative generality.** No hardware abstraction layer until a *second*
  real hardware target actually exists. One target, one code path — do not
  pre-build the seam for a second architecture you don't have yet.
- **The "temporary" hack.** If a fix isn't the right fix, it doesn't get
  merged with a `// TODO` — either it's right, or the ticket stays open.
  Tech debt in a hobby project has no economic incentive to ever get paid
  down, so the discipline has to be "don't take the loan."
- **Config-driven flexibility nobody asked for.** Every `#ifdef` branch,
  every "make it pluggable" abstraction, is a permanent tax on every future
  reader. Add the seam when the second real need for it exists, not before.
- **Parsers and complex logic in the kernel.** Filesystem parsing, network
  protocol parsing — none of it belongs in ring 0. If it can misparse
  attacker-controlled bytes, it belongs in an isolated, unprivileged,
  restartable process.
- **Legacy compatibility paths that outlive their reason.** xAPIC/PIC
  support, 32-bit compatibility mode, BIOS/CSM boot — each one is a decision
  to *not* carry, recorded explicitly in `04-DECISIONS.md`, rather than a
  default kept "just in case."

## Process principle

Every non-trivial decision gets an ADR entry in `04-DECISIONS.md` — context,
decision, alternatives considered, consequences — before it's acted on. The
goal isn't paperwork for its own sake; it's that six months from now, "why
did we do it this way" should have a two-minute answer instead of a
two-hour archaeology dig through code.
