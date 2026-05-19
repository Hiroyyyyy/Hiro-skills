# 03 Design Patterns

This file collects the design patterns used most often in this skill, including reset/clock, CDC, FSM, handshakes, and testbench templates.
Use it as the default reference when choosing implementation patterns.

## Scope

| Category | Content |
|---|---|
| Reset and clock | `rst_n`, synchronized release, clock-enable |
| CDC | Single-bit sync, pulse sync, multi-bit crossing |
| FSM | State encoding, registered outputs, common structure |
| RTL building blocks | MUX, encoder, counter, shift register |
| Handshake | valid/ready, AXI channel stability |
| Testbench | Verilog-2005 testbench skeleton |

## Reset and Clock

| Item | Rule |
|---|---|
| Main clock port | `i_clk` |
| Main internal clock alias | `clk` |
| Reset port | `i_rst_n` |
| Internal reset alias | `rst_n` |
| Reset polarity | Active low |
| Sequential block | `always @(posedge clk or negedge rst_n)` |
| Reset condition | `if (!rst_n)` |

```verilog
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        begin
            valid_reg <= 1'b0;
            data_reg  <= {DATA_WIDTH{1'b0}};
        end
    else
        begin
            valid_reg <= valid_next;
            data_reg  <= data_next;
        end
```

## Asynchronous Assertion, Synchronous Release

| Case | Pattern |
|---|---|
| `i_rst_n` is asynchronous | Synchronize release per clock domain |
| `i_rst_n` is already synchronized | Directly alias to `rst_n` |
| Multiple clock domains | Generate one reset per domain |

```verilog
reg [1:0] rst_sync;

always @(posedge clk or negedge i_rst_n)
    if (!i_rst_n)
        rst_sync <= 2'b00;
    else
        rst_sync <= {rst_sync[0], 1'b1};

assign rst_n = rst_sync[1];
```

## Clock-Enable

| Prohibited | Recommended |
|---|---|
| `assign clk_gated = clk & i_enable;` | Use `else if (i_enable)` inside the sequential block |

```verilog
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        data_reg <= {DATA_WIDTH{1'b0}};
    else if (i_enable)
        data_reg <= i_data;
```

## CDC

| Crossing Type | Recommended Pattern | Prohibited |
|---|---|---|
| Single-bit level | Two-flop synchronizer in destination domain | Direct combinational crossing |
| One-cycle pulse | Toggle synchronizer | Directly syncing source-domain pulse |
| Multi-bit data stream | Async FIFO | Two-flop sync on each bit |
| Slow configuration | Source stable + req/ack | Sampling a changing bus directly |
| Pointer/state | Gray coding + synchronization | Direct binary multi-bit crossing |

Single-bit level:

```verilog
reg [1:0] req_sync;

always @(posedge clk_dst or negedge rst_dst_n)
    if (!rst_dst_n)
        req_sync <= 2'b00;
    else
        req_sync <= {req_sync[0], req_src};

assign req_dst = req_sync[1];
```

One-cycle pulse:

```verilog
reg pulse_toggle_src;

always @(posedge clk_src or negedge rst_src_n)
    if (!rst_src_n)
        pulse_toggle_src <= 1'b0;
    else if (pulse_src)
        pulse_toggle_src <= ~pulse_toggle_src;

reg [2:0] pulse_sync;

always @(posedge clk_dst or negedge rst_dst_n)
    if (!rst_dst_n)
        pulse_sync <= 3'b000;
    else
        pulse_sync <= {pulse_sync[1:0], pulse_toggle_src};

assign pulse_dst = pulse_sync[2] ^ pulse_sync[1];
```

CDC comments:

- Document every CDC direction: `// CDC: clk_src -> clk_dst`.
- Keep synchronizer names concise: `req_sync`, `pulse_sync`.

## FSM

| Encoding | Use Case |
|---|---|
| one-hot | FPGA default when state count is modest |
| binary | Many states when one-hot is inappropriate |
| Gray | State or pointer crossing a clock domain |

State encoding:

```verilog
localparam [2:0]
    STATE_IDLE = 3'b001,  // Waiting for start
    STATE_RUN  = 3'b010,  // Work in progress
    STATE_DONE = 3'b100;  // Work completed
```

