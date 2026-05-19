# 02 Module Structure

This file defines how module files should be organized and structured in this skill.
Use its templates when generating modules to avoid structural mistakes.

## File Organization

| Order | Content | Requirement |
|---|---|---|
| 1 | File header | Include only `File`, `Date`, `Description` |
| 2 | Compiler directives | `` `resetall``, `` `timescale``, `` `default_nettype none`` |
| 3 | Module declaration | Separate parameter and port blocks |
| 4 | Internal declarations | `localparam`, `wire`, `reg`, memory |
| 5 | Continuous assignments | Output aliases and combinational wiring |
| 6 | Logic blocks | `always @*` and sequential `always` |
| 7 | Submodule instantiation | Named port connections |
| 8 | File ending | Final `` `resetall`` after `endmodule` |

## File Header

```verilog
// -----------------------------------------------------------------------------
// File : pulse_counter.v
// Date : 2026-05-19
// -----------------------------------------------------------------------------
// Description:
//   Counts accepted input pulses and exposes the current count.
// -----------------------------------------------------------------------------
```

File-header rules:

- Put it on the first line of the file.
- Do not include an author field.
- Do not include a change log.
- Use `YYYY-MM-DD` for `Date`.
- Write module purpose in `Description`, not a line-by-line implementation summary.

## Compiler Directives

```verilog
`resetall
`timescale 1ns / 1ps
`default_nettype none
```

Ending:

```verilog
`resetall
```

## Module Skeleton

```verilog
module pulse_counter #
(
    parameter COUNT_WIDTH = 8  // Counter width in bits
)
(
    input  wire                   i_clk,
    input  wire                   i_rst_n,
    input  wire                   i_pulse,
    input  wire                   i_clear,
    output wire [COUNT_WIDTH-1:0] o_count
);

wire clk;
wire rst_n;

assign clk   = i_clk;
assign rst_n = i_rst_n;

reg [COUNT_WIDTH-1:0] count_reg;

assign o_count = count_reg;

endmodule
```

Module-declaration rules:

| Item | Rule |
|---|---|
| File and module | One primary module per file by default; file name matches module name |
| Parameter block | Write `(` and `)` separately after `#` |
| Port block | Write `(` and `)` separately after parameter block |
| Port order | Clock, reset, inputs, outputs, bidirectional |
| Port type | Explicit `wire` on every port |
| Output port | Use `output wire`; do not use `output reg` |
| Parameter comments | Every `parameter` has a same-line comment |

## Parameters and Constants

| Type | Purpose | Example |
|---|---|---|
| `parameter` | User-configurable value | `DATA_WIDTH` |
| `localparam` | Internal constant, state encoding | `STATE_IDLE` |
| `integer` | Function-local calculation, loop variable | `index` |

Parameter rules:

- Every `parameter` must have a reasonable default value.
- Every `parameter` must have a same-line comment.
- Do not use `` `define`` to parameterize modules.
- Do not use `defparam` to override parameters.
- Prefer `localparam` for derived values.

```verilog
parameter DATA_WIDTH = 8,   // Data bus width in bits
parameter DEPTH      = 16   // Number of storage entries
```

## Parameter Validation

Parameter validation catches illegal configurations; it does not describe hardware behavior:

```verilog
initial
    if (COUNT_WIDTH < 1)
        begin
            $display("ERROR: COUNT_WIDTH must be at least 1 (instance %m)");
            $finish;
        end
```

## Verilog-2005 Width Function

```verilog
function integer clog2_int;
    input integer value;
    integer temp;
    begin
        temp = value - 1;
        for (clog2_int = 0; temp > 0; clog2_int = clog2_int + 1)
            temp = temp >> 1;
    end
endfunction
```

Function rules:

| Item | Rule |
|---|---|
| Time behavior | No `#delay` or event controls |
| Return style | Assign the function name; do not use `return` |
| Side effects | Do not modify module-level registers |
| Call relation | Do not call a `task` from a function |

## task Boundaries

| Scenario | Rule |
|---|---|
| Synthesizable RTL | Prefer not to use `task` |
| Testbench | Can wrap stimulus, checks, file I/O |
| Inside functions | Do not call `task` |

## Memories

```verilog
reg [DATA_WIDTH-1:0] mem [(2**ADDR_WIDTH)-1:0];

always @(posedge clk)
    if (wr_en)
        mem[wr_addr] <= wr_data;
```

Memory rules:

- Describe memory with two-dimensional `reg` arrays.
- Do not clear large RAMs entry by entry in reset.
- Initialization files belong to testbenches or simulation initialization.
- Read/write port behavior must be explicit in the requirement.

## Module Instantiation

```verilog
pulse_counter #
(
    .COUNT_WIDTH (COUNT_WIDTH)
)
pulse_counter_inst
(
    .i_clk   (clk),
    .i_rst_n (rst_n),
    .i_pulse (pulse_in),
    .i_clear (clear_in),
    .o_count (count_out)
);
```

Instantiation rules:

| Item | Rule |
|---|---|
| Port connection | Named ports only |
| Parameter override | Use only `#(...)` |
| Unused output | Leave open as `.o_unused()` |
| Unused input | Tie to an explicit constant |
| Alignment | Align `.`, port names, and expressions by column |
| Instance name | `_inst` suffix preferred |

## generate

```verilog
generate
    genvar bit_index;
    for (bit_index = 0; bit_index < DATA_WIDTH; bit_index = bit_index + 1)
        begin : gen_bit_mask
            assign masked_data[bit_index] = i_data[bit_index] & i_mask[bit_index];
        end
endgenerate
```

generate rules:

- Name every generate block.
- Use `lower_snake_case` for `genvar` names.
- Named blocks may keep `begin : block_name`.
- Do not instantiate recursively.
