# 11 — Getting Started

Everything in this repo so far is architecture, deliberately written
before any code. This doc is the bridge to actually starting: what to
install, what to test with, and a realistic sense of pacing.

## Toolchain

- **Cross-compiler**: `x86_64-elf-gcc` (or clang targeting `x86_64-elf`),
  built following the OSDev Wiki's "GCC Cross-Compiler" guide. Do not
  compile kernel code with the host system's native gcc — it assumes a
  hosted environment (libc availability, host ABI conventions) that a
  freestanding kernel doesn't have, and the mismatch causes exactly the
  kind of subtle bug that's expensive to trace back to its source.
- **Assembler**: NASM, for the boot/entry assembly that C can't express —
  GDT/IDT loading, the real-mode AP trampoline from `02-BOOT.md`.
- **Build system**: plain `make`. No build-system abstraction until a
  second real need for one exists, per `00-PHILOSOPHY.md`.
- **Linker script**: a custom `.ld` script placing kernel sections at the
  higher-half virtual address (`01-ARCHITECTURE.md`'s address space
  layout) while the physical load address differs. Worth drafting as the
  very first artifact once code starts, since nothing links without it.

## Bootloader

Limine (ADR-0008 in `04-DECISIONS.md`). Start from Limine's own
"barebones" template repository rather than integrating it from scratch —
it already demonstrates the exact handshake `02-BOOT.md` describes
(memory map, framebuffer, RSDP pointer, higher-half mapping) and is the
fastest path to a real M1 milestone.

## Fast-iteration testing

- **QEMU** (`qemu-system-x86_64`) as the primary inner loop — orders of
  magnitude faster than real-hardware iteration for the majority of bugs.
- **OVMF** (TianoCore EDK2's open UEFI firmware build) as the firmware
  QEMU boots, since real proprietary UEFI firmware isn't something to
  bundle or rely on for testing.
- **`-no-reboot -no-shutdown`**: without these flags, a triple fault
  silently resets the VM with no indication anything went wrong at all —
  this single pair of flags turns an invisible failure into a debuggable
  one, and is worth setting from the very first boot test.
- **`-d int,cpu_reset -D qemu.log`**: logs interrupt and reset activity
  to a file, useful for the exact "why did this exception fire" questions
  that come up constantly through `02-BOOT.md`'s exception-handling work.
- **`-s -S`**: exposes QEMU's built-in GDB stub and freezes the CPU at
  startup until a debugger attaches. Connecting with
  `target remote localhost:1234` lets the long-mode transition, GDT/IDT
  setup, and early paging code be single-stepped exactly like ordinary
  userspace code — one of the highest-leverage tools available for this
  entire phase of the project.

## Real hardware (the actual end goal — milestone M8)

- A **dedicated test machine**, not a daily driver — early bring-up will
  hang, corrupt boot media, or need a hard power cycle at some point; this
  is normal and expected, not a sign something is unusually wrong.
- A **USB-to-TTL serial adapter** (inexpensive, widely available). Before
  framebuffer output is working, serial is often the only diagnostic
  channel available — exactly why `02-BOOT.md` calls out serial panic
  output as needing to exist before framebuffer output does, not after.
- A **fast USB-rewrite workflow** — `dd`, or a tool like Ventoy that
  avoids a full reformat per iteration — since the M8 phase involves many
  more boot-and-observe cycles than QEMU testing alone would suggest.

## Reference material

- **OSDev Wiki** (osdev.org) — the community reference for exactly this
  kind of from-scratch build; likely to save more time than any single
  book, particularly for firmware and hardware quirks that are otherwise
  undocumented.
- **Intel Software Developer's Manual**, volumes 3A/3B/3C — the primary
  source for the APIC, paging, and exception-handling details already
  cited throughout `02-BOOT.md` and `03-SYSCALL-ABI.md`.
- **AMD64 Architecture Programmer's Manual** — worth cross-referencing
  specifically for SYSCALL/SYSRET, since AMD introduced the instruction
  pair before Intel adopted it.
- **ACPI Specification** (uefi.org/specifications) — for the MADT/RSDP
  parsing described in `02-BOOT.md`'s SMP bring-up section.
- **seL4 reference manual** — now that the capability object model's
  shape is settled in `01-ARCHITECTURE.md`, its actual API documentation
  is the most useful next read for implementation-level detail.
- **Limine Boot Protocol specification** and barebones template — the
  concrete handshake this project's kernel entry point targets.

## Project logistics

- **Git from the first file**, even working solo — commit discipline
  pairs naturally with the ADR discipline already established in
  `04-DECISIONS.md`: small, reasoned, individually-justifiable changes
  rather than large undifferentiated drops.
- **CI that boots to the M1 milestone in QEMU on every push** — cheap to
  set up this early, and catches regressions immediately rather than
  after they've compounded. This is roadmap milestone M0 itself, not a
  later nice-to-have.

## Honest timeline

Milestone-sized goals, not a single "finish the OS" target, are the
realistic way to think about pacing:

- **M1** (boot to a serial "hello world" via Limine): days to a couple of
  weeks once the toolchain itself is working.
- **M2–M4** (solid long-mode exception handling through a working
  capability core: `Untyped`/`Retype`/`CNode`/`TCB`): where most solo
  hobby efforts spend months, not weeks — this is genuinely the hardest,
  least linear part of the whole project.
- **M8** (real hardware boot with a working userspace driver): realistically
  a multi-month-to-a-year part-time effort for one person, based on how
  comparable hobby-kernel projects have actually gone. This is not a
  warning sign — it's the normal shape of a project at this scope, and
  exactly why the roadmap is staged the way it is.

## Community

The **OSDev Discord and forums** are the single highest-leverage resource
not already listed above — firmware and real-hardware quirks are almost
always a long-tail problem where someone has already hit the exact issue
on the exact chipset in question, and no reference manual covers that.
