# Implementation Plan — DDR4 Memory Controller

All modules are derived from `docs/ARCHITECTURE.md`.  
Work is organized into phases in dependency order. Pick the first phase with unchecked items.

---

## Phase 1: Foundation — ECC Engine and PHY Interface

These modules have no internal dependencies and are required by all other phases.

### RTL Modules

- [ ] `ecc_engine`: SECDED (72,64) encoder/decoder — 64-bit data, 8 check bits, syndrome generation, single-bit correction, double-bit detection
- [ ] `phy_if`: DDR4 PHY abstraction — command/address serialization, DQ/DQS bi-directional control, half-rate to full-rate conversion, timing model

### Testbenches

- [ ] `ecc_engine`: encode->decode round-trip (no error, 1-bit flip, 2-bit flip), sticky error flag, ECC_ERR_CLR behavior
- [ ] `phy_if`: command issue, write data path, read data capture with valid pulse

### Documentation

- [ ] `docs/ecc_engine.md`: purpose, port table, syndrome table, correction algorithm
- [ ] `docs/phy_if.md`: purpose, port table, timing model, integration notes

---

## Phase 2: Initialization and Mode Registers

Depends on: Phase 1 (`phy_if` for command issuance).

### RTL Modules

- [ ] `mode_reg_fsm`: MRS/EMRS command sequencer; holds MR0–MR6 shadow registers; issues MRS commands via timing FSM interface
- [ ] `init_fsm`: DDR4 power-up sequence FSM (RESET_ASSERT -> CKE_LOW_WAIT -> MRS_MR3..MR0 -> ZQCL -> INIT_DONE); asserts `init_done` on completion

### Testbenches

- [ ] `mode_reg_fsm`: verify each MR write command, spacing (tMRD/tMOD), and shadow register readback
- [ ] `init_fsm`: full power-up sequence, check state transitions, verify `init_done` assertion and timing constraints (tXPR, tZQINIT)

### Documentation

- [ ] `docs/mode_reg_fsm.md`: FSM state diagram, MR shadow register map, port table
- [ ] `docs/init_fsm.md`: FSM state diagram (full init flow), timing requirements, port table

---

## Phase 3: Command Queue and Timing FSM

Depends on: Phase 1 (`phy_if`), Phase 2 (`init_fsm`).

### RTL Modules

- [ ] `cmd_queue`: 32-entry synchronous FIFO for read and write requests; separate RD/WR queues; AXI-side enqueue, scheduler-side dequeue; backpressure to AXI via ready signals
- [ ] `timing_fsm`: JEDEC timing enforcement FSM; per-bank state (IDLE/ACTIVE/READ/WRITE/PRECHARGE); enforces tRCD, tRAS, tRP, tRC, tCCD_S/L, tRRD_S/L, tWTR_S/L, tWR, tRTP, tFAW; all timing values loaded from configurable registers

### Testbenches

- [ ] `cmd_queue`: FIFO full/empty, enqueue/dequeue ordering, backpressure assertion
- [ ] `timing_fsm`: ACT->RD, ACT->WR, WR->RD (tWTR), RD->PRE (tRTP), back-to-back ACT (tRRD/tFAW), same-bank ACT->ACT (tRC) — all timing violations rejected, correct commands accepted

### Documentation

- [ ] `docs/cmd_queue.md`: FIFO depth, port table, backpressure behavior
- [ ] `docs/timing_fsm.md`: per-bank FSM states, timing parameter table, command encoding

---

## Phase 4: Refresh FSM and Command Scheduler

Depends on: Phase 3 (`timing_fsm`, `cmd_queue`).

### RTL Modules

- [ ] `refresh_fsm`: tREFI down-counter, refresh credit counter (up to 8 deferred), precharge-all before REF, tRFC stall, `REFRESH_CTRL` register support
- [ ] `cmd_scheduler`: open-page hit/miss scheduler; row-hit priority; refresh priority over normal commands; address decode (rank/bg/bank/row/col from AXI address); interleaving across bank groups

### Testbenches

- [ ] `refresh_fsm`: periodic REF at tREFI, deferred refresh (credit accumulation), forced refresh at max credits, tRFC stall duration
- [ ] `cmd_scheduler`: row-hit vs row-miss sequencing, refresh preemption, multi-rank interleaving, back-to-back write/read turnaround

### Documentation

- [ ] `docs/refresh_fsm.md`: refresh credit FSM, timing diagram, port table
- [ ] `docs/cmd_scheduler.md`: scheduling policy, address decode map, port table

---

## Phase 5: AXI4 Slave Interface

Depends on: Phase 3 (`cmd_queue`).

### RTL Modules

- [ ] `axi4_slave_if`: AXI4 slave; accepts AW/W/AR channels; issues requests to `cmd_queue`; collects read data and write responses; issues R/B channel responses; supports burst length 1 and 8 (BL8); handles byte strobe for sub-word writes (triggers RMW flag)

### Testbenches

- [ ] `axi4_slave_if`: single read, single write, BL8 read burst, BL8 write burst, byte-masked write, queue-full backpressure, AXI ID matching on response

### Documentation

- [ ] `docs/axi4_slave_if.md`: AXI4 channel summary, supported burst types, RMW flow, port table

---

## Phase 6: Top-Level Integration

Depends on: All previous phases.

### RTL Modules

- [ ] `ddr4_ctrl_top`: wires all sub-modules; connects AXI slave -> cmd_queue -> scheduler -> timing_fsm + refresh_fsm + init_fsm -> phy_if; routes ECC engine inline; exposes `cfg_timing_*` register bank

### Testbenches

- [ ] `ddr4_ctrl_top`: end-to-end write + read (no ECC error), end-to-end read with injected single-bit ECC error (corrected), double-bit ECC error (SLVERR response), refresh preemption during active burst, initialization complete before first transaction

### Documentation

- [ ] `docs/ddr4_ctrl_top.md`: integration diagram, top-level port table cross-reference to ARCHITECTURE.md §3.1, bring-up checklist

---

## JSON Traceability

Each completed module must have a corresponding result file under `results/phase-<phase_name>/`:

- `results/phase-1-foundation/<module>_result.json`
- `results/phase-2-init/<module>_result.json`
- `results/phase-3-cmd-timing/<module>_result.json`
- `results/phase-4-refresh-sched/<module>_result.json`
- `results/phase-5-axi/<module>_result.json`
- `results/phase-6-top/<module>_result.json`

Required JSON fields per result file (see `.github/instructions/results-json.instructions.md`):
`module`, `rtl_done`, `tb_done`, `doc_done`, `simulation_passed`, `coverage_completed`,
`coverage_percentage`, `plan_item_completed`, `error_summary`
