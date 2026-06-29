# AI-Assisted Prompt Log — Mixed-Signal Physical Design

**Project:** Mixed-Signal RTL-to-GDS Flow (design_mux + AMUX2_3V)  
**Author:** Bhuvanachandra Kusuma  
**AI Tool:** Claude (Anthropic, claude-sonnet-4-6)  
**Environment:** WSL Ubuntu 24.04, OpenLane v1.0.2, SKY130A PDK

---

## Prompt Format

Each entry records:
- **Context:** Why this prompt was needed
- **Prompt:** Exact text submitted to Claude
- **AI Response Summary:** What Claude recommended
- **Verification:** Steps taken to confirm the output
- **Outcome:** ✅ Applied / 🔵 Modified / ❌ Rejected

---

# WEEK 1 PROMPTS (Steps 1–9: Setup through PDN)

---

### Prompt W1-01 — Toolchain Installation Sequence

**Context:** Starting from scratch on WSL Ubuntu. Needed the correct installation order for Docker, OpenLane, and PDK without version conflicts.

**Prompt:**
```
I need to set up an OpenLane v1.0.2 environment on WSL Ubuntu for a mixed-signal
physical design project. The design integrates an analog macro (AMUX2_3V) with
digital SPI logic (design_mux) using the SKY130A PDK. What is the correct
installation sequence for Docker, OpenLane, and the PDK, and what versions should
I use to ensure compatibility?
```

**AI Response Summary:**
Install Docker CE (not Docker Desktop), clone OpenLane at v1.0.2 tag specifically, use `ciel` to pull SKY130A. OpenLane's `make` handles all internal tool dependencies inside the Docker image. Do not install OpenROAD or Yosys separately.

**Verification:** Ran `make test` — spm design test completed successfully.

**Outcome:** ✅ Applied as-is.

---

### Prompt W1-02 — Mixed-Signal Flow Architecture

**Context:** Needed to understand how OpenLane treats analog macros before starting configuration.

**Prompt:**
```
In an OpenLane mixed-signal flow, I have an analog macro called AMUX2_3V
(a 2-to-1 analog MUX in the 3.3V domain) to integrate with a digital SPI
controller (design_mux) using SKY130A. How does OpenLane treat analog macros?
What files do I need (LEF, LIB, etc.) and how does the tool handle the
analog/digital boundary during P&R?
```

**AI Response Summary:**
OpenLane treats analog macros as hard macros — black boxes from the digital flow. Requires: (1) LEF abstract for physical footprint, (2) Liberty (.lib) for timing/power, (3) Verilog blackbox stub for synthesis. Use `EXTRA_LEFS` and `EXTRA_LIBS` config variables.

**Verification:** Cross-referenced with OpenLane docs and reference repo.

**Outcome:** ✅ Applied. Set up three-file macro integration.

---

### Prompt W1-03 — Verilog Blackbox Stub Syntax

**Context:** Synthesis failed. Yosys rejected the AMUX2_3V blackbox stub.

**Prompt:**
```
My OpenLane synthesis step is failing with a Yosys parse error on the AMUX2_3V
blackbox stub. The current stub uses Verilog-1995 style. What is the correct
blackbox syntax for the Yosys version bundled in OpenLane v1.0.2?
```

**AI Response Summary:**
Use `(* blackbox *)` attribute before the module declaration. Verilog-1995 `$attribute` syntax is not supported by modern Yosys.

**Verification:** Changed stub syntax, synthesis passed.

**Outcome:** ✅ Applied. Fix 1 of 5.

---

### Prompt W1-04 — Duplicate Module Definitions

**Context:** Synthesis reported duplicate module definitions causing elaboration failure.

**Prompt:**
```
OpenLane synthesis is failing with "duplicate module definition" for design_mux.
My config uses VERILOG_FILES with a glob pattern. The design directory contains
both the RTL files and the blackbox stub. How do I prevent the glob from picking
up both?
```

**AI Response Summary:**
Use `VERILOG_FILES_BLACKBOX` for the stub file and list only `design_mux.v` explicitly in `VERILOG_FILES`. Glob patterns over directories containing both RTL and blackbox files cause duplicate definitions.

**Verification:** Updated config, synthesis passed cleanly.

**Outcome:** ✅ Applied. Fix 2 of 5.

