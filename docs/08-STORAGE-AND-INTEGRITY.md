# 08 — Storage & Integrity (Ransomware/Malware Resistance)

## Reframing the goal

"Unhackable" and "ransomware-proof" are unfalsifiable claims — there's no
design review that can confirm them, and no system that executes arbitrary
code has ever earned them honestly. The actual, testable design target,
which fits `00-PHILOSOPHY.md` far better than an absolute claim would:

> **No single compromise has unbounded blast radius, and recovery from any
> single compromise is always possible.**

That's falsifiable (you can point at a specific compromise and ask "was
the blast radius actually bounded? was recovery actually possible?"), and
it's a stronger, more honest engineering commitment than "unhackable" ever
was.

## Why ransomware works today, and why that mechanism doesn't exist here

Conventional ransomware's core operation is trivial by design: gain code
execution as some user → enumerate every file that user's ambient
authority can reach → encrypt it → demand payment. It works because
ambient authority is the default: a process inherits everything its
launching user could touch, whether or not it ever needed to.

### Capability-scoped storage, deny-by-default
A process holds no filesystem authority except capabilities explicitly
granted to it — not "shouldn't," *cannot*, per the object model in
`01-ARCHITECTURE.md`. A newly launched program starts with zero file
capabilities. Access to a specific file or directory subtree is granted
by an explicit action (a file-picker-style grant, in the spirit of
Android's Storage Access Framework or GrapheneOS's scoped storage, but
enforced at the capability layer rather than as a permission check
layered on top of ambient access that still exists underneath). This
collapses "everything reachable" from "everything the user owns" down to
"only what this specific process was actually handed" — ransomware's
enumeration step has nothing to enumerate beyond its own grants.

### Immutable base system + atomic updates + verified boot
The root filesystem is read-only and integrity-verified (a hash-tree
check at boot, dm-verity-style, or a content-addressed immutable store in
the Nix/Fuchsia lineage). Updates install a new version alongside the old
one and switch atomically; the previous version stays available as an
instant rollback target. Verified boot measures each stage and refuses to
hand control to a modified next stage. Net effect: "malware persists by
corrupting the OS itself" stops being a live risk and becomes "boot the
last known-good version," a solved, recoverable case rather than a crisis.

### Copy-on-write snapshotting for user data
Periodic, cheap, automatic snapshots of user data volumes (ZFS/Btrfs-style
COW), retained independently of the live filesystem. Even in a scenario
where something *was* granted broad capability over a data volume and used
it destructively, recovery is "roll back to the last snapshot" — a
mitigation many organizations already rely on against ransomware today,
adopted here as a first-class design feature rather than an
afterthought backup policy.

### Deny-by-default execution capabilities
A new or unknown binary starts with a minimal capability set: no network,
no filesystem beyond its own package contents, no device access. Every
additional capability is an explicit grant, tied to the personality server
or launch context it's running under (see `09-PERSONALITY-SERVERS.md`).
This is the actual root-cause fix for "you ran a downloaded program and it
could reach everything you could reach" — the conventional-OS failure mode
that makes most malware damage possible in the first place.

## Honest limits — what this does not solve

- **Hardware-level side channels** (Spectre/Meltdown-class speculative
  execution attacks, Rowhammer-style memory disturbance) are a separate
  problem class, addressed by microarchitectural mitigations and CPU
  selection, not by the capability/storage model. Capability isolation
  assumes the hardware isolation it's built on actually holds.
- **Social engineering that gets a user to explicitly grant broad
  capabilities to something malicious** is not fixed by this
  architecture — the architecture reduces *ambient* risk, not *granted*
  risk. A user who grants an app access to their entire documents folder
  because a dialog told them to has re-created the ambient-authority
  problem by hand. The UX design of capability-grant dialogs is a real,
  open design problem this project needs to take seriously, not a solved
  one — a bad grant-consent UI can quietly undo everything above.
- **Supply-chain compromise** (a legitimately trusted, signed application
  is itself malicious, or gets compromised upstream) is mitigated by
  minimizing what any single application is ever granted, not eliminated
  by capability architecture alone.
- **None of the above makes the phrase "unhackable" true.** It makes
  specific, named failure modes — mass file encryption, kernel/OS
  persistence, ambient-authority abuse by a downloaded binary — either
  structurally bounded or recoverable. That's the honest claim, and it's
  the one worth building toward.

See ADR-0010 in `04-DECISIONS.md` for the decision record.
