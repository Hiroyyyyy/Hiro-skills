# 01 Coding Style

This file aligns coding style and writing conventions for this skill.
Use it first to align naming, spacing, and comment expectations before generation or review.

## Scope

| Category | Content |
|---|---|
| Language boundary | Verilog-2005, prohibited constructs, number literals |
| Naming | Modules, instances, ports, registers, synchronizers |
| Spacing | Indentation, spaces, alignment, blank lines |
| Comments | Parameter comments, intent comments, file-header description |
| Code shape | `begin/end`, `reg/wire`, assignments, `case` |

## Language Boundary

- Use Verilog-2005 only.
- Declare every signal before first use.
- Keep `` `default_nettype none`` to prevent implicit nets.
- Do not declare `reg` or `wire` inside unnamed `begin...end` blocks.

| Prohibited Construct | Replacement |
|---|---|
| `logic` | Use `reg` or `wire` by driver type |
| `always_ff` | `always @(posedge clk or negedge rst_n)` |
| `always_comb` | `always @*` |
| `always_latch` | Complete assignments; avoid latches |
| `interface` / `modport` | Expand into ordinary ports |
| `typedef enum` | `localparam` state encoding |
| `unique case` / `priority case` | `case` + `default` |
| `'0` | `{WIDTH{1'b0}}` |

## Naming

| Object | Rule | Example |
|---|---|---|
| Module | `lower_snake_case` | `pulse_counter` |
| Instance | `lower_snake_case`, `_inst` preferred | `pulse_counter_inst` |
| Internal signal | `lower_snake_case` | `data_valid` |
| `parameter` / `localparam` | `ALL_CAPS` | `DATA_WIDTH` |
| Input port | `i_` prefix | `i_valid` |
| Output port | `o_` prefix | `o_ready` |
| Bidirectional port | `io_` prefix | `io_sda` |
| Current register value | `_reg` suffix | `valid_reg` |
| Combinational next value | `_next` suffix | `valid_next` |
| Synchronizer | `_sync` suffix | `req_sync` |
| Active-low signal | `_n` suffix | `rst_n` |

Signal-name simplicity:

- Do not stack many suffixes.
- Name synchronizer shift registers directly as `req_sync`; do not write `req_sync_reg`.
- Port prefixes already express direction; do not use `_i`, `_o`, or `_io` suffixes.
- Names should describe hardware meaning; avoid weak names such as `tmp1`, `foo`, or `data2`.

## Spacing

| Item | Rule |
|---|---|
| Indentation | 4 spaces |
| Tabs | Prohibited |
| Trailing whitespace | Prohibited |
| Max line length | 100 characters |
| Comma | 1 space after comma |
| Binary operators | 1 space on both sides |
| Unary operators | No space between operator and operand |
| Function/task call | No space between name and `(` |
| Consecutive declarations | Align type, width, and name by column |
| Consecutive `assign` statements | Align LHS and `=` by column |
| Blank lines | Keep 1 blank line between logic sections |

Example:

```verilog
wire [DATA_WIDTH-1:0] sum_data;
wire                  sum_valid;

assign sum_data  = data_a + data_b;
assign sum_valid = i_valid & o_ready;
```

## Comments

| Object | Requirement |
|---|---|
| File-header `Description` | State module purpose and key behavior |
| `parameter` | Every parameter must have a same-line comment |
| `localparam` | States, constants, and encodings need short comments |
| CDC signal | Document crossing direction |
| Non-obvious logic | Explain hardware intent, not literal code |

Parameter-comment example:

```verilog
parameter DATA_WIDTH = 8,   // Data bus width in bits
parameter DEPTH      = 16   // Number of storage entries
```

## begin / end

| Case | Rule |
|---|---|
| Controlled body has 1 statement | Do not use `begin/end` |
| Controlled body has multiple statements | Use `begin/end` |
| `begin` / `end` | Each on its own line |
| `begin` indentation | 4 spaces deeper than control statement |
| `end` alignment | Align with matching `begin` |
| Named `generate` | `begin : block_name` is allowed |

Single statement:

```verilog
if (enable)
    data_next = i_data;
else
    data_next = data_reg;
```

Multiple statements:

```verilog
if (enable)
    begin
        data_next  = i_data;
        valid_next = 1'b1;
    end
else
    begin
        data_next  = data_reg;
        valid_next = 1'b0;
    end
```

## reg / wire and Assignments

| Driver | Declaration Type | Assignment Type |
|---|---|---|
| `always @(posedge ...)` | `reg` | Nonblocking `<=` |
| `always @*` | `reg` | Blocking `=` |
| `assign` | `wire` | Continuous assignment |
| Output port | `output wire` | Driven by internal signal through `assign` |

Rules:

- Do not drive a `reg` with `assign`.
- Do not drive a `wire` from an `always` block.
- Do not mix `=` and `<=` in the same `always` block.
- Each signal must have one driver.

Output port pattern:

```verilog
output wire o_valid;

reg valid_reg;

assign o_valid = valid_reg;
```

## case

| Item | Rule |
|---|---|
| Exact matching | Use `case` |
| Default branch | Required |
| Prohibited | `casex`, `full_case`, `parallel_case` |
| Single-statement case item | Do not wrap in `begin/end` |
| Multi-statement case item | Use `begin/end` |

```verilog
case (state_reg)
    STATE_IDLE:
        valid_next = 1'b0;
    STATE_RUN:
        begin
            valid_next = 1'b1;
            busy_next  = 1'b1;
        end
    default:
        valid_next = 1'b0;
endcase
```

## Number Literals

| Case | Rule | Example |
|---|---|---|
| Constant | Explicit width | `8'd16` |
| Single bit | Explicit width | `1'b0` |
| Parameterized zero | Replication operator | `{DATA_WIDTH{1'b0}}` |
| Long literal | Underscore grouping allowed | `16'h12_ab` |
| Narrow to wide | Explicit zero/sign extension | `{8'd0, byte_data}` |

## Prohibited Quick Reference

| Prohibited Item | Reason |
|---|---|
| Placeholder code | Module must be complete |
| Implicit nets | Hides spelling mistakes |
| Forward references | Hard to read and error-prone |
| Multiple drivers | Hardware meaning is ambiguous |
| Gated clocks | FPGA clock networks should not be rewritten by combinational logic |
| Unsynchronized CDC | May create metastability |
| `#delay` in RTL | Not synthesizable |
| `$display` / `$finish` in RTL | Testbench or parameter-check only |
