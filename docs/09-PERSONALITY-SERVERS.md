# 09 — Personality Servers

## Core idea

The kernel, per `01-ARCHITECTURE.md`, has no opinion about what "the OS"
looks like to an application — it provides capabilities, IPC, scheduling,
and MMU control, full stop. **What an application experiences as "the
operating system's API" is entirely defined by a personality server**: a
userspace process (or small set of cooperating processes) that translates
an application-facing API/ABI into capability invocations on the
underlying kernel primitives. Multiple personality servers can coexist on
one running kernel, isolated from each other, and a binary is launched
"under" whichever one it needs.

This is not a new idea invented for this project, which is exactly why
it's trustworthy as a foundation:

- **Windows NT's original architecture** had a small native kernel API
  with separate userspace "environment subsystems" — Win32, POSIX, OS/2 —
  each presenting a different face to applications running under it. The
  "personality" was never a kernel property; it was a userspace subsystem
  choice, made per process.
- **L4Linux**, in the L4/Fiasco microkernel research lineage, runs a
  paravirtualized Linux as a userspace task on top of a much smaller
  native microkernel — a "personality" at the "whole guest OS" end of the
  spectrum, on a similar underlying idea.

## Where Wine actually belongs

Real Wine already implements the overwhelming majority of the Win32 API
surface in userspace, translating calls to POSIX equivalents — this is a
deliberate architectural choice on Wine's part specifically to avoid
needing custom kernel modules for normal operation, not an accident of
history. Putting that translation layer *inside* this kernel instead would
mean placing one of the largest, most historically bug-dense API surfaces
in computing directly in the most trusted execution context available —
precisely the "parsers and complex logic don't belong in the kernel" rule
from `00-PHILOSOPHY.md`, violated in the most consequential way possible.

The fix is the personality-server model applied directly: **a Win32
personality server is an ordinary, unprivileged, capability-scoped,
independently restartable userspace process**, holding only the
capabilities it needs to implement Win32 semantics for its client
processes (file access mediated through the same capability-scoped
storage as everything else in `08-STORAGE-AND-INTEGRITY.md`, not a
special ambient-access carve-out). If it has a bug and gets exploited, the
blast radius is exactly what its own capability grants allow — nothing
about the exploit reaches the kernel, and nothing about it reaches another
personality's processes. Same practical outcome as running Windows
binaries; none of the risk of running them in ring 0.

## "Ability to redefine what the OS is supposed to do"

This is the same feature, described from a different angle. Since "what
the OS does" is entirely late-bound to which personality server a program
is launched under, adding a genuinely new definition of "the OS" — a
custom native API surface, a POSIX-compatible one, a Win32 one, or
something not yet invented — means writing a new userspace server. It
never means touching the kernel, and it never means a reboot to switch
which personality a given launch uses. In the most literal available
sense, what this OS "is" to a given process is redefinable per-launch.

## Design mechanics

- **Personality selection at launch**: determined either explicitly (a
  launcher specifies which personality a binary needs) or by binary
  format detection (a PE header implies Win32, an ELF header implies the
  native or POSIX personality) — detection logic itself lives in the
  loader, in userspace, not the kernel.
- **Capability scoping per personality**: each personality server holds
  only the capabilities required to implement its own API surface for its
  own clients — e.g. the POSIX personality translates `open`/`read`/
  `write` into IPC calls to the filesystem server, and holds no ambient
  filesystem authority of its own beyond what it needs to do that
  translation.
- **Isolation between personalities**: personality servers do not share
  capabilities or address space with each other by default. Cross-
  personality communication, if ever needed, is an explicit, narrow,
  capability-gated channel — never an implicit shared resource.

## Honest limits

Full binary compatibility with real Windows software — the level real
Wine achieves — represents over two decades of reverse-engineering Win32
semantics by a large community. A from-scratch Win32 personality server
here should not target general compatibility on day one; the realistic
starting point is a small, explicit set of API calls sufficient for a
specific target binary or narrow class of binaries, expanded
incrementally. Treat broad compatibility as a legitimate long-term stretch
goal (see `06-ROADMAP.md` — not yet scheduled), not an early milestone.

See ADR-0011 in `04-DECISIONS.md` for the decision record.
