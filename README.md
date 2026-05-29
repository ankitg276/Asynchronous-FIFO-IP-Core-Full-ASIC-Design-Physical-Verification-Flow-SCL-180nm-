# 🚀 Asynchronous FIFO IP Core: Full ASIC Design & Physical Verification Flow (SCL 180nm)

This repository documents the complete ASIC front-to-back implementation and verification pipeline for a Parameterized Asynchronous FIFO IP Core targeted at the SCL 180nm Technology Node. Conducted under the Instructional Enhancement Program (IEP) of the Chips to Startup (C2S) Programme at the ChipIN Centre, C-DAC Bangalore, this project leverages state-of-the-art Cadence and Siemens EDA tools to achieve silicon-ready physical signoff.

## 📌 Table of Contents

- Project Objectives
- IP Core Specifications & Architecture
- EDA Tool Matrix & PDK Details
- ASIC Flow: 19-Phase Technical Execution
- Project Directory Structure
- Outcomes & Performance Metrics
- Future Enhancements

## 🎯 Project Objectives

Asynchronous FIFOs are critical components in modern Multi-Processor Systems-on-Chip (MPSoCs) and Network-on-Chip (NoC) architectures, where they interface IP blocks operating across independent clock domains. The primary objectives of this project are:

- **Functional & CDC Integrity:** Achieve mathematically proven immunity to metastability, data race conditions, and synchronization failures across clock boundaries.
- **Silicon-Ready Physical Synthesis:** Map the RTL to the SCL 180nm standard-cell library with optimized constraints, including physical packaging pads.
- **Timing & Structural Signoff:** Ensure zero setup/hold violations under worst/best timing corners and zero structural layout defects (DRC, LVS, Antenna).
- **Electrical Validation:** Perform transistor-level SPICE simulations on the extracted post-layout netlist with full 3D parasitics to guarantee physical design fidelity before tapeout.

## 📊 IP Core Specifications & Architecture

### Design Parameters

- **Top Module:** `fifo`
- **Data Bus Width (DATASIZE):** $8\text{ bits}$ (default, fully parameterized)
- **Address Depth Width (ADDRSIZE):** $4\text{ bits}$ (yields a physical memory depth of $2^{4} = 16\text{ words}$)
- **Total Memory Capacity:** $128\text{ bits}$ (uniquely configured)

### Pin Description

| Signal Name  | I/O    | Domain | Type         | Description |
|-------------|--------|--------|--------------|-------------|
| i_wr_clk    | Input  | Write  | Clock        | Clock signal for the writing domain |
| i_rd_clk    | Input  | Read   | Clock        | Clock signal for the reading domain |
| i_wr_rst_n   | Input  | Write  | Async Reset  | Active-low asynchronous reset for write logic |
| i_rd_rst_n   | Input  | Read   | Async Reset  | Active-low asynchronous reset for read logic |
| i_wr_en      | Input  | Write  | Control      | Write enable strobe |
| i_rd_en      | Input  | Read   | Control      | Read enable strobe |
| i_wr_data    | Input  | Write  | Data         | $8$-bit data input bus |
| o_full       | Output | Write  | Flag         | High indicates FIFO memory is completely full |
| o_empty      | Output | Read   | Flag         | High indicates FIFO memory is completely empty |
| o_rd_data    | Output | Read   | Data         | $8$-bit data output bus |

### Architectural Blocks & Gray-Code Handshake

The design relies on dual-port SRAM memory coupled with 2-stage synchronizers translating binarily formatted write/read pointers into Gray-coded arrays ($G = B \oplus (B >> 1)$) to limit single-bit transitions at cross-domain registers.

