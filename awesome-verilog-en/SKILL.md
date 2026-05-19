---
name: awesome-verilog-en
description: FPGA-focused Verilog-2005 RTL generation, review, debugging, and testbench guidance. Use when generating synthesizable Verilog modules from natural language, deriving interfaces, running self-checks, or reviewing FSM, CDC, and synthesis-safety issues.
---

# FPGA Verilog Skill

## Overview

Convert natural-language requirements into synthesizable FPGA-oriented Verilog-2005 RTL. Prioritize correct hardware behavior, synthesis safety, and clear interfaces. Always self-check generated code before output.

## Response Organization

Emit only the sections needed for the task:

- New module generation: `Interface Derivation` → complete RTL → `Self-Check Summary`
- RTL review: list issues and locations first, then give fixes; if the user asks, provide corrected code directly
- Debug explanation: explain the hardware behavior first, then likely causes and the smallest useful change
- For CDC, FSM, handshake, or arithmetic tasks, state key assumptions and the edge cases most likely to fail

## Interaction Principle

If the user's request conflicts with any rule in this skill (naming, reset, formatting, etc.), **confirm with the user first** before proceeding. State the conflicting rule, the user's request, and ask which to follow. Do not silently override either side.

## Mandatory Rules

1. **Use Verilog-2005 syntax only**
   - Prohibited: `logic`, `always_ff`, `always_comb`, `always_latch`, `interface`, `modport`, `unique case`, `priority case`, `typedef enum`, `return` in functions, variable declarations in unnamed `begin...end` blocks

2. **Use port-name prefixes**
   - Input `i_`, output `o_`, bidirectional `io_`; do not use suffixes `_i`/`_o`/`_io`
   - Clock port `i_clk`, reset port `i_rst_n`; internal aliases may be `clk`/`rst_n`

3. **Active-low async reset `rst_n`**
   - Sensitivity list `always @(posedge clk or negedge rst_n)`, reset test `if (!rst_n)`, reset branch at the top of the block
   - If reset enters asynchronously, synchronize release per clock domain

4. **`begin`/`end` formatting**
   - Do not wrap single statements; use `begin`/`end` only for multi-statement bodies
   - Put `begin` and `end` each on its own line, indented 4 spaces from the controlling statement, with `end` aligned to `begin`

5. **`reg`/`wire` driver rules**
   - Use `reg` for `always`-driven signals, `wire` for `assign`-driven signals; output ports use `output wire`, driven by `assign` from internal `_reg`
   - Do not drive `reg` with `assign`; do not drive `wire` from `always`; each signal has exactly one driver

6. **Assignment rules**
   - Nonblocking `<=` in sequential logic; blocking `=` in combinational logic; never mix in the same `always` block

7. **No placeholder code**
   - No `// TODO`, `// placeholder`, or `// implement later`; every module must be complete and synthesizable

8. **No forward references**
   - Declare all signals before first use; use `` `default_nettype none`` to prevent implicit nets

9. **Stable handshake protocols**
   - Hold `valid` until the corresponding `ready` accepts it; keep data and sideband signals stable when `valid=1` and `ready=0`

10. **Explicit hierarchy connections**
    - Use named port connections for instantiation only; no positional connections; override parameters with `#(...)` only; no `defparam`

11. **Self-check before output**
    - Run self-check after generating code (see below); fix any `[FAIL]` items before output

## Workflow

### Step 1: Interface Derivation

Derive the interface from the user request. If details are missing, choose conservative defaults and state assumptions. At minimum consider: clock domains, reset source, handshake/backpressure, width/signedness, latency, and CDC boundaries.

```text
## Interface Derivation

Module name: <lower_snake_case_name>
Parameters:
  parameter DATA_WIDTH = 8   // Data bus width in bits
Ports:
  input  wire                  i_clk
  input  wire                  i_rst_n
  input  wire [DATA_WIDTH-1:0] i_data
  output wire [DATA_WIDTH-1:0] o_result
```

### Step 2: Code Generation

Generate one complete `.v` module. See `references/02_module_structure.md` and `references/01_coding_style.md` for structure and style details. If the module would exceed roughly 500 lines, recommend submodule partitioning or ask the user whether to split across files.

### Step 3: Self-Check

```text
## Self-Check Summary

[PASS/FAIL] Complete file header (File/Date/Description filled)
[PASS/FAIL] File structure (resetall / timescale / default_nettype / endmodule / resetall)
[PASS/FAIL] Verilog-2005 syntax (no SystemVerilog syntax)
[PASS/FAIL] No simulation-only behavior (no #delay/$display/$finish/$monitor in RTL)
[PASS/FAIL] Spacing (indentation, spaces, alignment, and trailing whitespace follow rules)
[PASS/FAIL] Parameter comments (every parameter has a same-line comment)
[PASS/FAIL] reg/wire driver rules
[PASS/FAIL] No forward references
[PASS/FAIL] No multiple drivers
[PASS/FAIL] No placeholders or empty implementation
[PASS/FAIL] Width matching
[PASS/FAIL] Arithmetic width safety (carry, truncation, and sign extension handled explicitly)
[PASS/FAIL] No unused signals
[PASS/FAIL] No latch inference (all always @* paths assign values, case has default)
[PASS/FAIL] Reset coverage (all registers reset, or datapath exceptions explained)
[PASS/FAIL] Port naming (uses i_/o_/io_ prefixes)
[PASS/FAIL] begin/end formatting
[PASS/FAIL/N/A] Clock-domain crossing check (CDC signals are synchronized)
[PASS/FAIL/N/A] AXI/valid-ready handshake check
[PASS/WARN/N/A] User style applied without overriding mandatory rules
[PASS/FAIL] Prohibited construct check (no SystemVerilog leakage)
```

`[FAIL]` must be fixed before output; `[WARN]` must include an explanation; `N/A` only when the task does not involve that item.

## Reference Routing

Load the smallest relevant reference set for the task:

| Topic Area | Reference File | Content Boundary |
|---|---|---|
| Coding style | `references/01_coding_style.md` | Naming, spacing, comments, declarations, assignments, case, prohibited constructs |
| Module structure | `references/02_module_structure.md` | Header, directives, ports/parameters, functions/tasks, memories, instantiation, generate |
| Design patterns | `references/03_design_patterns.md` | Reset, clock, CDC, FSM, common RTL, handshakes, testbench |
| Review | `references/04_review_checklist.md` | Review order, anti-patterns, error reference, self-check checklist |