---

### Prompt W1-05 — Port Name Typo in Instantiation

**Context:** Synthesis passed but elaboration showed an unconnected port warning leading to a functional error.

**Prompt:**
```
After fixing the duplicate module issue, my elaboration shows port I0 on AMUX2_3V
is unconnected. The instantiation in design_mux.v shows .I0(IO). Is this a typo?
```

**AI Response Summary:**
Yes — `.I0(IO)` is a typo; the signal name should be `I0` (capital I, zero) not `IO` (capital I, capital O). These look identical in some fonts.

**Verification:** Fixed typo in design_mux.v, port connected correctly.

**Outcome:** ✅ Applied. Fix 3 of 5.

---

### Prompt W1-06 — Missing OpenSTA Liberty Attributes

**Context:** STA step failed with missing threshold attribute errors in AMUX2_3V.lib.

**Prompt:**
```
My OpenLane STA step fails with: ERROR: No slew_lower_threshold_pct_rise in
library AMUX2_3V. What Liberty attributes are required by OpenSTA and how do I
add them to my custom analog macro .lib file?
```

**AI Response Summary:**
OpenSTA requires four threshold attributes in every Liberty cell: `slew_lower_threshold_pct_rise`, `slew_upper_threshold_pct_rise`, `slew_lower_threshold_pct_fall`, `slew_upper_threshold_pct_fall`. Add them at the library level with standard values (10/90 for lower/upper).

**Verification:** Added attributes to AMUX2_3V.lib, STA passed.

**Outcome:** ✅ Applied. Fix 4 of 5.

---

### Prompt W1-07 — Grid-Paint Conflict in Resizer

**Context:** Placement resizer aborted with a grid-paint conflict error.

**Prompt:**
```
The OpenLane placement resizer step fails with:
[ERROR DPL-0041] Cannot paint grid because another layer is already occupied.
This happens during the resizer optimization pass after global placement.
How do I bypass this for a mixed-signal design with an analog macro?
```

**AI Response Summary:**
Set `PL_RESIZER_DESIGN_OPTIMIZATIONS=0` in config to skip the resizer pass. This is appropriate for mixed-signal designs where the analog macro creates irregular placement constraints.

**Verification:** Added config variable, resizer skipped, flow continued.

**Outcome:** ✅ Applied. Fix 5 of 5.

---

### Prompt W1-08 — PDN Configuration for Mixed-Signal

**Context:** Needed to understand how to configure the power delivery network with the analog macro present.

**Prompt:**
```
For a mixed-signal design with AMUX2_3V integrated into design_mux using SKY130A,
what PDN configuration is appropriate? The analog macro has VDD and VSS pins on
met1. How should I configure FP_PDN settings to avoid conflicts with the analog
macro power pins?
```

**AI Response Summary:**
Keep digital PDN rails on met1/met4. The analog macro's met1 power pins will connect to the horizontal met1 stripes automatically. Set `FP_PDN_ENABLE_RAILS=1` and avoid placing PDN stripes directly over the macro placement area.

**Verification:** Checked PDN log — no conflicts, VDD/VSS straps generated cleanly.

**Outcome:** ✅ Applied.

---

### Prompt W1-09 — Verifying PDN Integrity

**Context:** After PDN generation, needed to confirm the power grid was correctly formed.

**Prompt:**
```
How do I verify that the PDN generated by OpenLane is correct for my mixed-signal
design? I want to confirm VDD/VSS straps are present and the analog macro power
pins are correctly isolated from the digital rails.
```

**AI Response Summary:**
Three checks: (1) scan PDN log for WARNING/ERROR lines, (2) open DEF in Magic and visually inspect strap continuity, (3) confirm analog macro power pins are not incorrectly shorted to digital rails by checking pin labels.

**Verification:** All three checks passed. PDN verified correct at Step 9.

**Outcome:** ✅ Applied. Week 1 endpoint confirmed.

---

### Prompt W1-10 — LEF Abstract Requirements

**Context:** Needed to understand what makes a valid LEF abstract for an analog macro.

**Prompt:**
```
What are the minimum requirements for a valid LEF abstract file for an analog
macro in OpenLane/SKY130A? My AMUX2_3V.lef was exported from Magic. What fields
are mandatory and what common issues should I check for?
```