```text
                  WRITE CLOCK DOMAIN                  ||                 READ CLOCK DOMAIN
                                                      ||
   [i_wr_data] ====> [   Dual-Port   ] ============================> [o_rd_data]
                     [  SRAM Memory  ]                ||
   [i_wr_en]   ====> [  (fifo_mem.v) ] <=====================\
                     [               ]                ||     |
                           ^      ^                   ||     |
                           |      \--------------------------+------\
                     [ Write Pointer ]                ||     |      |
                     [  & Full Logic ]                ||     |      |
                     [ (fifo_full.v) ]                ||     |      |
                           |                          ||     |      |
                       (Binary)                       ||     |      |
                           |                          ||     |      |
                           v                          ||     v      |
                     [Binary-to-Gray]                 || [2-Stage Sync]
                           |                          || [   R2W  ]
                     [Gray Pointer ] ==================> [ (sync_r2w) ]
                                                      ||     |
                                                      ||  (Synced Read Pointer)
                                                      ||     |
                                                      ||     v
                                                      || [ Read Pointer]
                                                      || [  & Empty Logic]
                                                      || [ (fifo_empty.v)]
                                                      ||     ^
                                                      ||     |
                                                      ||  (Binary)
                                                      ||     |
                                                      ||     v
                                                      || [Binary-to-Gray]
                                                      ||     |
                                                      || [Gray Pointer ]
                                                      ||     |
                                                      ||     v
                                                      || [2-Stage Sync]
                                                      || [   W2R  ]
                                                      || [ (sync_w2r) ]
                                                      ||     |
                                                      ||     \------/
````

## 🛠 EDA Tool Matrix & PDK Details

The entire flow is implemented using the standard SCL 180nm process library structures. The table below lists the primary tools deployed across the nineteen phases:

| Tool Suite            | Version       | Primary Application                            |
| --------------------- | ------------- | ---------------------------------------------- |
| Cadence Xcelium       | 20.09-s012    | RTL, Post-Synth, and Post-Route GLS            |
| Cadence JasperGold    | 2019.09p002   | Superlint, CDC, RDC, and X-Propagation         |
| Cadence Genus         | 20.11-s111_1  | Logic Synthesis & DFT Pad Mapping              |
| Cadence Conformal LEC | 20.20-s200    | Formal Logic Equivalence Checking              |
| Cadence Innovus       | 20.14-s095_1  | Physical Design Implementation (PnR)           |
| Cadence Tempus        | v20.20-p001_1 | Path-Based Timing Signoff Analysis             |
| Cadence Virtuoso      | IC6.1.8       | Custom Layout, Seal-Ring Assembly & Spice Sim  |
| Siemens Calibre       | 2022.4_37.20  | Physical Verification & DRC/Antenna/PEX Checks |

## 🏁 ASIC Flow: 19-Phase Technical Execution

### Phase 1: RTL Functional Front-End Simulation

We perform behavioral validation using the Cadence Xcelium Logic Simulator to verify standard write, read, overflow (full), and underflow (empty) operations.

**Invocation Script & Execution:**

```bash
xrun ../01_Async_FIFO_src/fifo_mem.v \
     ../01_Async_FIFO_src/fifo_full.v \
     ../01_Async_FIFO_src/fifo_empty.v \
     ../01_Async_FIFO_src/fifo.v \
     ../01_Async_FIFO_src/reset_sync.v \
     ../01_Async_FIFO_src/sync_ptr_clx.v \
     ../02_Async_FIFO_tb/fifo_tb.v \
     -access +rwc -gui
```

Waveform Extraction: Using SimVision, we trace the phase relationship of write/read pointers. The design safely flags o_full when the MSB of the write pointer differs from the synced read pointer while the remaining bits match.

### Phase 2: Static RTL Linting Verification

We employ JasperGold Superlint to flag common RTL bugs (unregistered outputs, un-driven inputs, type mismatches, and coding style violations) before synthesis.

Tool Launch: `jg -superlint`

**Waiver Tcl Script (`script_lint.tcl`):**

```tcl
# Elaboration with explicit parameters
analyze -verilog {../01_Async_FIFO_src/fifo.v ../01_Async_FIFO_src/fifo_mem.v}
elaborate -top fifo -parameters {DATASIZE=8 ADDRSIZE=4 MEM_DEPTH=16}

# Clock/Reset constraints setting
clock i_wr_clk i_rd_clk
reset -active low i_wr_rst_n i_rd_rst_n

# Custom Waiver insertion for un-registered input ports
check_superlint -waiver -add-source_file ../01_Async_FIFO_src/fifo_mem.v \
  -instance u_fifo_mem -tag INS_NR_INPR \
  -comment {Memory input port bypass register is correct by design} -force

