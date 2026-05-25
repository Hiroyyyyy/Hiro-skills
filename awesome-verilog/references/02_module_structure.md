# 02 Module Structure

本文件用于约束模块文件的组织结构和层次化写法。
生成新模块时按这里的模板落地，可减少结构性错误。

## 文件组织

| 顺序 | 内容 | 要求 |
|---|---|---|
| 1 | 文件头 | 只包含 `File`、`Date`、`Description`，放在文件第一行 |
| 2 | 编译指令 | `` `timescale``、`` `default_nettype none`` |
| 3 | 模块声明 | 参数块和端口块分开 |
| 4 | 内部声明 | `localparam`、`wire`、`reg`、memory |
| 5 | 连续赋值 | 输出别名和组合连线 |
| 6 | 逻辑块 | `always @*` 和时序 `always` |
| 7 | 子模块例化 | 命名端口连接 |

## 文件头

```verilog
// ---------------------------------------------------------------------
// File : pulse_counter.v
// Date : 2026-05-19
// ---------------------------------------------------------------------
// Description:
//   统计已接收的输入脉冲并输出当前计数值。
// ---------------------------------------------------------------------
```

文件头规则：

- 放在文件第一行。
- 不包含作者字段。
- 不包含变更日志。
- `Date` 使用 `YYYY-MM-DD`。
- `Description` 写模块用途，不写实现细节流水账。

## 编译指令

```verilog
`timescale 1ns / 1ps
`default_nettype none
```

编译指令规则：

- 文件头之后、`module` 之前。
- 不使用 `` `resetall``。

## 模块骨架

```verilog
module pulse_counter #
(
    parameter COUNT_WIDTH = 8  // 计数器位宽
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

模块声明规则：

| 项目 | 规则 |
|---|---|
| 文件和模块 | 默认一文件一个主模块，文件名匹配模块名 |
| 参数块 | `#` 后单独写 `(` 和 `)` |
| 端口块 | 参数块之后单独写 `(` 和 `)` |
| 端口顺序 | 时钟、复位、输入、输出、双向 |
| 端口类型 | 所有端口显式写 `wire` |
| 输出端口 | 使用 `output wire`，不使用 `output reg` |
| 参数注释 | 每个 `parameter` 定义同一行写注释 |

## 参数与常量

| 类型 | 用途 | 示例 |
|---|---|---|
| `parameter` | 用户可配置值 | `DATA_WIDTH` |
| `localparam` | 内部常量、状态编码 | `STATE_IDLE` |
| `integer` | 函数局部计算、循环变量 | `index` |

参数规则：

- 每个 `parameter` 必须有合理默认值。
- 每个 `parameter` 必须有同一行注释。
- 禁止使用 `` `define`` 参数化模块。
- 禁止使用 `defparam` 覆盖参数。
- 派生值优先用 `localparam`。

```verilog
parameter DATA_WIDTH = 8,   // 数据总线位宽
parameter DEPTH      = 16   // 存储条目数
```

## Verilog-2005 位宽函数

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

函数规则：

| 项目 | 规则 |
|---|---|
| 时间行为 | 不包含 `#delay` 或事件控制 |
| 返回方式 | 给函数名赋值，不使用 `return` |
| 副作用 | 不修改模块级寄存器 |
| 调用关系 | 函数中不调用 `task` |

## task 使用边界

| 场景 | 规则 |
|---|---|
| 可综合 RTL | 优先不用 `task` |
| testbench | 可用于封装激励、检查、文件 I/O |
| 函数内部 | 不调用 `task` |

## 存储器

```verilog
reg [DATA_WIDTH-1:0] mem [(2**ADDR_WIDTH)-1:0];

always @(posedge clk)
    if (wr_en)
        mem[wr_addr] <= wr_data;
```

存储器规则：

- 使用二维 `reg` 数组描述 memory。
- 大 RAM 不在复位中逐项清零。
- 初始化文件只用于 testbench 或仿真初始化。
- 读写端口行为要在需求中明确。

## 模块例化

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

例化规则：

| 项目 | 规则 |
|---|---|
| 端口连接 | 只用命名端口 |
| 参数覆盖 | 只用 `#(...)` |
| 未用输出 | 写空连接 `.o_unused()` |
| 未用输入 | 接显式常量 |
| 对齐 | `.`、端口名、表达式按列对齐 |
| 实例名 | 推荐 `_inst` 后缀 |

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

generate 规则：

- 每个 generate block 必须命名。
- `genvar` 名称使用 `lower_snake_case`。
- 命名块可保留 `begin : block_name`。
- 禁止递归例化。
