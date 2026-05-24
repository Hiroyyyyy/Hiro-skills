# 03 Design Patterns

本文件汇总本 skill 常用设计模式，包括复位时钟、CDC、FSM、握手和 testbench 模板。
遇到实现选择时优先查这里。

## 覆盖范围

| 类别 | 内容 |
|---|---|
| 复位与时钟 | `rst_n`、同步释放、clock-enable |
| CDC | 单比特同步、脉冲同步、多比特跨域 |
| FSM | 状态编码、寄存输出、常见结构 |
| RTL 基础块 | MUX、编码器、计数器、移位寄存器 |
| 握手 | valid/ready、AXI 通道稳定性 |
| testbench | Verilog-2005 测试平台骨架 |

## 复位与时钟

| 项目 | 规则 |
|---|---|
| 主时钟端口 | `i_clk` |
| 主时钟内部别名 | `clk` |
| 复位端口 | `i_rst_n` |
| 复位内部别名 | `rst_n` |
| 复位极性 | 低有效 |
| 时序块 | `always @(posedge clk or negedge rst_n)` |
| 复位判断 | `if (!rst_n)` |

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

## 异步置位、同步释放

| 场景 | 写法 |
|---|---|
| `i_rst_n` 异步输入 | 每个时钟域同步释放 |
| `i_rst_n` 已同步 | 可直接别名为 `rst_n` |
| 多时钟域 | 每个域单独生成本域 `rst_*_n` |

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

| 禁止 | 推荐 |
|---|---|
| `assign clk_gated = clk & i_enable;` | 在时序块中使用 `else if (i_enable)` |

```verilog
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        data_reg <= {DATA_WIDTH{1'b0}};
    else if (i_enable)
        data_reg <= i_data;
```

## CDC

| 跨域类型 | 推荐模式 | 禁止 |
|---|---|---|
| 单比特电平 | 目标域双触发器同步 | 直接组合跨域 |
| 单周期脉冲 | toggle 同步 | 直接同步源域脉冲 |
| 多比特数据流 | 异步 FIFO | 多 bit 分别打两拍 |
| 慢速配置 | 源域保持稳定 + req/ack | 目标域直接采样变化中总线 |
| 指针/状态 | Gray 编码 + 同步 | 二进制多 bit 直接跨域 |

单比特电平：

```verilog
reg [1:0] req_sync;

always @(posedge clk_dst or negedge rst_dst_n)
    if (!rst_dst_n)
        req_sync <= 2'b00;
    else
        req_sync <= {req_sync[0], req_src};

assign req_dst = req_sync[1];
```

单周期脉冲：

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

CDC 注释：

- 每个 CDC 附近写方向：`// 跨时钟域: clk_src -> clk_dst`。
- 同步器命名保持简洁：`req_sync`、`pulse_sync`。

## FSM

| 编码 | 使用场景 |
|---|---|
| one-hot | FPGA 默认，状态数不大 |
| binary | 状态数较多且 one-hot 不合适 |
| Gray | 状态或指针跨时钟域 |

状态编码：

```verilog
localparam [2:0]
    STATE_IDLE = 3'b001,  // 等待启动
    STATE_RUN  = 3'b010,  // 运行中
    STATE_DONE = 3'b100;  // 工作完成
```

寄存输出 FSM 片段：

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

FSM 规则：

- 状态寄存器只在一个时序块中赋值。
- 非法状态通过 `default` 回到安全状态。
- 输出默认寄存，避免组合毛刺。
- 状态位宽必须匹配状态编码位宽。

## 常用 RTL 基础块

| 模式 | 推荐写法 |
|---|---|
| 简单 MUX | 连续赋值 |
| 多路 MUX | `always @*` + 默认值 + `case` |
| 优先级编码器 | 按 `if` / `else if` 顺序表达优先级 |
| 计数器 | 显式加 1 位宽 |
| 移位寄存器 | 拼接表达式 |
| 算术 | 显式处理进位、截断、符号扩展 |

计数器：

```verilog
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        count_reg <= {COUNT_WIDTH{1'b0}};
    else if (i_clear)
        count_reg <= {COUNT_WIDTH{1'b0}};
    else if (i_enable)
        count_reg <= count_reg + {{COUNT_WIDTH-1{1'b0}}, 1'b1};
```

算术位宽：

```verilog
wire [DATA_WIDTH:0]   sum_full;
wire [DATA_WIDTH-1:0] sum_data;
wire                  sum_carry;

assign sum_full  = {1'b0, data_a} + {1'b0, data_b};
assign sum_data  = sum_full[DATA_WIDTH-1:0];
assign sum_carry = sum_full[DATA_WIDTH];
```

## Valid / Ready

| 规则 | 要求 |
|---|---|
| valid 保持 | `valid` 拉高后保持到 `valid && ready` |
| 数据稳定 | `valid=1` 且 `ready=0` 时数据不变 |
| 组合环路 | 避免 `valid` 和 `ready` 互相组合依赖 |

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

## AXI 握手

| 接口 | 检查重点 |
|---|---|
| AXI-Stream | `tvalid` 保持；`tdata/tkeep/tlast` stall 时稳定 |
| AXI4-Lite | AW、W、B、AR、R 通道独立握手 |
| AXI4-Full | burst beat 连续；`WLAST/RLAST` 只在最后一拍 |

AXI 规则：

- 主设备 `AWVALID` / `ARVALID` 不组合依赖对应 `READY`。
- `BVALID` / `RVALID` 保持到对应 `READY` 接收。
- `AWLEN` / `ARLEN` 表示 beat 数减一。

## Testbench

| 部分 | 内容 |
|---|---|
| 基础结构 | `timescale`、测试模块、DUT 例化 |
| 时钟 | `initial` + `forever` |
| 复位 | `task apply_reset` |
| 激励 | 每个事务一个 task |
| 检查 | 使用 `!==` 捕获 X/Z |
| 文件 I/O | `$readmemh`、`$fopen`、`$fdisplay` |

时钟和复位：

```verilog
localparam CLK_PERIOD = 10;  // 时钟周期，单位 ns

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

检查 task：

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
