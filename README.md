# Mixed-Signal Physical Design — AI-Assisted OpenLane Flow
**Author:** Bhuvanachandra Kusuma  
**Course Task:** Week 1 (complete) + Week 2–3 (in progress)  
**Reference Repo:** [praharshapm/vsdmixedsignalflow](https://github.com/praharshapm/vsdmixedsignalflow)

---

## Overview

This repository documents an AI-assisted reproduction of a mixed-signal RTL-to-GDS physical design flow using **OpenLane v1.0.2**, **SKY130A PDK**, and the analog macro **AMUX2_3V** (2:1 analog multiplexer).

The goal is **not** to copy the reference repo, but to use AI-generated prompts (Claude / Anthropic) to understand, configure, execute, debug, and verify each stage of the flow independently.

---

## Tools & Environment

| Tool | Version |
|------|---------|
| OpenLane | v1.0.2 (The-OpenROAD-Project) |
| SKY130A PDK | via `ciel` |
| Magic | 8.x (SKY130 tech) |
| Netgen | for LVS |
| OS | WSL Ubuntu (BhuvanLAPTOP) |
| AI Tool | Claude (Anthropic's AI assistant) |

---

## Repository Structure

```
├── README.md               ← This file
├── PROMPTS.md              ← All AI prompts used, with outputs and verification notes
├── designs/
│   └── design_mux/
│       ├── config.json     ← OpenLane configuration
│       ├── src/
│       │   ├── design_mux.v
│       │   ├── raven_spi.v
│       │   ├── spi_slave.v
│       │   ├── lef/AMUX2_3V.lef
│       │   └── lib/AMUX2_3V.lib
│       ├── bb/
│       │   └── AMUX2_3V.v  ← Blackbox stub (fixed from original)
│       └── macro_placement.cfg  ← Week 2: manual macro placement
├── results/
│   ├── screenshots/        ← Magic layout screenshots
│   └── logs/               ← Key OpenLane log excerpts
└── report/
    ├── week1_report.tex    ← IEEE two-column LaTeX report
    └── week1_report.pdf    ← Compiled output
```

---

## Week 1 — Results Summary

| Step | Status | Notes |
|------|--------|-------|
| Tool installation (Docker, OpenLane, SKY130) | ✅ Done | `make test` passed |
| Design directory setup | ✅ Done | All source files in place |
| Synthesis | ✅ Done | After 3 bug fixes (see below) |
| Floorplanning | ✅ Done | |
| Global placement | ✅ Done | |
| PDN generation | ✅ Done | Layout verified in Magic |
| Detailed placement | ❌ Blocked | Macro placed outside floorplan boundary |
| Routing | ⏳ Week 2 | |
| DRC / LVS | ⏳ Week 2 | |
| Final GDS | ⏳ Week 2 | |

### 5 Compatibility Issues Fixed (Week 1)

| # | Issue | Fix |
|---|-------|-----|
| 1 | `AMUX2_3V.v` used Verilog-1995 style + invalid `initial` on wire | Replaced with `(* blackbox *)` stub |
| 2 | `VERILOG_FILES` glob caused duplicate module definitions | Listed only `design_mux.v` explicitly |
| 3 | `.I0(IO)` typo in AMUX2_3V instantiation | Corrected to `.I0(I0)` |
| 4 | `AMUX2_3V.lib` missing 8 OpenSTA threshold attributes | Added standard SKY130 50/30/70% values |
| 5 | Resizer DPL-0041 grid-paint conflict near macro | Set `PL_RESIZER_DESIGN_OPTIMIZATIONS=0` |

---

## Week 2–3 — Plan

1. **Fix macro placement** — use `macro_placement.cfg` to manually place `AMUX2_3V` within the 122.4µm floorplan boundary
2. **Complete placement** — re-run detailed placement and legalization
3. **CTS + Routing** — complete clock tree synthesis and global/detailed routing
4. **DRC** — run Magic DRC on final layout
5. **LVS** — run Netgen LVS comparing GDS netlist vs schematic
6. **Final GDS** — verify output with KLayout

---

## AI Workflow

All prompts, AI responses, manual verifications, and edits are documented in [`PROMPTS.md`](./PROMPTS.md).

**Key principle:** Every AI-generated command, file, or config value was run or cross-checked against actual tool output before being accepted. AI was used as a co-pilot, not a black box.

---

## How to Reproduce

```bash
# 1. Clone this repo
git clone <this-repo-url>

# 2. Set up OpenLane (if not already installed)
git clone https://github.com/The-OpenROAD-Project/OpenLane.git ~/OpenLane
cd ~/OpenLane && make pull-openlane

# 3. Copy design files
cp -r designs/design_mux ~/OpenLane/designs/

# 4. Run the flow
cd ~/OpenLane
make quick_run QUICK_RUN_DESIGN=design_mux
```

---

## Report

The Week 1 IEEE two-column report is in `report/week1_report.pdf`.  
It was generated using LaTeX (IEEEtran class) with AI assistance and compiled using `xelatex`.