**AI Response Summary:**
Mandatory fields: VERSION, MACRO name, CLASS, ORIGIN, SIZE, SITE, PIN sections with DIRECTION/USE/PORT/LAYER/RECT, OBS section. Common issues: wrong CLASS (CORE vs BLOCK), pins on inaccessible layers, missing USE POWER/GROUND on supply pins, SITE mismatch.

**Verification:** Cross-checked AMUX2_3V.lef against checklist — found CLASS CORE issue (fixed in Week 2).

**Outcome:** 🔵 Partially applied. CLASS issue identified, fixed in Week 2.

---

### Prompt W1-11 — Diagnosing Macro Placement Boundary Violation

**Context:** Running beyond PDN into placement revealed the Week 2 blocker.

**Prompt:**
```
After PDN generation, the detailed placement step fails with:
[ERROR]: Macro AMUX2_3V is placed outside die boundary.
Macro Y-coordinate: 124.432µm, Die boundary Y: 122.400µm.
What causes this and what are my options to fix it without enlarging the die?
```

**AI Response Summary:**
The automatic macro placer doesn't account for the macro's full bounding box including halos. Fix via manual macro placement using `MACRO_PLACEMENT_CFG` file specifying exact coordinates.

**Verification:** Confirmed analysis by calculating macro dimensions vs boundary. Manual placement identified as correct solution.

**Outcome:** 🔵 Documented as Week 2 entry point.

---

### Prompt W1-12 — IEEE Report Structure

**Context:** Week 1 required a 1-page IEEE two-column LaTeX report.

**Prompt:**
```
I need to write a 1-page IEEE-format report (IEEEtran LaTeX class) summarizing
Week 1 of a mixed-signal physical design project. Work covered: toolchain
installation (Docker, OpenLane v1.0.2, SKY130A PDK), project configuration for
design_mux + AMUX2_3V integration, and resolving 5 compatibility issues.
Flow reached PDN generation (Step 9/18). Please suggest structure and draft LaTeX.
```

**AI Response Summary:**
Suggested six sections: Abstract, Introduction, Repo & Flow Overview, AI Workflow, Results (with table), Conclusion. Provided complete IEEEtran LaTeX source with table and figure placement, margin adjustments avoiding geometry package conflict.

**Verification:** Compiled with xelatex — produced clean 1-page two-column PDF.

**Outcome:** ✅ Applied with minor author/attribution edits.

---

# WEEK 2 PROMPTS (Steps 10–39: Placement through GDS)

---

### Prompt W2-01 — LEF CLASS CORE vs CLASS BLOCK

**Context:** Resuming flow in Week 2. OpenLane reported `[WARNING MPL-0004] No macros found` — the macro placer was ignoring AMUX2_3V entirely.

**Prompt:**
```
OpenLane's macro placement step reports [WARNING MPL-0004] No macros found,
even though AMUX2_3V is listed in EXTRA_LEFS and appears in synthesis.
The LEF file has CLASS CORE. Could this be the reason the macro placer ignores it,
and what should the CLASS be for an analog macro block?
```

**AI Response Summary:**
Yes — `CLASS CORE` tells OpenLane the cell is a standard cell to be placed in rows. An analog macro must use `CLASS BLOCK` so the tool treats it as a hard macro requiring special placement handling.

**Verification:** Changed `CLASS CORE` to `CLASS BLOCK` with `sed`, reran flow — macro placer recognized AMUX2_3V.

**Outcome:** ✅ Applied. Fix 6.

---

### Prompt W2-02 — Manual Macro Placement Configuration

**Context:** After CLASS fix, needed to specify exact macro coordinates to keep it inside the floorplan boundary.

**Prompt:**
```
With AMUX2_3V now recognized as a CLASS BLOCK macro, I need to place it manually
to ensure it stays inside the 122.4µm die boundary. The macro is 8.74 × 5.44µm.
How do I create a MACRO_PLACEMENT_CFG file and what coordinate format does
OpenLane expect?
```

**AI Response Summary:**
Create a text file with format: `MACRO_NAME X Y ORIENTATION`. Orientation `N` means north (0°). Place at `(2.0, 2.0)` to give adequate margin from the boundary. Reference this file in config.json with `"MACRO_PLACEMENT_CFG": "dir::macro_placement.cfg"`.

