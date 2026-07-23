# 03 — Syscall ABI

## Two separate concerns, addressed separately

This doc deliberately splits into two layers that are easy to conflate:

1. **How does a thread cross the user/kernel boundary at all** — answered by
   SYSCALL/SYSRET.
2. **How does an I/O-heavy client avoid paying that crossing cost per
   operation** — answered by a shared-ring async layer, built *on top of*
   (1), not a replacement for it.

Treating these as alternatives to each other produces a wrong mental model
at the foundation of the whole ABI, so they're pinned down explicitly here.

## Layer 1: SYSCALL/SYSRET as the sole kernel entry mechanism

x86-64 offers three historical ways into the kernel: `INT 0x80` (software
interrupt — full IDT gate lookup, slow), `SYSENTER`/`SYSEXIT` (Intel's fast
path, but cannot cleanly return to a 64-bit CPL3 context), and
`SYSCALL`/`SYSRET` (the fast path this project uses exclusively, no legacy
entry vector supported — see ADR-0005).

**Enabling it:**
- `IA32_EFER` MSR (`0xC0000080`), bit 0 (`SCE`, syscall enable) = 1.
- `IA32_STAR` MSR (`0xC0000081`): packs the segment selectors used on entry
  (kernel CS/SS) and return (user CS/SS) into one MSR.
- `IA32_LSTAR` MSR (`0xC0000082`): the 64-bit RIP the CPU jumps to on
  `SYSCALL`.
- `IA32_CSTAR` MSR (`0xC0000083`): entry point for `SYSCALL` from 32-bit
  compatibility-mode user code — left unprogrammed / trapped as invalid,
  since this project has no 32-bit userspace ambitions (ADR-0007 territory).
- `IA32_FMASK` MSR (`0xC0000084`): a mask applied to RFLAGS on entry —
  critically, mask `IF` here so interrupts are atomically disabled for the
  first instructions of the kernel entry path, before the kernel has a
  known-good stack to take an interrupt on.

**The gotcha to design around, not discover later:** `SYSCALL` unconditionally
clobbers `RCX` (loaded with the return RIP) and `R11` (loaded with the
return RFLAGS) so that `SYSRET` can restore them. Neither register is
available for argument passing. This is exactly why Linux's x86-64 syscall
convention uses `R10` instead of `RCX` for its fourth argument — same
constraint, same fix, worth inheriting rather than rediscovering:

| Purpose | Register |
|---|---|
| Syscall/capability-invocation number | `RAX` |
| Arg 1–3 | `RDI`, `RSI`, `RDX` |
| Arg 4 (not `RCX` — clobbered) | `R10` |
| Arg 5–6 | `R8`, `R9` |
| Clobbered by the instruction itself | `RCX` (return RIP), `R11` (return RFLAGS) |

## Minimal syscall surface

In keeping with `00-PHILOSOPHY.md`, this is not a POSIX-style surface with
hundreds of entry points. Following the seL4 model, there are on the order
of a dozen true kernel entry reasons, and *everything* else is a capability
invocation shaped as a message to one of those entry points — creating an
object, mapping a page, and sending an IPC message are all "the same kind
of operation" from the kernel's point of view, just addressed at different
capabilities:

- `Send` / `NBSend` (non-blocking send) / `Recv` / `NBRecv` / `Call` /
  `Reply` — the IPC primitive family (see `01-ARCHITECTURE.md`'s `Endpoint`
  object).
- `Yield` — cooperative scheduling hint.
- A small family of debug/introspection entries, compiled out of
  non-debug builds entirely (not just gated at runtime) so they cannot be
  a production attack surface.

A large, ad-hoc syscall table is itself a tech-debt generator: every entry
is a permanent piece of kernel ABI that must be supported forever. Routing
everything through capability invocation on a handful of object types means
new functionality is new *object types and userspace servers*, not new
*kernel syscalls*.

## Fast-path IPC

The common case — a short `Call`/`ReplyRecv` round trip between two
threads, arguments fitting in registers — should be hand-tuned to stay
entirely in registers with no memory access on the hot path, mirroring
seL4's fastpath design (a deliberately separate, minimal code path from the
general slow path, which handles every non-common case — multi-word
transfers, capability transfer, error conditions — in ordinary, less
performance-critical C).

## Layer 2: async I/O — a shared-ring model, explicitly *not* a kernel entry mechanism

For I/O-heavy workloads (storage, networking), paying a full SYSCALL/SYSRET
round trip *per operation* is real, avoidable overhead. The fix is a shared
memory ring between the client and the userspace server that owns the
device (e.g. a block-driver process) — architecturally the same idea as
Linux's `io_uring`, deliberately reused here because it's a well-proven
pattern, not reinvented from scratch:

- A **submission ring** and **completion ring**, in memory mapped into both
  the client and the server, addressed via pre-registered, capability-backed
  buffers (no raw pointers crossing the boundary — the buffer itself was
  granted as a capability up front).
- A **`Notification` object** (see `01-ARCHITECTURE.md`) signals ring
  activity — this is a single bit being set, not a syscall, and can be
  waited on by a blocked thread or polled by a busy one.
- `SYSCALL` is used only to *set up* the ring and notification objects once,
  and optionally to block-wait when spinning would waste CPU — never per
  I/O operation in the steady state. A sufficiently busy server can run in
  a polling mode that avoids the boundary crossing almost entirely, exactly
  as `io_uring`'s `SQPOLL` mode does.

The design takeaway: SYSCALL/SYSRET answers "how do we get into the kernel
at all," the ring answers "how do we avoid needing to, most of the time."
Both belong in the same ABI, at different layers, not as competing choices.
