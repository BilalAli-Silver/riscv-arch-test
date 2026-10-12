# ACT Configuration for the OpenHW CVA6 Verilator Testharness

These configurations run the [CVA6](https://github.com/openhwfoundation/cva6) core in its Verilator
testharness (`corev_apu/tb/ariane_testharness.sv`), pinned to CVA6 commit `81245a4`. The CVA6 +
Ara vector configuration is in [`../cva6-ara`](../cva6-ara).

| Config             | ISA                                          | Modes   |
| ------------------ | -------------------------------------------- | ------- |
| `cv32a65x`         | RV32IMC_Zicsr_Zifencei_Zcb_Zba_Zbb_Zbc_Zbs   | M       |
| `cv32a60x`         | RV32IMC_Zicsr_Zifencei_Zcb_Zba_Zbb_Zbc_Zbs   | M       |
| `cv32a6_imac_sv32` | RV32IMAC_Zicsr_Zifencei_Zba_Zbb_Zbc_Zbs_Zkn  | M, S, U |

`cv32a65x` is the 6-stage, single-issue, M-mode-only release configuration described in the
[CV32A65X Design Document](https://docs.openhwfoundation.org/projects/cva6-user-manual/04_cv32a65x/design/design.html).
Its RTL parameters are in `core/include/cv32a65x_config_pkg.sv`. `cv32a60x` is the same ISA with
single issue (`SuperscalarEn=0`) and no PMP entries (`NrPMPEntries=0`); its parameters are in
`core/include/cv32a60x_config_pkg.sv`. `cv32a6_imac_sv32` adds A, B, and Zkn, machine/supervisor/user, and Sv32; its parameters are in `core/include/cv32a6_imac_sv32_config_pkg.sv`. The RTL has 8 writable PMP entries with NAPOT enabled. UDB only allows `NUM_PMP_ENTRIES` of 0, 16, or 64, so the yaml declares 16 entries and sets `NUM_USABLE_PMP_ENTRIES` to 8. Entries 8 through 15 stay zero, which is what the RTL does. The three configurations share `ci.yaml` and `install-cva6.sh`.

## Building and running

```bash
.github/scripts/install-cva6.sh ~/repos/cva6-builds
export PATH=$HOME/repos/cva6-builds/bin:$PATH
export CVA6_SNAPSHOT=$HOME/repos/cva6-builds
make cv32a65x EXCLUDE_EXTENSIONS=Sm,ExceptionsZc,InterruptsSm
make cv32a60x EXCLUDE_EXTENSIONS=Sm,ExceptionsZc,InterruptsSm
make cv32a6_imac_sv32 EXCLUDE_EXTENSIONS=Sm,ExceptionsZc,InterruptsSm
```

The install script clones CVA6, builds the Verilator and Spike versions CVA6's own CI uses (the
testharness links Spike's libraries), verilates one testharness per configuration under
`<install-dir>/cva6/work-ver-<config>/`, and installs the `run-<config>.sh` runners. It needs
`riscv64-unknown-elf-gcc` on `PATH`. The two exports do what `.github/scripts/setup-cva6.sh` does in
CI. The excluded extensions and their reasons are listed in `ci.yaml`.

The testharness halts on a write to `tohost`: 1 is a pass, any other value a fail. The runner passes
the `tohost` address to the simulator and turns the testharness verdict into an `RVCP-SUMMARY`
line. Each simulation uses about 2.5 GB of memory and `make` runs one per CPU, so on machines with
less than 2.5 GB per CPU run the ELFs with fewer jobs (`./run_tests.py -j N`).

With `DEBUG=True`, the runner also writes `work/<config>/logs/<test>.trace.log` (RVFI trace
decoded with `spike-dasm`) and `<test>.fst` (waveform).

A local `cv32a6_imac_sv32` run with the same exclusions was 290 passed and 27 failed. The yaml
is left matching the RTL. The failures are:

`ExceptionsSvZaamo_sv32_exceptions_Zaamo_Smode-00`,
`ExceptionsSvZaamo_sv32_exceptions_Zaamo_Umode-00`,
`ExceptionsSvZaamoSm_sv32_exceptions_Zaamo_mprv_S_Mmode-00`,
`ExceptionsSvZalrsc_sv32_exceptions_Zalrsc_Smode-00`,
`ExceptionsSvZalrsc_sv32_exceptions_Zalrsc_Umode-00`,
`ExceptionsSvZalrscSm_sv32_exceptions_Zalrsc_mprv_S_Mmode-00`,
`ExceptionsSvZalrscSm_sv32_medeleg_Zalrsc_Smode-00`,
`ExceptionsSvZalrscSm_sv32_medeleg_Zalrsc_Umode-00`,
`ExceptionsZalrsc-00`,
`InterruptsS_enable_S-00`,
`InterruptsS_enable_U-00`,
`InterruptsS_priority_sie_S-00`,
`InterruptsS_priority_sie_U-00`,
`InterruptsS_priority_sip_S-00`,
`InterruptsS_priority_sip_U-00`,
`InterruptsS_trigger_S-00`,
`InterruptsS_trigger_U-00`,
`InterruptsS_wfi_timeout_S-00`,
`InterruptsS_wfi_timeout_U-00`,
`PMPSm_cfg_A_tor_zero-00`,
`PMPSm_grain-00`,
`PMPZalrsc_cfg_wr-00`,
`Sv_sv32_spage_Smode-00`,
`Sv_sv32_spage_mstatus_sum_set_Smode-00`,
`Sv_sv32_upage_Umode-00`,
`SvPMP_sv32_pmp_on_pte_Smode-00`,
`SvPMP_sv32_pmp_on_pte_Umode-00`.
