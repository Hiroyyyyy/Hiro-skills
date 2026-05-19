# 04 Review Checklist

This file is the pre-output review guide for this skill.
It provides review order, anti-patterns, and a final self-check checklist to keep outputs reliable.

## Review Order

| Order | Check | Focus |
|---|---|---|
| 1 | Language | Verilog-2005 only, no SystemVerilog syntax or other prohibited constructs |
| 2 | File structure | Header, compiler directives, final `` `resetall`` |
| 3 | Spacing | Indentation, spaces, alignment, trailing whitespace |
| 4 | Parameter comments | Every `parameter` has a same-line comment |
| 5 | Ports | `i_`, `o_`, `io_` prefixes; outputs are `output wire` |
| 6 | Declarations | Every signal declared before first use |
| 7 | Drivers | `reg/wire` driver relation, single driver |
| 8 | Combinational logic | Defaults, complete paths, `case default` |
| 9 | Sequential logic | Reset coverage, nonblocking assignment, single-statement block style |
| 10 | CDC | Cross-domain signals synchronized |
| 11 | Handshake | valid/ready and AXI channel stability |
| 12 | Completeness | No placeholders, no unused signals, no empty implementation |

## Anti-Pattern: Combinational Loop

| Problem | Impact | Fix |
|---|---|---|
| Combinational signal depends on itself | Unstable hardware meaning | Use a register or explicit next value |

Wrong:

```verilog
assign ready_next = ready_next & i_enable;
```

Correct:

```verilog
always @*
    begin
        ready_next = ready_reg;

        if (i_enable)
            ready_next = 1'b1;
        else
            ready_next = 1'b0;
    end
```

## Anti-Pattern: Unintentional Latch

| Problem | Impact | Fix |
|---|---|---|
| Some combinational paths do not assign | Latch inferred | Set defaults at block top |

Wrong:

```verilog
always @*
    if (i_enable)
        data_next = i_data;
```

Correct:

```verilog
always @*
    begin
        data_next = data_reg;

        if (i_enable)
            data_next = i_data;
    end
```

## Anti-Pattern: Multiple-Driven Output

| Problem | Impact | Fix |
|---|---|---|
| Output driven by both `assign` and `always` | Multiple-driver conflict | Drive output through one internal `_reg` and `assign` |

Wrong:

```verilog
assign o_valid = valid_reg;

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        o_valid <= 1'b0;
```

Correct:

```verilog
reg valid_reg;

assign o_valid = valid_reg;

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        valid_reg <= 1'b0;
    else
        valid_reg <= valid_next;
```

## Anti-Pattern: Explicit Combinational Sensitivity List

| Problem | Impact | Fix |
|---|---|---|
| `always @(a or b)` misses an input | Behavior mismatch | Use `always @*` |

Wrong:

```verilog
always @(a or b)
    y = a & b & c;
```

Correct:

```verilog
always @*
    y = a & b & c;
```

## Common Error Quick Reference

| Symptom | Root Cause | Fix |
|---|---|---|
| `reg` driven by continuous assignment | `assign` drives a `reg` | Change to `wire`, or drive it in `always` |
| Signal cannot be found | Forward reference or implicit net | Declare before first use |
| Declaration inside unnamed block fails | Variable declared inside unnamed `begin...end` | Move it to module scope or outside unnamed block |
| Multiple drivers | Same signal driven by multiple sources | Keep one driver source |
| Latch inferred | Some `always @*` path does not assign | Set defaults at the top |
| Parameter lacks description | `parameter` has no comment | Add same-line comment to the definition |
| Spacing is inconsistent | Indentation, spaces, or alignment drifted | Apply 4-space indentation and column alignment |
| valid glitch | valid deasserted before ready accepts it | Hold valid until `valid && ready` |
| AXI response dropped | `BVALID` / `RVALID` not held | Hold until the matching ready accepts it |
| CDC metastability | Cross-domain signal not synchronized | Use single-bit synchronizer, multi-bit FIFO, or handshake |
| Width warning | Connection or arithmetic width mismatch | Explicitly truncate, extend, or zero-pad |
| Signed comparison acts wrong | One side is unsigned or widths differ | Declare signed explicitly and extend to equal width |
| Positional instantiation miswired | Port order changed or manually mismatched | Use named port instantiation |
| Simulation control in RTL | Uses `force`, `release`, `fork`, or `join` | Move it to the testbench or rewrite as hardware logic |

## Self-Check Checklist

```text
## Self-Check Summary

[PASS/FAIL] Complete file header (File/Date/Description filled)
[PASS/FAIL] File structure (resetall / timescale / default_nettype / endmodule / resetall)
[PASS/FAIL] Verilog-2005 syntax (no SystemVerilog syntax)
[PASS/FAIL] No simulation-only behavior (no #delay/$display/$finish/$monitor in RTL)
[PASS/FAIL] Spacing (indentation, spaces, alignment, and trailing whitespace follow rules)
[PASS/FAIL] Parameter comments (every parameter has a same-line comment)
[PASS/FAIL] reg/wire driver rules (always drives reg, assign drives wire)
[PASS/FAIL] No forward references (all signals declared before use)
[PASS/FAIL] No multiple drivers (each signal has one driver)
[PASS/FAIL] No placeholders or empty implementation
[PASS/FAIL] Width matching (assignment and port widths match)
[PASS/FAIL] Arithmetic width safety (carry, truncation, and sign extension handled explicitly)
[PASS/FAIL] No unused signals
[PASS/FAIL] No latch inference (all always @* paths assign values, case has default)
[PASS/FAIL] Reset coverage (all registers reset, or datapath exceptions explained)
[PASS/FAIL] Port naming (uses i_/o_/io_ prefixes, not suffixes)
[PASS/FAIL] begin/end formatting (single statements unwrapped; multi-statement blocks wrapped, own-line, aligned)
[PASS/FAIL/N/A] Clock-domain crossing check (CDC signals are synchronized)
[PASS/FAIL/N/A] AXI/valid-ready handshake check (valid held, data stable, responses not dropped)
[PASS/WARN/N/A] User style applied without overriding mandatory rules
[PASS/FAIL] Prohibited construct check (no SystemVerilog leakage)
```

Handling rules:

- If any item is `[FAIL]`, fix it before final output.
- `[WARN]` must explain the reason.
- Use `N/A` only when the task truly does not involve that item.
