# Nereus OS

*Named for the Greek shapeshifting sea-god — a nod to the personality-server
architecture (`docs/09-PERSONALITY-SERVERS.md`), and a name that (unlike its
sibling Proteus) doesn't collide with an existing circuit simulator or Linux
distro.*

A capability-based microkernel operating system for x86-64, written in C,
booting natively via UEFI on real hardware, built around three pillars:
an **unlinkable network identity** (rotating MAC/IP, fingerprint-normalized
stack), **capability-scoped storage** that bounds the blast radius of any
single compromise instead of claiming to be "unhackable," and a **pluggable
personality-server model** so the OS's application-facing API is late-bound
per process rather than fixed in the kernel.

seL4 is cited once, deliberately, for exactly one thing: proof that a
capability-object, minimal-TCB kernel is a sound mechanism. Nothing past
that base is seL4's — the privacy layer, the storage/ransomware-resistance
model, and the personality-server architecture have no seL4 analog.

## Core properties

| Axis | Choice |
|---|---|
| Kernel design | Capability-based microkernel (seL4-style) |
| Language | C (freestanding, no libc in kernel) |
| Target | x86-64, UEFI only, no legacy BIOS/CSM |
| Interrupt model | x2APIC only (no persistent xAPIC/PIC fallback) |
| Kernel entry | SYSCALL/SYSRET, register-only fast path |
| Memory model | No kernel heap — untyped-object capability allocation |
| Drivers | Userspace, IOMMU-isolated, supervised/restartable |
| Network identity | Unlinkable by design: rotating MAC/IP, normalized TCP/IP fingerprint |
| Storage/malware model | Capability-scoped, deny-by-default; bounded blast radius + guaranteed recovery |
| App compatibility | Pluggable personality servers (native, POSIX, Win32) — none privileged, none in-kernel |
| Status | **Docs-only, pre-code** |

## Why this combination

The goal isn't "a kernel that happens to work" — it's a kernel where entire
bug classes are structurally impossible rather than defended against at
runtime. See `docs/00-PHILOSOPHY.md` for the reasoning; the short version is
that a small, capability-mediated, statically-allocated kernel makes most of
the traditional OS bug taxonomy (privilege escalation, driver-crash-kills-box,
ABI drift, use-after-free-of-a-capability) either impossible by construction
or contained to a single restartable process.

## Doc index

1. `docs/00-PHILOSOPHY.md` — engineering principles, what to use / what to avoid, and why
2. `docs/01-ARCHITECTURE.md` — kernel object model, address space layout, component diagram
3. `docs/02-BOOT.md` — UEFI hand-off through long mode, x2APIC bring-up, SMP start
4. `docs/03-SYSCALL-ABI.md` — SYSCALL/SYSRET mechanics, calling convention, async I/O layering
5. `docs/04-DECISIONS.md` — ADR log: every major decision, alternatives considered, consequences
6. `docs/05-FUN-SUBSYSTEMS.md` — the "interesting shit" — creative subsystems worth building
7. `docs/06-ROADMAP.md` — milestone sequence from hello-world to real-hardware boot
8. `docs/07-PRIVACY-AND-IDENTITY.md` — unlinkable network identity: MAC/IP rotation, fingerprint normalization, honest limits
9. `docs/08-STORAGE-AND-INTEGRITY.md` — ransomware/malware resistance: capability-scoped storage, snapshots, honest limits
10. `docs/09-PERSONALITY-SERVERS.md` — pluggable OS "personality" model; where Wine-equivalent compatibility actually lives
11. `docs/10-USER-INTERFACE.md` — powerbox-style capability granting, secure input path, pluggable shell, anti-fingerprinting rendering

## Build status

Nothing compiles yet — this is the architecture pass, deliberately, so the
first line of code is written against a spec instead of vibes. Planned
toolchain (see roadmap M0): `x86_64-elf-gcc` cross-compiler, `qemu-system-x86_64`
with OVMF (UEFI firmware for QEMU) as the fast inner-loop target, with real
UEFI hardware boot tested from milestone M8 onward — not deferred to "someday."

