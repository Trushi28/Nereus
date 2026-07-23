# 01 — Architecture Overview

## System shape

```
                         ┌─────────────────────────────┐
                         │           Hardware           │
                         └───────────────┬──────────────┘
                                          │
                         ┌────────────────▼─────────────┐
                         │            Kernel              │
                         │  capability objects · sync IPC  │
                         │  scheduler · MMU control         │
                         │  interrupt→notification dispatch │
                         └───────────────┬───────────────┘
                                          │ (all authority starts here)
                         ┌────────────────▼───────────────┐
                         │           Root task              │
                         │  holds every Untyped + IRQ +      │
                         │  boot-time device capability      │
                         │  splits authority to children      │
                         └───┬──────┬──────┬──────┬────────┘
                             │      │      │      │
                    ┌────────▼┐ ┌───▼───┐ ┌▼─────┐ ┌▼──────────┐
                    │ Memory   │ │Device │ │ FS   │ │  Init /   │
                    │ manager  │ │drivers│ │server│ │  service  │
                    │(policy)  │ │(user- │ │      │ │  manager  │
                    │          │ │space) │ │      │ │           │
                    └──────────┘ └───────┘ └──────┘ └─────┬─────┘
                                                            │
                                                     ┌──────▼──────┐
                                                     │ User programs │
                                                     └───────────────┘
```

The kernel is deliberately small and dumb. It provides *mechanism*
(capability objects, synchronous IPC, scheduling, MMU control, interrupt
delivery) and holds essentially no *policy*. Every policy decision — how
much memory a process gets, which driver owns which device, what the
filesystem looks like — lives in userspace, starting with the root task.

## What the kernel does and does not do

| Kernel does | Kernel does not do |
|---|---|
| Capability object management (create/derive/revoke) | Filesystems |
| Synchronous IPC (Call/Send/Recv/Reply) | Network protocol stacks |
| Scheduling (fixed-priority, budget-aware) | Device drivers (except a minimal boot console) |
| MMU/page-table manipulation | Memory allocation *policy* |
| Interrupt delivery as capability-guarded notifications | Process/service lifecycle policy |
| Thread and address-space object lifecycle (mechanism only) | Naming, mounting, permissions beyond capability possession |

## Capability object taxonomy

Every kernel-managed resource is a typed object, referenced only through a
capability held in a process's CNode (a capability address space, itself a
kernel object). There is no other way to name a kernel resource — no global
handle table, no PID-indexed lookup for privileged operations.

| Object | Represents | Key operations |
|---|---|---|
| `Untyped` | A range of physical memory, not yet typed | `Retype` into any object below |
| `CNode` | A capability address space (array of capability slots) | `Mint`, `Copy`, `Move`, `Delete`, `Revoke` |
| `TCB` | A schedulable thread | `Configure`, `SetPriority`, `Resume`, `Suspend` |
| `Endpoint` | A synchronous IPC rendezvous point | `Send`, `Recv`, `Call`, `Reply` |
| `Notification` | An asynchronous, non-blocking signal (word-sized bitset) | `Signal`, `Wait`, `Poll` |
| `PageTable` / `PML4` | A level of the x86-64 page table hierarchy | `Map`, `Unmap` |
| `Frame` | A physical page usable for a mapping | `Map`, `Unmap` |
| `IRQControl` → `IRQHandler` | Authority to receive a specific interrupt | `Ack`, bound to a `Notification` |
| `ASIDPool` | Address-space-ID allocation for TLB tagging | `Assign` |

`Retype` is the only way new objects come into existence, and it always
consumes `Untyped` capacity — there is no allocation path that doesn't
originate from a capability the caller already held. This is the mechanism
that deletes "kernel heap exhaustion" as a category: exhaustion becomes "the
root task's memory-manager policy ran out of budget it chose to grant,"
which is a userspace accounting problem, not a kernel one.

## Address space layout (x86-64, canonical, higher-half kernel)

```
0xFFFF_FFFF_FFFF_FFFF  ┐
                       │  Kernel image, fixed link address
0xFFFF_FFFF_8000_0000  ┘

0xFFFF_8000_0000_0000  ┐
                       │  Direct physical memory map (all of RAM,
                       │  offset-mapped, kernel-only)
0xFFFF_8000_0000_0000  ┘  (size = installed RAM)

        ── non-canonical hole (bits 47:63 must match bit 47) ──

0x0000_7FFF_FFFF_FFFF  ┐
                       │  Userspace, per-process, fully attacker-controlled
0x0000_0000_0000_0000  ┘
```

Standard 4-level paging (CR4.LA57 left off initially — 5-level paging is a
day-two optimization for >256 TiB address spaces, not a day-one need; see
`04-DECISIONS.md` if this changes). The direct physical map lets the kernel
touch any physical page without a per-access page-table walk, at the cost of
that mapping being kernel-only and never exposed to user page tables.

## Boot-time authority distribution

At boot, the kernel constructs exactly one fully-privileged process: the
root task. It receives capabilities to every unused `Untyped` region of
physical memory, every `IRQControl` capability, and capabilities describing
boot-provided devices (framebuffer, boot console UART). Every other process
in the system receives authority *only* by explicit grant from something
that already held it — there is no back door, no root-equivalent syscall,
no "are you PID 0" check anywhere in the kernel. Authority is a capability
you can point to, not a role you can claim.