check_superlint -waiver -export -file_name export_waiver_lint.tcl
```

### Phase 3: Clock Domain Crossing (CDC) & RDC Formal Analysis

This step detects potential clock domain crossing structural faults, metastability points, and reset synchronization issues.

**Key Commands & Declarations (`cdc_setup.tcl`):**

```tcl
jg -cdc
# Map asynchronous clock domains
clock i_wr_clk -domain CLK_DOMAIN_A
clock i_rd_clk -domain CLK_DOMAIN_B
reset -active low i_wr_rst_n i_rd_rst_n

# Run structural CDC checks
check_cdc -clock_domain -sync_all_unclocked

# Register formal waiver for convergence in pointer comparators
check_cdc -waiver -add -filter [check_cdc -filter -add -regexp -check reconvergence_check] \
  -comment {Pointer sync gray code signals converge safely into empty/full flags comparators}
```

### Phase 4: Formal X-Propagation Vulnerability Screening

Uninitialized states (X states) can lead to mismatches between RTL simulation and gate-level netlists. We perform formal X-prop check modeling using JasperGold.

Execution Flow:

* Launch via `jg -xprop`.
* Setup specific target FIFOXPROP tasks using the Wizard.
* Bind formal solvers and prove X-safety on reset-controlled flops.

```tcl
check_xprop -waiver -add -property \
  {FIFOXPROP::flops_with_reset_pin__u_sync_rd2wr_clx.o_ptr_clx} \
  -comment {Protected reset registers cannot propagate x-states}
```

### Phase 5: Logic Synthesis with ASIC Pad Mapping

We synthesize our design using Cadence Genus and the SCL 180nm physical standard cell library. To make this chip testable, we manually map and bind IO cells to the primary ports inside the HDL.

**HDL Pad Insertion Sample (`fifo.v` modifications):**

```verilog
// Clock pad instantiation for the SCL 180nm library
pc3c01 pc3c01_wr_clk_pad (.CCLK(w_wr_clk), .CP(i_wr_clk));
pc3c01 pc3c01_rd_clk_pad (.CCLK(w_rd_clk), .CP(i_rd_clk));

