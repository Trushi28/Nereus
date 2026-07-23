# 02 — Boot Sequence

Scope: UEFI firmware hand-off through a fully running BSP (boot-strap
processor) in long mode with x2APIC active, plus the AP (application
processor) bring-up sequence for SMP. No legacy BIOS/CSM path exists —
see ADR-0007.

## 1. Firmware hand-off

1. UEFI firmware loads the **Limine** bootloader from the EFI System
   Partition (see ADR-0008 in `04-DECISIONS.md`). Limine queries the UEFI
   memory map (`GetMemoryMap`), the ACPI RSDP (via the UEFI configuration
   table, `EFI_ACPI_TABLE_GUID`), and the framebuffer via the Graphics
   Output Protocol (GOP) — none of which this project re-derives.
2. Limine builds its own initial page tables (identity-mapping low memory
   and higher-half-mapping the kernel image), loads the kernel ELF, and
   calls `ExitBootServices` — after this call, UEFI boot services are gone
   for good; only runtime services (e.g. for later reboot/shutdown) remain
   callable, and the OS owns the memory map from here on.
3. Limine jumps to the kernel entry point per the **Limine Boot Protocol**,
   passing a boot-info structure: final memory map, ACPI RSDP pointer,
   framebuffer description, and the physical address of any modules
   (initrd-equivalent). The kernel's entry point and boot-info parsing
   target this protocol directly — see `docs/11-GETTING-STARTED.md` for
   toolchain and template setup.

## 2. CPU bring-up to long mode (BSP)

By the time the kernel's own entry point runs, UEFI has typically already
enabled paging and 64-bit long mode for its own environment, but the kernel
should not assume it inherits a page table layout it trusts. Full explicit
sequence, for auditability and so the AP bring-up path (which starts from
real mode, no firmware help) shares the same logic:

1. Confirm long mode support: `CPUID.80000001H:EDX.LM[bit 29]`.
2. Enable PAE: `CR4.PAE = 1` (bit 5).
3. Load `CR3` with the physical address of a fresh PML4 (identity-mapped
   low region + higher-half kernel mapping, per the layout in
   `01-ARCHITECTURE.md`).
4. Set `IA32_EFER.LME = 1` (bit 8, MSR `0xC0000080`) — long mode *enabled*,
   not yet *active*.
5. Enable paging: `CR0.PG = 1` (bit 31). This triggers actual entry into
   IA-32e mode, but the CPU is in 64-bit *compatibility* submode until CS
   is reloaded from a descriptor with the `L` bit set.
6. Load a minimal flat GDT (paging does protection now, segmentation is
   vestigial) with a 64-bit code segment (`L=1`), far-jump/`retf` into it.
   This is the actual transition into full 64-bit long mode.
7. Load a proper 64-bit GDT + TSS. The TSS's IST (Interrupt Stack Table)
   entries matter here: dedicate **IST1 to `#DF`** (double fault),
   **IST2 to NMI**, **IST3 to `#MC`** (machine check). Handling these
   critical exceptions on a *known-good* dedicated stack, rather than
   whatever stack was active when they fired, is what prevents a stack
   overflow or corruption from cascading into a triple fault instead of a
   diagnosable panic.
8. Load the IDT with handlers for at least: `#DE #DB #NMI #BP #OF #BR #UD
   #NM #DF #TS #NP #SS #GP #PF #MF #AC #MC #XM` — every one of these should
   produce a structured diagnostic dump (registers, faulting address for
   `#PF` via `CR2`, error code) before halting, never a silent reset.

## 3. x2APIC bring-up

1. Check support: `CPUID.01H:ECX.x2APIC[bit 21]`. This project does not
   maintain a permanent xAPIC runtime path — see ADR-0004. If x2APIC is
   genuinely absent (pre-2012-ish hardware), that machine is out of scope.
2. Defensively mask the legacy 8259 PICs via their OCW1 mask registers
   (ports `0x21`/`0xA1`) even though most modern UEFI/ACPI boards don't
   wire them for interrupt delivery — a masked-but-present PIC asserting
   a spurious vector is a classic, easy-to-avoid source of confusing
   early bring-up bugs.
3. Enable x2APIC mode: `IA32_APIC_BASE` MSR (`0x1B`) — set bit 10 (`EXTD`,
   x2APIC enable) and bit 11 (global APIC enable). Order matters: enable
   xAPIC globally first if it wasn't already, then set `EXTD`.
4. From here, all Local APIC register access is via `RDMSR`/`WRMSR`
   instead of MMIO, at MSR address `0x800 + (legacy_offset / 0x10)`:
   - Local APIC ID: MSR `0x802` (read-only in x2APIC, 32-bit ID)
   - Task Priority Register (TPR): MSR `0x808`
   - Spurious Interrupt Vector Register: MSR `0x80F` — set bit 8 to enable
     the APIC and pick a spurious vector (conventionally `0xFF`)
   - Interrupt Command Register (ICR): MSR `0x830` — a single 64-bit MSR
     in x2APIC, versus the split 32+32-bit pair at legacy offsets
     `0x300`/`0x310` in xAPIC. This is where INIT/SIPI IPIs get sent
     (below) and where inter-core signaling for the scheduler goes later.
5. Program the APIC timer for TSC-deadline mode
   (`CPUID.01H:ECX.TSC_Deadline[bit 24]`, then `IA32_TSC_DEADLINE` MSR
   `0x6E0`) in preference to the legacy one-shot/periodic count-down
   register — TSC-deadline gives direct "fire at this absolute TSC value"
   semantics with no periodic-reload race window.

## 4. SMP bring-up (application processors)

APs are not running any code at all until the BSP explicitly starts them —
there is no independent AP boot path, which means the trampoline below
must be hand-built and placed in low memory:

1. Parse the ACPI MADT (Multiple APIC Description Table) to enumerate
   every logical processor and its Local APIC ID, plus IOAPIC entries for
   later interrupt routing.
2. Place a small **real-mode trampoline** in memory below 1 MiB (APs start
   execution in real mode, exactly like a fresh boot, regardless of what
   mode the BSP is in). The trampoline repeats the BSP's long-mode
   transition (steps in §2) for that core, then jumps into a shared
   64-bit AP entry point in the kernel.
3. For each AP's Local APIC ID, via the BSP's ICR (MSR `0x830`):
   - Send **INIT IPI**, wait (~10 ms, per Intel MP spec guidance).
   - Send **SIPI** with vector = `trampoline_physical_address >> 12`
     (the AP starts executing at `vector * 0x1000` in real mode).
   - Send a **second SIPI** — historically required for older CPUs,
     harmless and conventional to keep on modern ones.
4. Each AP signals readiness (e.g. via a shared boot-status cache line)
   before the BSP proceeds to bring up the next one — bringing all cores
   up in parallel without this handshake is a classic source of
   heisenbugs from shared low-memory trampoline state being reused before
   the previous AP finished reading it.

## Diagnostics as a first-class citizen, not an afterthought

Every exception handler installed in step 2.8 above should be treated as
production code, not scaffolding — a machine that resets instead of
panicking with a register dump is undebuggable on real hardware where you
don't have a debugger attached. Serial-port panic output should exist
*before* framebuffer output does, precisely because it works even when GOP
setup itself is the thing that's broken.
