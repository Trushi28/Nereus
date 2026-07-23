# 06 — Roadmap

Each milestone should be a genuinely runnable, demoable state — not "half
of a feature." Real hardware enters the loop at M8, deliberately not
deferred to the very end, so hardware-only bugs surface while the design
is still cheap to change.

| # | Milestone | Exit condition |
|---|---|---|
| M0 | Toolchain + scaffolding | Cross-compiler (`x86_64-elf-gcc`), `qemu-system-x86_64` + OVMF harness, and CI all run a trivial build/boot smoke test |
| M1 | UEFI hand-off → kernel stub | Bootloader (ADR-0008 pending) hands off to a kernel entry point that prints to serial and halts cleanly |
| M2 | Long mode solidified | Real GDT/IDT/TSS with IST-backed `#DF`/NMI/`#MC` handlers; every CPU exception produces a structured register dump instead of a silent reset |
| M3 | x2APIC + timer + minimal scheduling | TSC-deadline timer firing; two kernel-only test threads round-robin on the BSP |
| M4 | Capability core: `Untyped`/`Retype`/`CNode`/`TCB` | Root task creates and schedules a real (non-kernel-builtin) userspace thread |
| M5 | Synchronous IPC | Root task and a second process exchange a `Call`/`Reply` round trip |
| M6 | SMP | All detected cores brought up via INIT-SIPI-SIPI, each running the scheduler |
| M7 | First userspace driver | Serial UART fully owned by a userspace driver process, IOMMU-isolated, with a supervisor that restarts it on crash |
| M8 | **Real hardware boot** | Boots via USB on at least one real UEFI x86-64 machine — not just QEMU — reaching the same state as M7 |
| M9 | Minimal shell + fun-subsystems layer | Forth-style bring-up console (see `05-FUN-SUBSYSTEMS.md`) usable interactively over serial |

## Sequencing notes

- **M2 before M3**: a scheduler without solid exception handling just
  produces unexplained resets under load — diagnosability has to come
  first, not be bolted on when something breaks.
- **M8 is not "the end"**: it's placed deliberately mid-roadmap so that
  real-hardware-only surprises (ACPI table quirks, a UEFI implementation
  that behaves differently from OVMF) get found while the capability
  model and IPC design are still cheap to adjust, rather than after M9+
  work has been built on top of assumptions QEMU happened to satisfy.
- **Nothing after M9 is fixed yet** — deliberately. Once the base is
  running on real hardware with a working driver model, the fun-subsystem
  ideas in `05-FUN-SUBSYSTEMS.md` become genuinely buildable rather than
  speculative, and priority among them is worth revisiting with real
  running-system experience in hand rather than deciding now.
