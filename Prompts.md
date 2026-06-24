# AI Prompt Log — Mixed-Signal Physical Design Flow
**Author:** Bhuvanachandra Kusuma  
**AI Tool Used:** Claude (Anthropic's AI assistant) — claude.ai  
**Repo:** [praharshapm/vsdmixedsignalflow](https://github.com/praharshapm/vsdmixedsignalflow)  
**Flow:** OpenLane v1.0.2 + SKY130A PDK + AMUX2_3V analog macro

---

> **How to read this file:**  
> Each prompt block shows the exact question/instruction given to the AI, the AI tool used, what the AI generated, what was manually verified or edited, and the outcome. Week 1 prompts are complete. Week 2 prompts are added as the flow progresses.

---

## WEEK 1 — Repo Understanding, Setup, Flow Execution, Error Fixing

---

### Prompt W1-01 — Repo Comprehension

**AI Tool:** Claude (claude.ai)  
**Purpose:** Understand the structure and goal of the reference repository before touching any files.

**Prompt given to AI:**
```
I have a GitHub repo: https://github.com/praharshapm/vsdmixedsignalflow
It is about mixed-signal physical design using OpenLane and SKY130.
Please explain:
1. What is the overall goal of this repo?
2. What does each folder/file do (Verilog/, LEF/, LIB/, openlane/)?
3. What is the data flow — from inputs to final GDS?
4. What tools are required to run this?
5. What is AMUX2_3V and how is it used?
```

**AI Output Summary:**
- Repo implements RTL-to-GDS for `design_mux` (digital SPI logic) integrating analog macro `AMUX2_3V` (2:1 analog MUX)
- Data flow: Verilog RTL + LEF (physical abstract) + LIB (timing model) → OpenLane → GDS
- `AMUX2_3V` is treated as a blackbox during synthesis; its physical footprint comes from the LEF file
- Tools needed: Docker, OpenLane, SKY130 PDK, Magic

**Manual Verification:** Cross-checked against actual repo README and folder structure on GitHub. Confirmed accurate.

**Outcome:** ✅ Clear understanding of repo structure and flow established before any setup work.

---

### Prompt W1-02 — Tool Installation

**AI Tool:** Claude (claude.ai)  
**Purpose:** Generate step-by-step commands to install Docker, OpenLane v1, and SKY130 PDK on WSL Ubuntu.

**Prompt given to AI:**
```
I am on WSL Ubuntu (hostname BhuvanLAPTOP, user bhuvan).
I want to install:
1. Docker (using get-docker.sh method)
2. OpenLane v1 from https://github.com/The-OpenROAD-Project/OpenLane
3. SKY130A PDK using the ciel tool
Give me the exact shell commands to install and verify each one.
```

**AI Output (key commands generated):**
```bash
# Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# OpenLane
git clone https://github.com/The-OpenROAD-Project/OpenLane.git ~/OpenLane
cd ~/OpenLane
make pull-openlane

# SKY130A PDK via ciel
pip install ciel
ciel get sky130A

# Verify
cd ~/OpenLane
make test
```

**Manual Verification:** Ran all commands. `make test` completed with "Basic test passed". OpenLane v1.0.2 confirmed.

**Manual Edits:** None required — commands worked as generated.

**Outcome:** ✅ Full toolchain installed and verified.

---

### Prompt W1-03 — LEF/LIB/Verilog Role Explanation

**AI Tool:** Claude (claude.ai)  
**Purpose:** Understand what each input file format does in a mixed-signal PnR flow before creating/editing them.

**Prompt given to AI:**
```
In an OpenLane mixed-signal flow using SKY130, explain the role of:
1. LEF file — what fields matter (CLASS, SITE, SIZE, PIN, OBS, DIRECTION, USE)?
2. LIB file — what does a minimal timing model for an analog macro need?
3. Blackbox Verilog — why is it needed and what should it contain?
Explain each in the context of integrating an analog macro called AMUX2_3V
(a 2:1 analog MUX with pins I0, I1, out, select) into a digital OpenLane flow.
```

**AI Output Summary:**
- **LEF:** Describes physical footprint. `CLASS MACRO`, `SITE unithddbl` (dual-height), `SIZE` (width × height in µm), `PIN` blocks with `DIRECTION INPUT/OUTPUT` and `USE SIGNAL/POWER`, `OBS` layer for routing blockage
- **LIB:** Needs cell name, area, pin definitions with direction and function, timing threshold attributes (`slew_lower_threshold_pct_fall`, etc.), and capacitance values. Minimal model is enough — analog timing is not modeled precisely
- **Blackbox Verilog:** Used by Yosys during synthesis to recognize the module interface without synthesizing internal logic. Must use `(* blackbox *)` attribute in modern Verilator/OpenLane v1

**Manual Verification:** Cross-checked against actual `AMUX2_3V.lef` and `AMUX2_3V.lib` files in the reference repo. AI explanation matched.

**Outcome:** ✅ Clear mental model of all three file formats established.

---

### Prompt W1-04 — Design Directory and Config File Setup

**AI Tool:** Claude (claude.ai)  
**Purpose:** Create the `design_mux` project directory structure and generate `config.json` for OpenLane v1.

**Prompt given to AI:**
```
I am setting up an OpenLane v1 design called "design_mux" at ~/OpenLane/designs/design_mux/.
The design has:
- Digital RTL: design_mux.v, raven_spi.v, spi_slave.v (in src/)
- Analog macro blackbox Verilog: AMUX2_3V.v (in bb/)
- Macro LEF: AMUX2_3V.lef (in src/lef/)
- Macro LIB: AMUX2_3V.lib (in src/lib/)

Generate:
1. Shell commands to create the directory structure and copy files from /tmp/vsdref/
2. A config.json for OpenLane v1 with CLOCK_PORT=select, CLOCK_PERIOD=10,
   FP_CORE_UTIL=20, PL_TARGET_DENSITY=0.4, and correct EXTRA_LEFS/EXTRA_LIBS paths.
```

**AI Output (config.json generated):**
```json
{
  "DESIGN_NAME": "design_mux",
  "VERILOG_FILES": "dir::src/design_mux.v",
  "VERILOG_FILES_BLACKBOX": "dir::bb/AMUX2_3V.v",
  "CLOCK_PORT": "select",
  "CLOCK_PERIOD": 10,
  "FP_CORE_UTIL": 20,
  "PL_TARGET_DENSITY": 0.4,
  "EXTRA_LEFS": "dir::src/lef/AMUX2_3V.lef",
  "EXTRA_LIBS": "dir::src/lib/AMUX2_3V.lib"
}
```

**Manual Edits:**
- Initial AI draft used `"VERILOG_FILES": "dir::src/*.v"` (glob pattern) — changed to explicit `design_mux.v` only after discovering this caused duplicate module errors (see W1-06)

**Outcome:** ✅ Design directory and config created successfully.

---

### Prompt W1-05 — First Flow Run and Error Analysis

**AI Tool:** Claude (claude.ai)  
**Purpose:** Run OpenLane and diagnose the first batch of errors.

**Prompt given to AI:**
```
I ran: make quick_run QUICK_RUN_DESIGN=design_mux
OpenLane failed during Verilog linting with these errors:
[ERROR] ...AMUX2_3V.v:3: Unsupported: Verilog 1995 style port declaration
[ERROR] ...AMUX2_3V.v:5: wire 'I0' declared as input port
[ERROR] ...AMUX2_3V.v: Cannot assign to wire 'out' via initial block
Explain what each error means and give me the fix.
```

**AI Output — Root Cause:**
The original `AMUX2_3V.v` from the reference repo used Verilog-1995-style port declarations (port list in module header, then separate `input`/`output`/`wire` declarations below), plus an invalid `initial` block assigning to a `wire`. This was valid in older Verilator but rejected by the modern version bundled with OpenLane v1.0.2.

**AI Output — Fix:**
```verilog
(* blackbox *)
module AMUX2_3V (
   input  I0,
   input  I1,
   output out,
   input  select
);
endmodule
```

**Manual Verification:** Confirmed the original file structure. Applied fix. Re-ran flow — linting passed.

**Outcome:** ✅ Error #1 resolved. Flow advanced past Verilog linting.

---

### Prompt W1-06 — Duplicate Module Definition Error

**AI Tool:** Claude (claude.ai)  
**Purpose:** Fix Yosys "re-definition of module" error during synthesis.

**Prompt given to AI:**
```
After fixing the blackbox stub, synthesis now fails with:
[ERROR] Yosys: Re-definition of module 'raven_spi'
[ERROR] Yosys: Re-definition of module 'spi_slave'
My config.json has: "VERILOG_FILES": "dir::src/*.v"
design_mux.v already `include`s raven_spi.v and spi_slave.v internally.
What is causing this and how do I fix it?
```

**AI Output — Root Cause:**
The `dir::src/*.v` glob pattern passed all `.v` files to Yosys. Since `design_mux.v` already has `` `include "raven_spi.v" `` internally, those modules were being parsed twice — once via include, once via the glob.

**AI Output — Fix:**
```json
"VERILOG_FILES": "dir::src/design_mux.v"
```
List only the top-level file explicitly; let `include` directives handle sub-modules.

**Manual Verification:** Applied fix to `config.json`. Re-ran — synthesis completed without redefinition errors.

**Outcome:** ✅ Error #2 resolved. Synthesis completed.

---

### Prompt W1-07 — Port Typo Fix

**AI Tool:** Claude (claude.ai)  
**Purpose:** Fix an undriven net warning that caused synthesis to produce incorrect netlist.

**Prompt given to AI:**
```
Synthesis completed but I see this warning:
[WARNING] Yosys: Module 'design_mux', net 'IO' is not connected.
Looking at design_mux.v, I see the AMUX2_3V instantiation has: .I0(IO)
Is this a bug? The macro pin is I0 (letter I, zero) — is IO (letter I, letter O) a typo?
```

**AI Output:** Yes — `.I0(IO)` connects macro pin `I0` to a net called `IO` (I-O), which doesn't exist in the module. The correct connection is `.I0(I0)` (connecting pin `I0` to signal `I0`). This is a one-character typo (letter O vs digit 0) in the original repo's Verilog.

**Manual Verification:** Opened `design_mux.v`, confirmed the typo on the AMUX2_3V instantiation line. Fixed to `.I0(I0)`.

**Outcome:** ✅ Error #3 resolved. Net connectivity warning eliminated.

---

### Prompt W1-08 — Missing LIB Timing Attributes

**AI Tool:** Claude (claude.ai)  
**Purpose:** Fix OpenSTA timing analysis failure caused by incomplete `.lib` file.

**Prompt given to AI:**
```
The flow fails during timing analysis with:
[ERROR STA-0037] Liberty attribute 'slew_lower_threshold_pct_fall' not found in cell AMUX2_3V
The AMUX2_3V.lib file from the reference repo does not have timing threshold attributes.
What attributes are missing and what values should I use for SKY130?
```

**AI Output — Missing Attributes (8 total):**
```
slew_lower_threshold_pct_fall  : 30.0;
slew_upper_threshold_pct_fall  : 70.0;
slew_lower_threshold_pct_rise  : 30.0;
slew_upper_threshold_pct_rise  : 70.0;
input_threshold_pct_fall       : 50.0;
input_threshold_pct_rise       : 50.0;
output_threshold_pct_fall      : 50.0;
output_threshold_pct_rise      : 50.0;
```
These are standard SKY130 `fd_sc_hd` threshold percentages and should be added inside the `library()` block of `AMUX2_3V.lib`.

**Manual Verification:** Opened `AMUX2_3V.lib`, confirmed all 8 attributes were absent. Added them under the `library(AMUX2_3V)` opening block. Re-ran — OpenSTA passed.

**Outcome:** ✅ Error #4 resolved. Timing analysis completed.

---

### Prompt W1-09 — Resizer Grid-Paint Conflict

**AI Tool:** Claude (claude.ai)  
**Purpose:** Fix detailed placement legalization failure near the analog macro.

**Prompt given to AI:**
```
The flow fails at detailed placement with:
[ERROR DPL-0041] Placement grid paint conflict near macro AMUX2_3V
The macro is a dual-height cell and the standard cell rows are single-height.
How do I work around this in OpenLane v1 config.json?
```

**AI Output:** The OpenROAD resizer tries to optimize and re-legalize cells near the macro boundary, which causes grid conflicts with the dual-height macro footprint. Workaround: disable resizer optimizations for this design.

```json
"PL_RESIZER_DESIGN_OPTIMIZATIONS": 0
```

**Manual Verification:** Added to `config.json`. Re-ran placement — DPL-0041 error gone, flow advanced to PDN generation.

**Outcome:** ✅ Error #5 resolved. Flow reached Step 9 (PDN generation).

---

### Prompt W1-10 — PDN and Layout Verification

**AI Tool:** Claude (claude.ai)  
**Purpose:** Understand what a successful PDN output looks like and how to view it in Magic.

**Prompt given to AI:**
```
OpenLane successfully completed PDN generation (Step 9) for design_mux.
The run directory is at ~/OpenLane/runs/design_mux/results/
What output files should exist at this stage, and how do I open the layout
in Magic with SKY130A tech files to verify the PDN and macro placement visually?
```

**AI Output:**
```bash
# Files to check after PDN:
# runs/design_mux/results/floorplan/design_mux.floorplan.def
# runs/design_mux/results/placement/design_mux.placement.def

# Open in Magic:
cd ~/OpenLane
magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech \
  runs/design_mux/results/placement/design_mux.placement.def
```

**Manual Verification:** Ran the Magic command. Layout opened showing AMUX2_3V macro, PDN power rails (met1/met4), and placed standard cells.

**Outcome:** ✅ PDN and placement layout verified in Magic. Screenshot captured for report.

---

### Prompt W1-11 — Macro Placement Boundary Issue (Root Cause)

**AI Tool:** Claude (claude.ai)  
**Purpose:** Root-cause why detailed placement (Step 11) was blocked.

**Prompt given to AI:**
```
OpenLane fails at detailed placement (Step 11) with:
[ERROR] Macro AMUX2_3V placed at Y=124.432um exceeds floorplan boundary of 122.4um.
The floorplan is 122.4um tall. The automatic macro placer put the macro outside bounds.
What is the root cause and what is the correct fix for Week 2?
```

**AI Output — Root Cause:** OpenROAD's automatic macro placer does not always respect floorplan boundaries when the macro is large relative to the core area. The AMUX2_3V (dual-height, ~5.44µm tall) was placed at Y=124.432µm, which puts its top edge outside the 122.4µm boundary.

**AI Output — Week 2 Fix Strategy:**
Use manual macro placement via a `macro_placement.cfg` file to place the macro explicitly at a known in-bounds, grid-aligned location:
```
# macro_placement.cfg format:
# <macro_name> <X> <Y> <orientation>
AMUX2_3V 10.0 10.0 N
```
Then reference it in `config.json`:
```json
"MACRO_PLACEMENT_CFG": "dir::macro_placement.cfg"
```

**Outcome:** ✅ Root cause identified. Fix queued for Week 2. Flow state preserved at Step 9.

---

### Prompt W1-12 — IEEE LaTeX Report Generation

**AI Tool:** Claude (claude.ai)  
**Purpose:** Generate the 1-page IEEE two-column LaTeX report for Week 1 submission.

**Prompt given to AI:**
```
Generate a 1-page IEEE two-column LaTeX report (IEEEtran class) for my Week 1
mixed-signal physical design task. Include:
- Title: "AI-Assisted Mixed-Signal Physical Design Flow"
- Author: Bhuvanachandra Kusuma
- Sections: Abstract, Objective, Repo Summary, AI Workflow, Experiments,
  Results & Observations, Challenges & Next Steps, Conclusion
- A table summarizing the 5 compatibility issues fixed (Issue / Root Cause / Fix)
- A figure placeholder for the Magic layout screenshot
- Compiled with xelatex
The AI tool used throughout should be referenced as "Claude (Anthropic's AI assistant)".
```

**AI Output:** Complete `.tex` file generated with IEEEtran class, two-column layout, table using `\footnotesize` with `|c|p{1.55in}|p{1.45in}|` columns, figure with `[H]` placement, margin adjustments via `\setlength` (geometry package avoided due to column width conflicts).

**Manual Edits Made:**
- Replaced placeholder author name with "Bhuvanachandra Kusuma"
- Inserted actual Magic layout screenshot (`design_mux_pdn_placement_cropped.png`)
- Adjusted `\topmargin` and `\textheight` to fit all content on one page
- Table moved to immediately after `\end{abstract}` to prevent bottom page cutoff
- Verified compiled PDF visually — all content within one page

**Outcome:** ✅ Report compiled successfully with `xelatex`. Submission-ready.

---

## WEEK 2 — Manual Macro Placement, Routing, DRC/LVS, Final GDS

> Prompts below are added progressively as each block is executed and verified.

---

### Prompt W2-01 — Manual Macro Placement Configuration

**AI Tool:** Claude (claude.ai)  
**Purpose:** Fix the macro placement boundary issue blocking detailed placement.

**Prompt given to AI:**
```
In my OpenLane v1 design_mux flow, AMUX2_3V macro placement fails because
the auto-placer puts it at Y=124.432um outside the 122.4um floorplan.
My floorplan is set with FP_CORE_UTIL=20, and the macro is dual-height (~5.44um tall).
Generate a macro_placement.cfg with safe in-bounds coordinates,
and update config.json to reference it.
```

**Status:** 🔄 In progress — Week 2

---

### Prompt W2-02 — Completing Placement through Routing

**Status:** 🔲 Pending

---

### Prompt W2-03 — DRC with Magic

**Status:** 🔲 Pending

---

### Prompt W2-04 — LVS with Netgen

**Status:** 🔲 Pending

---

### Prompt W2-05 — Final GDS Verification

**Status:** 🔲 Pending

---

## Summary Table

| Prompt ID | Block | AI Tool | Outcome |
|-----------|-------|---------|---------|
| W1-01 | Repo comprehension | Claude | ✅ Complete |
| W1-02 | Tool installation | Claude | ✅ Complete |
| W1-03 | LEF/LIB/Verilog explanation | Claude | ✅ Complete |
| W1-04 | Config file setup | Claude | ✅ Complete |
| W1-05 | Blackbox Verilog fix | Claude | ✅ Complete |
| W1-06 | Duplicate module fix | Claude | ✅ Complete |
| W1-07 | Port typo fix | Claude | ✅ Complete |
| W1-08 | LIB threshold attributes | Claude | ✅ Complete |
| W1-09 | Resizer conflict workaround | Claude | ✅ Complete |
| W1-10 | PDN/layout verification | Claude | ✅ Complete |
| W1-11 | Macro placement root cause | Claude | ✅ Complete |
| W1-12 | LaTeX report generation | Claude | ✅ Complete |
| W2-01 | Manual macro placement | Claude | 🔄 In progress |
| W2-02 | Placement → routing | Claude | 🔲 Pending |
| W2-03 | Magic DRC | Claude | 🔲 Pending |
| W2-04 | Netgen LVS | Claude | 🔲 Pending |
| W2-05 | Final GDS verification | Claude | 🔲 Pending |
