# 05 — Fun Subsystems

*Three ideas that started here outgrew this doc and became full
architecture pillars: unlinkable network identity → `07-PRIVACY-AND-IDENTITY.md`,
ransomware/malware resistance → `08-STORAGE-AND-INTEGRITY.md`, and the
Win32-compat / "redefine the OS" idea → `09-PERSONALITY-SERVERS.md`. What's
below is still-smaller-scope creative material.*

Ideas that are genuinely interesting *because* of the capability-microkernel
architecture — most of these are hard or meaningless to build on a
conventional monolithic kernel, which is exactly what makes them worth
doing here instead of being generic feature-list filler.

## Live capability graph inspector

Every process's authority is, by construction, a graph: which capabilities
it holds, which objects those point to, and the derivation tree showing
which capability was minted from which. A live visualizer that walks this
graph and renders it — think a security-relationship "task manager" rather
than a process list — is unique to capability systems and doubles as the
best debugging tool for "why does process X have access to Y" questions
that are otherwise genuinely hard to answer on a conventional OS.

## Deterministic record & replay for IPC

In a small microkernel, synchronous IPC between threads is close to the
*only* source of true nondeterminism (scheduling order, message arrival
order). Recording the order and content of `Endpoint` traffic is a small,
well-bounded amount of data compared to recording, say, all memory writes —
and it's enough to deterministically replay a crash. This is a much more
tractable version of "time-travel debugging" than it would be on a
monolithic kernel with shared mutable state everywhere.

## Forth-style bring-up console

Before real userland exists, early hardware bring-up benefits from an
interactive console for poking registers, reading memory, and testing
drivers by hand. A tiny Forth interpreter is a historically strong fit for
this role (see Open Firmware / OpenBoot) — small enough to write in an
afternoon, powerful enough to script real bring-up tasks, and genuinely
useful rather than a toy once serial output exists.

## Lightweight formal spec of the scheduler/capability invariants

Full seL4-style verification (Isabelle/HOL, multi-year research effort) is
out of scope for a hobby timeline, but a **TLA+ model** of a narrower
question — e.g. "can the capability derivation tree ever reach a state
where a revoked capability's descendants remain usable" — is a genuinely
achievable weekend-to-week project, and it's the kind of "unconventional
method" that catches a design bug before a single line of C exists, rather
than after it's shipped.

## Boot speed as a tracked benchmark, not an incidental property

Treat "milliseconds from UEFI hand-off to first userspace instruction" as
a first-class, regression-tracked number from the very first milestone
that boots at all — the same way a game studio tracks frame time. It's a
genuinely fun target to chase, and it forces good habits (no needless
firmware round-trips, no accidental busy-waits during bring-up) early,
before they calcify into "the boot path" nobody wants to touch.

## Capability-fuzzing chaos harness

A QEMU-based test harness that randomly interleaves capability
revocations, IPC delays, and process restarts is a natural fit for finding
TOCTOU-style capability bugs (a capability revoked in one thread while
another thread is mid-invocation on it) — a bug class that's specific to
this architecture and worth building custom tooling for rather than
relying on generic fuzzers built for POSIX-shaped systems.

## Self-expiring capabilities ("capability time bombs")

A capability minted with a built-in expiry — after a wall-clock deadline
or a fixed number of uses, it silently stops working, with no separate
revoke call required. This turns "someone forgot to revoke access after
the task finished" from an ever-present operational risk into a bug class
that structurally can't accumulate: a grant to an untrusted or short-lived
process is self-cleaning by construction. Directly buildable on the
existing capability-derivation model in `01-ARCHITECTURE.md` — the expiry
is just another field checked at invocation time.

## Amnesic session mode

A boot mode where the entire user-data volume is a copy-on-write snapshot
that's discarded — not archived, discarded — on shutdown, in the
tradition of Tails OS's amnesic design. Nothing persists across a session
unless explicitly exported to durable storage first. This is a strong,
literal complement to the unlinkability work in
`07-PRIVACY-AND-IDENTITY.md`: rotating network identity stops an observer
from linking sessions to each other over the wire, while amnesic mode
stops the machine itself from accumulating a persistent history to link
them with in the first place.

## Cryptographic duress wipe

A physical trigger (a key combo, a specific USB device's removal, a
hardware button) that instantly discards the decryption key protecting
the data volume — not an overwrite, which takes time and can be
interrupted, but crypto-shredding: the ciphertext remains on disk but
becomes permanently unrecoverable the moment the key is gone. This is a
real, established technique (this is fundamentally what an encrypted
volume's "erase" operation already is on many systems) elevated here to a
first-class, near-instant, physically-triggered response rather than a
menu option three clicks deep.

## Decoy capability grants for unproven software

Rather than a binary trust decision ("allow" or "deny") for software
you're not sure about, hand it a capability to a decoy object —
indistinguishable from the real one from the process's point of view —
and observe what it actually does before ever exposing real data. This is
the "make the problem not exist" philosophy applied to the specific
question of vetting untrusted binaries: instead of defending real data
against a program you're unsure of, don't give the question a chance to
matter, by never handing over the real object until behavior has been
observed against a fake one.

## Causal capability trace

An extension of the live capability graph inspector above: for any
observed effect (a file changed, a network packet sent), walk backward
through the exact chain of capability invocations and IPC calls that
caused it. Where the graph inspector shows *what authority exists right
now*, the causal trace answers *why did this specific thing just happen*
— pairing naturally with the deterministic IPC replay idea already above,
since a recorded IPC trace is exactly the data a causal trace needs to
walk.

## Reproducible builds and an on-device transparency log

Every system binary builds deterministically, so two independent builds
from the same source produce byte-identical output — the discipline
behind Debian's and Tor Browser's reproducible-builds efforts, adopted
here as a baseline expectation rather than an optional audit feature.
Paired with a signed, append-only, boot-verified log of every kernel and
personality-server version that has ever run on this specific machine (a
Certificate-Transparency-style log, but of your own system's own
history) — this closes the loop with `08-STORAGE-AND-INTEGRITY.md`'s
verified-boot story: not just "the currently running system is what it
claims to be," but "here's the full, tamper-evident history of what ran
before it."

## Hardware radio kill switch as a first-class capability object

A software-only "airplane mode" toggle is a promise a compromised kernel
or firmware bug can simply ignore — this has been demonstrated in
practice on more than one conventional OS. The fix: wire the actual
physical kill switch to cut power or data lines to the radio hardware
directly (the Librem-laptop precedent), and expose only a **read-only**
capability describing current switch state to software. This gives
software an honest way to know "are radios physically dead right now,"
rather than trusting a state that malicious code could spoof — a small,
concrete example of turning a software promise into a hardware fact.