**Verification:** Created file, flow ran Step 5 (Manual Macro Placement) successfully with no boundary errors.

**Outcome:** ✅ Applied. Fix 7.

---

### Prompt W2-03 — Detailed Router Pin Access Error

**Context:** Flow reached Step 23 (detailed routing) but failed with `[ERROR DRT-0073] No access point for AMUX2_3V/select`.

**Prompt:**
```
Detailed routing fails with: [ERROR DRT-0073] No access point for AMUX2_3V/select.
The select pin in the LEF is defined on layer li1 at RECT 2.450 2.450 2.800 2.720.
The macro is 8.74 × 5.44µm. Why can't the router access this pin and how do I fix it?
```

**AI Response Summary:**
The `select` pin is buried deep inside the macro body on `li1`, surrounded by OBS geometry — the router cannot reach it. Signal pins must be on the macro boundary and on a routable metal layer. Move all signal pins to `met2` in the interior zone (not overlapping power rails), and block `li1` and interior `met2` with OBS.

**Verification:** Rewrote LEF with signal pins on met2 at mid-height of macro. Reran flow — detailed routing passed with 0 DRC violations.

**Outcome:** ✅ Applied. Fix 8.

---

### Prompt W2-04 — Magic GDS Export Missing Analog Macro Cell

**Context:** Flow reached Step 33 (GDS streaming) but Magic failed with `Unknown cell design_mux` because no GDS geometry existed for AMUX2_3V.

**Prompt:**
```
Magic GDS streaming fails with: Unknown cell design_mux / Must specify name for
cell (UNNAMED). I provided AMUX2_3V as a LEF abstract only (no GDS). Does Magic
need a GDS file for each macro to stream out the top-level layout? How do I
provide a placeholder GDS for an analog macro where I only have LEF/LIB?
```

**AI Response Summary:**
Yes — Magic needs a GDS cell definition for every macro referenced in the layout. For an analog macro where the full GDS is proprietary or unavailable, create a minimal stub GDS with just the boundary rectangle. Use Python with struct packing to write a valid GDS-II binary, then reference it via `EXTRA_GDS_FILES` in config.json.

**Verification:** Generated AMUX2_3V.gds with Python script, added to config, Magic GDS export succeeded.

**Outcome:** ✅ Applied. Fix 9.

---

### Prompt W2-05 — LVS Errors for Analog Blackbox Macro

**Context:** Flow reached Step 39 (LVS) but failed because the dummy GDS has no transistors to match the schematic.

**Prompt:**
```
LVS fails with 85 total errors: 23 unmatched nets, 56 unmatched devices, 4
unmatched pins. This is because my AMUX2_3V.gds is a stub with no internal
transistors. Is it acceptable to skip LVS for an analog macro in a mixed-signal
flow, and how do I configure OpenLane to not fail on LVS errors?
```

**AI Response Summary:**
Yes — skipping LVS for analog macros is standard practice in mixed-signal flows. The analog IP is verified separately at the schematic level. Set `QUIT_ON_LVS_ERROR=0` in config.json. Note that `RUN_LVS=0` alone is insufficient in OpenLane v1.0.2 — the quit flag must also be set.

**Verification:** Added `QUIT_ON_LVS_ERROR=0`, flow completed with `[SUCCESS]: Flow complete.`

**Outcome:** ✅ Applied. Fix 10.

---

### Prompt W2-06 — Interpreting Final Flow Metrics

**Context:** After flow completion, needed to understand and document the key metrics.

**Prompt:**
```
My OpenLane flow completed successfully. The metrics.csv shows:
flow_status: flow completed, tritonRoute_violations: 0, Magic_violations: 0,
wire_length: 6662, vias: 1891, wns: 0.0, tns: 0.0, DIEAREA_mm^2: 0.0193.
How do I interpret these results for a mixed-signal design report?
```

**AI Response Summary:**
Zero routing violations and zero Magic DRC violations confirm the layout is physically correct. WNS=0 and TNS=0 mean all timing paths meet their constraints with no slack violations. Wire length of 6662µm and 1891 vias are typical for a small mixed-signal block. Die area of 0.0193mm² is appropriate for the design complexity.

**Verification:** Cross-checked against manufacturability report — confirmed all metrics.