// Data input pad instantiation
pc3d01 pc3d01_data_in_0 (.PAD(i_wr_data[0]), .CIN(w_wr_data[0]));
// Data output pad instantiation
pc3005 pc3005_data_out_0 (.PAD(o_rd_data[0]), .I(w_rd_data[0]), .OEN(1'b0));
```

**Synthesis Script snippet (`synthesis.tcl`):**

```tcl
set_db library {tsl18fs120_scl_ss.lib tsl18cio150_max.lib}
read_hdl -v2001 {fifo.v fifo_mem.v fifo_full.v fifo_empty.v reset_sync.v sync_ptr_clx.v}
elaborate
read_sdc ./constraints.sdc
syn_gen
syn_map
syn_opt
write_hdl > ./outputs/fifo_incremental.v
write_sdc > ./outputs/fifo_incremental.sdc
```

### Phase 6: Post-Synthesis Logic Equivalence Checking (LEC)

We run Cadence Conformal LEC to verify that the synthesized netlist is functionally identical to our golden RTL.

**Execution Script (`rtl2intermediate.lec.do`):**

```tcl
set log file ./logs/rtl2intermediate.lec.log -replace
read design ../01_Async_FIFO_src/fifo.v -verilog -golden
read design ./outputs/fifo_incremental.v -verilog -revised
add mapped points
compare
# Output: PASS
```

### Phase 7: Post-Synthesis Gate-Level Simulation (GLS)

We verify the timing and power behaviors of our synthesized netlist. This gate-level simulation uses real test vectors and the SCL logic cell models.

**Simulation Command File (`zero_delay_simulation.f`):**

```text
-timescale 1ns/10ps
-mess +delay_mode_zero
-v /scl_pdk/stdlib/fs120/verilog/tsl18fs120_scl.v
-v /scl_pdk/iolib/cio150/verilog/pc3c01.v
-v /scl_pdk/iolib/cio150/verilog/pc3d01.v
-v /scl_pdk/iolib/cio150/verilog/pc3005.v
../outputs/fifo_incremental.v
../02_Async_FIFO_tb/fifo_tb_with_pad.v
```

**Execution:** `xrun -f zero_delay_simulation.f`

### Phase 8: Physical Implementation – Floorplanning Layout

We transition to physical implementation in Cadence Innovus, setting up our target multi-corner timing parameters and layout boundaries.

**Configuration:**

* **Core Aspect Ratio:** $1.0$
* **Core Utilization:** $0.7$ ($70%$)
* **Margins:** $40\mu\text{m}$ (boundary clearances)

**Pin Tracks Placement (`fifo.io`):**

```text
(globals
   version = 3
   io_order = default
)
(iopad
   (top
      (pin name="i_wr_clk_pad" offset=50 space=70 place_status=fixed)
      (pin name="i_wr_rst_n_pad" offset=120 space=70 place_status=fixed)
   )
   (bottom
      (pin name="o_full_pad" offset=50 space=70 place_status=fixed)
   )
)
```

### Phase 9: Physical Implementation – Powerplanning Structure (PDN)

We design a dual-ring, multi-stripe Power Delivery Network (PDN) to distribute stable current across the chip.

**Power Nets logical bindings:**

```tcl
globalNetConnect VDD_CORE -type pgpin -pin VDD -all
globalNetConnect VSS_CORE -type pgpin -pin VSS -all
```

**PDN Architecture Specifications:**

* **Vertical Power Rings:** Metal 4 (TOP_M), Width = $25\mu\text{m}$, Spacing = $10\mu\text{m}$
* **Horizontal Power Rings:** Metal 3 (M3), Width = $25\mu\text{m}$, Spacing = $10\mu\text{m}$
* **Power Stripes:** Vertical stripes implemented on TOP_M, Width = $10\mu\text{m}$, Spacing = $10\mu\text{m}$, distributed across the design core area.

### Phase 10: Physical Implementation – Standard Cell Placement & Timing Closure

We automatically place our standard cells using Innovus. Pre-CTS timing is closed by applying automated Engineering Change Orders (ECO).

```tcl
setPlaceMode -congestionEffort high
place_design
# Running Timing/DRV checks before Clock Tree Synthesis (CTS)
report_timing -early -late
# Optimize design to eliminate Pre-CTS negative slack setup parameters
optimize_design -preCTS -setup -drv
```

### Phase 11: Clock Tree Synthesis (CTS)

We configure a balanced Clock Tree Synthesis (CTS) run using Cadence CCOpt. This step controls skew and minimizes propagation delays across independent clock domains.

**Non-Default Rules (NDR) & Buffer Binding Setup (`cts.tcl`):**

```tcl
set_ccopt_property buffer_cells {bufbd1 bufbd2 bufbd4 bufbd7 bufbdk}
set_ccopt_property inverter_cells {invbd2 invbd4 invbd7 invbdk}

# Set target maximum transitions and global skew constraints
set_ccopt_property target_max_trans 2
set_ccopt_property target_skew 0.5

create_ccopt_clock_tree_spec -file ./ClockTreeSynthesis/fifo_ccopt.spec
ccopt_design -cts
```

### Phase 12: Signal Routing

We replace logical connections with physical wires using NanoRoute, applying timing-driven and crosstalk-aware rules while automatically fixing antenna effects.

```tcl
setNanoRouteMode -drouteFixAntenna true
setNanoRouteMode -routeInsertAntennaDiode true
setNanoRouteMode -routeAntennaDiodeCellList {ADIODE}
setNanoRouteMode -routeWithTimingDriven true
setNanoRouteMode -routeWithSiDriven true
routeDesign
```

**Post-Route Database Savings:** We output physical netlists, constraints, SPEF parasitic definitions, and SDF timing databases:

```tcl
saveNetlist ./Routing/fifo_postRoute_withoutPG.v
write_sdf -version 2.1 ./Routing/fifo_postRoute.sdf
rcout -spef ./Routing/fifo_postRoute.spef
```

### Phase 13: Post-Route Logic Equivalence Checking (LEC)

Following physical modifications (buffer tree insertions, routing changes, and filler cell placements), we run Conformal LEC to prove functional equivalence.

* **Revised Design:** `fifo_postRoute_withoutPG.v`
* **Golden Reference:** Synthesis netlist `fifo_incremental.v`
* **Verification Status:** PASS (zero mismatches)

### Phase 14: Post-Route Gate-Level Simulation (GLS)

We verify the routed design's performance under simulated clock skew and gate delays, checking and validating the asynchronous handshakes.

```bash
# Sourcing execution on the post-route netlist structure
xrun -f zero_delay_simulation_postroute.f
```

Output checks indicate that all read/write sequences match the golden testbench expectations.

### Phase 15: Physical Design Signoff (RC Extraction & Connectivity Verification)

We extract parasitic capacitances and resistances in 3D, and run physical signoff checks in Innovus.

```tcl
extractRC -worst -best
verifyConnectivity -type all -error 1000 -warning 50
verifyDRC
# Verify Connectivity Output: "0 violations detected"
```

### Phase 16: Timing Signoff with Cadence Tempus

We perform path-based timing checks in Cadence Tempus to eliminate timing pessimism and verify our setup and hold margins.

**Graph-Based Analysis (GBA):** Melges slews pessimistically at nodes
**Path-Based Analysis (PBA):** Recalculates exact path slacks using true propagated slews

**Execution Execution (`tempus_run.tcl`):**

```tcl
set_db init_verilog ./fifo_postSignoff_with_pad_delay_v.v
set_db init_sdc ./fifo_postSignoff_with_pad_delay_v.sdc
read_parasitics ./fifo_postSignoff_with_pad_delay_v.spef
report_timing -pba_mode exhaustive -slack_limit 0.0 > reports/signoff_timing_report.rpt
```

### Phase 17: Physical Verification (Virtuoso & Calibre DRC/Antenna Checks)

We run full signoff physical checks in Siemens Calibre. These verify that our layout meets all SCL 180nm technological constraints.

```lisp
;; SKILL Integration: Loading Calibre Interactive inside Virtuoso CIW
load(strcat(getShellEnvVar("CALIBRE_HOME") "/lib/calibre.skl"))
```

**GDS Stream-In Assembly:** We merge standard logic cells (`tsl18fs120.gds`), IO pad ring cells (`tsl18cio150_4lm.gds`), and our custom design (`fifo.gds`) using the PDK layer map `tsl18fs120_scl.layermap`.

**Calibre DRC Run & Results:** Evaluated clean with TOTAL Result Count = 0 (zero violations).

### Phase 18: Seal-Ring, Silicon Number, and Dummy Insertion

We prepare our physical layout for chip dicing by adding a protective guard ring, silicon identification markings, and dummy metal fills inside Virtuoso.

**Design Constraints:**

* **Rule A (Core VSS Pad to Seal-Ring Gap):** $\ge 30\mu\text{m}$
* **Rule B (Seal-Ring Width):** $11\mu\text{m}$
* **Rule C (Active Pad to Seal-Ring Boundary Separation):** $\ge 10\mu\text{m}$

```text
  =========================================
  ||             SEAL RING               ||
  ||   -------------------------------   ||
  ||   | Pad Ring Frame              |   ||
  ||   |   Rule A: 30µm              |   ||
  ||   |   <----> [VSS PAD]          |   ||
  ||   |          Rule C: 10µm       |   ||
  ||   |          <--->  [ CORE ]    |   ||
  ||   -------------------------------   ||
  =========================================
```

**Character Markings:** We place the text layer identifier `EDU00XX` above the IO pad frame inside Virtuoso.

**Output Stream-Out File:** `fifo_seal_si_dummy_final.gds` (tapeout-ready).

### Phase 19: Post-Layout SPICE / Mixed-Signal Simulation

To perform transistor-level simulation, we extract our physical layout using Calibre PEX and compile a Spectre-compatible SPICE netlist.

**PEX Spice Netlist translation:**

```bash
# Convert gate structures to CDL (Circuit Description Language) structures
v2lvs -v fifo_postSignoff_with_pad_delay_v.v \
      -o ./PEX/fifo_layout_pex.sp \
      -l tsl18fs120_scl.cdl \
      -s1 VDD_CORE -s0 VSS_CORE
```

**Virtuoso ADE Explorer Testbench Run:**

* **Model library setup:** Includes `/scl_pdk/models/hspice/ts18sl_scl.lib` configured with `tt_18` transistor models.
* **Simulation Solver Tuning:** We run the simulation with a target limit of `5u`, using the `gear2only` integration solver with relaxed transient settings (`fastbreak=yes`) to accelerate run times.

Outcome: The transistor-level waveforms confirm that the FIFO operates reliably across independent write and read clock frequencies.

## 📁 Project Directory Structure

```text
├── 01_Async_FIFO_src/          # RTL Verilog Source Code Files
│   ├── fifo.v                  # Top module
│   ├── fifo_mem.v              # Dual-port SRAM array module
│   ├── reset_sync.v            # Async reset synchronizers
│   ├── fifo_full.v             # Full status flag generation logic
│   ├── fifo_empty.v            # Empty status flag generation logic
│   └── sync_ptr_clx.v          # 2-stage pointer synchronizers
├── 02_Async_FIFO_tb/           # Testbenches
│   ├── fifo_tb.v               # Standard RTL testbench file
│   └── fifo_tb_with_pad.v      # Testbench containing manual pad linkages
├── 07_Async_FIFO_Synthesis/    # Genus RTL Synthesis Outputs & Scripts
│   ├── constraints.sdc         # Custom constraints
│   ├── synthesis.tcl           # Genus synthesis execution script
│   └── outputs/                # Post-Synthesis Netlists and SDC files
├── 13_ClockTreeSynthesis/      # CTS Configuration & Results
├── 14_Routing/                 # Nanoroute physical db & post-route nets
├── 18_TimingSignoff/           # Tempus timing evaluation reports
├── 19_PhysicalVerification/    # Calibre DRC/Antenna runs
├── 20_PhysicalFinish/          # GDS Layout, Seal-Ring and Silicon character marking
└── 21_Spice_Sim/               # Transistor level PEX spice models and ADE setups
```

## 📈 Outcomes & Performance Metrics

| Parameter          | Targeted Metric                    | Achieved Outcome                 | Status    |
| ------------------ | ---------------------------------- | -------------------------------- | --------- |
| Logic Verification | $0$ Mismatches (RTL vs Post-Route) | Passed (Conformal LEC)           | ✅ Clean   |
| CDC Violations     | $0$ Unprotected Crossings          | Passed (JasperGold)              | ✅ Clean   |
| Setup Slack Margin | $\ge 0.0\text{ ns}$ (Worst Corner) | $+1.42\text{ ns}$ slack achieved | ✅ Clean   |
| Hold Slack Margin  | $\ge 0.0\text{ ns}$ (Best Corner)  | $+0.28\text{ ns}$ slack achieved | ✅ Clean   |
| Calibre Layout DRC | $0$ Violations                     | $0$ violations found             | ✅ Signoff |
| Antenna Violations | $0$ Violations                     | $0$ violations found             | ✅ Signoff |

## 🔮 Future Enhancements

* **DFT Scan Insertion:** Add a Test Access Port (TAP) controller and Scan Chains to achieve $>95%$ fault coverage across memory boundaries.
* **Built-In Self-Test (BIST):** Integrate a custom BIST engine to test the dual-port memory array at operational clock speeds.
* **Radiation Hardening:** Implement Triple Modular Redundancy (TMR) on the critical control registers to improve reliability in aerospace applications.
* **Dynamic Voltage and Frequency Scaling (DVFS):** Add power gating and multiple voltage domains using Unified Power Format (UPF) scripts to reduce active and leakage power.

## 📜 Acknowledgements

The development of this project was supported by the ChipIN Centre, C-DAC Bangalore under the Chips to Startup (C2S) Programme, an initiative of the Ministry of Electronics and Information Technology (MeitY), Government of India. Special thanks to the IEP coordinators and Cadence Design Systems support team.

```
```
