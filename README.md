# Tournament-Branch-Predictor---ASIC-Design-and-Implementation

An end-to-end physical design implementation of a Tournament Branch Prediction Unit (BPU) for a 5-stage RISC-V pipeline, carried from synthesizable RTL through floorplanning, placement, clock tree synthesis, routing, and static timing sign-off to a final GDSII layout, using the GPDK45 (45nm) process with the Cadence Genus/Innovus toolchain.

This repository documents the complete digital VLSI implementation flow applied to a real microarchitectural block, rather than the RTL/microarchitecture alone.

---

## Motivation

Branch predictors are a good vehicle for learning physical design because they combine several distinct structures in one small block:

- Table-based storage (LHT, PHT, GHR-indexed tables) — SRAM-like array behavior
- Control logic and multiplexing — the choice/meta predictor
- A tag-compare datapath — the BTB
- A design that is small enough to floorplan and route by hand-tuned constraints, but complex enough to expose real congestion, timing, and clock-tree challenges

The goal of this project was to take a functionally verified tournament predictor design all the way to a manufacturable layout, and to build hands-on fluency with every stage of the ASIC implementation flow along the way.

---

## Flow Overview

```text
RTL (Verilog) 
      │
      ▼
Logic Synthesis (Genus)
      │
      ▼
Floorplanning (Innovus)
      │
      ▼
Power Planning (rings, stripes, rails)
      │
      ▼
Placement
      │
      ▼
Clock Tree Synthesis (CTS)
      │
      ▼
Routing (Global + Detailed)
      │
      ▼
Static Timing Analysis (Sign-off)
      │
      ▼
Physical Verification (DRC / LVS)
      │
      ▼
GDSII
```

---

## Microarchitecture Under Implementation

The predictor combines local and global prediction schemes with a dynamic choice predictor:

- **Local Predictor** — Local History Table (LHT) + Pattern History Table (PHT) using 2-bit saturating counters, tuned for loop-heavy and per-branch-correlated behavior.
- **Global Predictor** — Gshare-style design with a Global History Register (GHR) XOR-folded against the PC to index a shared Pattern History Table.
- **Choice Predictor** — A GHR-indexed meta-predictor with its own 2-bit saturating counters that dynamically arbitrates between the local and global predictions.
- **Branch Target Buffer (BTB)** — Direct-mapped, full-PC tag validation, with anti-aliasing protection against tag collisions.

Functional verification and microarchitectural benchmarking (gem5, IPC/accuracy comparisons against baseline/Gshare/Perceptron predictors) were completed prior to this implementation and informed area/timing budgets going into physical design — the emphasis of this repository is the backend flow that follows.

---

## RTL & Synthesis

- RTL written in synthesizable Verilog, structured to keep the local, global, choice, and BTB datapaths as separable hierarchical blocks — this made floorplanning by functional region straightforward later.
- Logic synthesis performed in **Cadence Genus**, targeting the GPDK45 standard cell library.
- Synthesis constraints (SDC) developed to set realistic clock period targets, input/output delays, and false-path/multicycle exceptions where applicable (e.g., predictor table update paths that don't sit on the critical prediction path).
- Post-synthesis netlist checked for:
  - Timing closure at the target frequency
  - Area utilization
  - Absence of latches/combinational loops from unintended inference

---

## Floorplanning

- Defined core and die area based on the synthesized cell area plus routing/congestion margin.
- Partitioned the floorplan around the four functional regions (local predictor, global predictor, choice predictor, BTB) to keep related logic physically close and minimize long cross-block routes.
- Placed macros/hard blocks (where used) with attention to pin access and blockage regions to avoid downstream routing congestion.
- Defined power planning: core rings, power stripes, and standard cell rails, sized to keep IR drop within budget across the die.

---

## Placement

- Standard-cell placement performed in **Innovus**, using the floorplan and power structures above as constraints.
- Iterated on placement to manage:
  - Congestion around the tag-compare/BTB region, which has denser fan-in/fan-out than the predictor tables
  - Timing-driven placement for paths feeding the choice predictor's final mux, which sits on the critical path to `next_pc`
- Ran trial placement + timing/congestion analysis loops before committing to a final placement.

---

## Clock Tree Synthesis (CTS)

- Built the clock tree in Innovus with target skew and latency constraints appropriate for the pipeline's 5-stage timing budget.
- Balanced clock tree structure across the floorplan to minimize skew between the fetch-stage prediction logic and the update logic that writes back to the PHTs/BTB after branch resolution.
- Verified post-CTS:
  - Clock skew within spec
  - Clock latency balance across major register banks
  - No hold violations introduced by the clock tree itself

---

## Routing

- Global and detailed routing performed in Innovus following CTS.
- Addressed routing congestion hotspots identified in placement, particularly around the BTB tag-compare logic and the choice-predictor mux fan-in.
- Verified routing quality:
  - Zero DRC violations post-route
  - Acceptable via count and wire length distribution
  - No routing shorts/opens

---

## Static Timing Analysis (STA)

- Full STA performed across setup and hold corners.
- Focused analysis on:
  - The prediction path (PC → table lookup → choice mux → `next_pc`), which is the timing-critical path for pipeline fetch bandwidth
  - The update path (branch resolution → PHT/BTB write), which had more timing slack and was a candidate for multicycle path exceptions
- Closed timing violations through a combination of resizing, buffering, and placement adjustment rather than relaxing constraints where avoidable.
- Confirmed setup and hold closure at sign-off corners before proceeding to physical verification.

---

## Physical Verification

- DRC (Design Rule Check) and LVS (Layout vs. Schematic) run to confirm the final layout is manufacturable and functionally matches the synthesized netlist.
- Final GDSII generated after clean DRC/LVS sign-off.

---

## Skills Demonstrated

- RTL design for synthesis (not just simulation)
- Logic synthesis and SDC constraint development (Genus)
- Floorplanning, power planning, and macro/pin placement
- Timing-driven and congestion-aware standard cell placement
- Clock tree synthesis and skew/latency management
- Global and detailed routing, DRC-clean closure
- Static timing analysis across setup/hold corners, exception handling (false paths / multicycle paths)
- DRC/LVS physical verification and GDSII generation
- End-to-end ownership of a real microarchitectural block from RTL to tape-out-ready layout

---

## Tools Used

- **RTL / Simulation:** Verilog HDL
- **Synthesis:** Cadence Genus
- **Physical Design (Floorplan → Route → STA):** Cadence Innovus
- **Process:** GPDK45 (45nm academic PDK)
- **Prior microarchitectural evaluation:** gem5 Architectural Simulator

---

## Directory Structure

```text
├── rtl/              Synthesizable Verilog RTL
├── constraints/       SDC timing constraints
├── sta/                Static timing analysis reports
├── verification/       DRC / LVS reports
├── gds/                Final GDSII output
├── docs/               Flow documentation and notes
└── README.md
```

---

## Future Work

- Extend the physical design flow to advanced sign-off checks (EM/IR analysis)
- Explore multi-corner multi-mode (MCMM) timing closure
- Re-target the flow to an open-source PDK (SKY130) for cross-node comparison
- Integrate the predictor into a full custom RISC-V core physical design

---

## Authors

Vicky Kumar

Electronics Engineering
GCET