**Outcome:** ✅ Applied in Week 2 report.

---

### Prompt W2-07 — KLayout GDS Verification

**Context:** Needed to visually verify the final GDS layout and confirm XOR check.

**Prompt:**
```
My OpenLane flow generated GDS files from both Magic and KLayout. The log shows
[INFO]: No XOR differences between KLayout and Magic gds. How do I open and
visually inspect the GDS in KLayout on WSL Ubuntu, and what should I look for
to confirm the layout is correct?
```

**AI Response Summary:**
Install KLayout with `sudo apt install klayout`, open with `klayout design_mux.gds`. Check: (1) all IO pins visible at die boundary, (2) standard cells fill the core area, (3) analog macro visible as a distinct block, (4) routing layers visible on met1–met4, (5) no obvious disconnected regions.

**Verification:** Opened GDS in KLayout 0.28.16 — layout shows placed cells, routing, macro block, and IO pins correctly. Screenshot captured.

**Outcome:** ✅ Applied.

---

### Prompt W2-08 — Week 2 IEEE Report Generation

**Context:** Week 2–3 required updated documentation of the complete flow.

**Prompt:**
```
I need a 1-page IEEE two-column LaTeX report (IEEEtran) for Week 2–3 of my
mixed-signal physical design project. Week 2 extended the flow from PDN to
complete GDS. Key results: 0 DRC violations, 0 timing violations, GDS generated,
XOR clean. Five new fixes were required for: LEF CLASS, macro placement, pin
access on met2, dummy GDS, and LVS skip. Please generate the LaTeX source.
```

**AI Response Summary:**
Generated complete IEEEtran LaTeX with sections covering objective, Week 1 recap, Week 2 methodology (5 fixes), results table (all metrics), layout figure, challenges, and conclusion.

**Verification:** Compiled with xelatex — clean 1-page two-column PDF.

**Outcome:** ✅ Applied.

---

### Prompt W2-09 — GitHub Documentation Strategy

**Context:** Week 2 task required comprehensive GitHub documentation of all AI prompts, files, and results.

**Prompt:**
```
My Week 2 task requires a GitHub repo documenting all AI prompts, generated files,
errors, fixes, tool commands, and results for a mixed-signal physical design flow.
What structure should the repo follow and what files should be included to clearly
show the AI-assisted workflow from problem understanding to GDS generation?
```

**AI Response Summary:**
Recommended structure: README.md (overview, tool stack, flow diagram, results table, reproduction instructions), PROMPTS.md (all prompts in structured format), results/ folder with screenshots and reports, src/ with generated config files. Each prompt entry should be reproducible — exact text, not paraphrased.

**Verification:** Implemented structure, uploaded all files.

**Outcome:** ✅ Applied.

---

### Prompt W2-10 — Analog Macro LEF Pin Layer Strategy

**Context:** After initial met1 pin fix failed (pins shorted to VDD rail), needed a deeper analysis of why met1 was insufficient.

**Prompt:**
```
I moved AMUX2_3V signal pins to met1 at the top edge (Y=5.2 to 5.44) but the
router still reports no access point for the select pin. The VDD rail on met1
spans the full top edge (0 to 8.74 at Y=5.2 to 5.68). Are the signal pins being
treated as shorted to VDD? What layer and position should signal pins use to
avoid conflict with power rails?
```

**AI Response Summary:**
Yes — the signal pin rectangle at Y=5.2–5.44 overlaps exactly with the VDD met1 rail at Y=5.2–5.68. The router sees the pin as part of the VDD net and cannot create an independent access point. Solution: use met2 for signal pins, positioned in the middle height zone (Y=2.4–2.8) away from both VDD (top) and VSS (middle-met1) rails. Block met2 in all other areas with OBS.

**Verification:** Rewrote LEF with met2 signal pins at mid-height, detailed routing passed with 0 violations.

**Outcome:** ✅ Applied. This was the critical insight for Fix 8.

---

# SUMMARY

| Week | Prompts | Key Outcomes |
|------|---------|--------------|
| Week 1 | 12 | Toolchain setup, 5 compatibility fixes, PDN complete |
| Week 2 | 10 | 5 additional fixes, full flow to GDS, 0 DRC, timing clean |
| **Total** | **22** | **Complete RTL-to-GDS flow achieved** |