Registered-output FSM snippet:

```verilog
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        state_reg <= STATE_IDLE;
    else
        case (state_reg)
            STATE_IDLE:
                if (i_start)
                    state_reg <= STATE_RUN;
                else
                    state_reg <= STATE_IDLE;
            STATE_RUN:
                if (work_done)
                    state_reg <= STATE_DONE;
                else
                    state_reg <= STATE_RUN;
            default:
                state_reg <= STATE_IDLE;
        endcase

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        valid_reg <= 1'b0;
    else
        begin
            valid_reg <= 1'b0;

            if (state_reg == STATE_DONE)
                valid_reg <= 1'b1;
        end
```

FSM rules:

- Assign the state register in exactly one sequential block.
- Recover illegal states through `default`.
- Register outputs by default to avoid combinational glitches.
- Match state-register width to state-encoding width.

## Common RTL Building Blocks

| Pattern | Recommended Form |
|---|---|
| Simple MUX | Continuous assignment |
| Multi-way MUX | `always @*` + default + `case` |
| Priority encoder | `if` / `else if` order expresses priority |
| Counter | Explicit increment width |
| Shift register | Concatenation expression |
| Arithmetic | Explicit carry, truncation, sign extension |

Counter:

```verilog
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        count_reg <= {COUNT_WIDTH{1'b0}};
    else if (i_clear)
        count_reg <= {COUNT_WIDTH{1'b0}};
    else if (i_enable)
        count_reg <= count_reg + {{COUNT_WIDTH-1{1'b0}}, 1'b1};
```

Arithmetic width:

```verilog
wire [DATA_WIDTH:0]   sum_full;
wire [DATA_WIDTH-1:0] sum_data;
wire                  sum_carry;

assign sum_full  = {1'b0, data_a} + {1'b0, data_b};
assign sum_data  = sum_full[DATA_WIDTH-1:0];
assign sum_carry = sum_full[DATA_WIDTH];
```

## Valid / Ready

| Rule | Requirement |
|---|---|
| valid hold | Once high, hold `valid` until `valid && ready` |
| data stability | Keep data stable when `valid=1` and `ready=0` |
| combinational loop | Avoid mutual combinational dependence between `valid` and `ready` |

```verilog
assign i_ready = !valid_reg || o_ready;
assign o_valid = valid_reg;
assign o_data  = data_reg;

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        begin
            valid_reg <= 1'b0;
            data_reg  <= {DATA_WIDTH{1'b0}};
        end
    else if (i_ready)
        begin
            valid_reg <= i_valid;
            data_reg  <= i_data;
        end
```

## AXI Handshake

| Interface | Check Focus |
|---|---|
| AXI-Stream | Hold `tvalid`; keep `tdata/tkeep/tlast` stable during stalls |
| AXI4-Lite | AW, W, B, AR, R channels handshake independently |
| AXI4-Full | Burst beats are contiguous; `WLAST/RLAST` only on final beat |

AXI rules:

- Master-side `AWVALID` / `ARVALID` must not combinationally depend on matching `READY`.
- Hold `BVALID` / `RVALID` until the matching `READY` accepts it.
- `AWLEN` / `ARLEN` encode beats minus one.

## Testbench

| Part | Content |
|---|---|
| Basic structure | `timescale`, test module, DUT instantiation |
| Clock | `initial` + `forever` |
| Reset | `task apply_reset` |
| Stimulus | One task per transaction |
| Checks | Use `!==` to catch X/Z |
| File I/O | `$readmemh`, `$fopen`, `$fdisplay` |

Clock and reset:

```verilog
localparam CLK_PERIOD = 10;  // Clock period in ns

initial
    begin
        clk = 1'b0;
        forever
            #(CLK_PERIOD / 2) clk = ~clk;
    end

task apply_reset;
    begin
        rst_n = 1'b0;
        repeat (4)
            @(posedge clk);
        rst_n = 1'b1;
        @(posedge clk);
    end
endtask
```

Check task:

```verilog
task expect_count;
    input [7:0] expected;
    if (o_count !== expected)
        begin
            $display("ERROR: expected %h got %h at time %0t", expected, o_count, $time);
            $finish;
        end
endtask
```
