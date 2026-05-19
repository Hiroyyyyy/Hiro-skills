# 01 Coding Style

本文件用于统一本 skill 的编码风格和书写习惯。
生成或审查代码前，可先用它快速对齐命名、spacing 与注释规则。

## 覆盖范围

| 类别 | 内容 |
|---|---|
| 语言边界 | Verilog-2005、禁止构造、数字字面量 |
| 命名 | 模块、实例、端口、寄存器、同步器 |
| spacing | 缩进、空格、对齐、空行 |
| 注释 | 参数注释、关键意图注释、文件头描述 |
| 代码形态 | `begin/end`、`reg/wire`、赋值、`case` |

## 语言边界

- 只使用 Verilog-2005。
- 所有信号必须在首次使用前声明。
- 保留 `` `default_nettype none``，防止隐式线网。
- 禁止在未命名 `begin...end` 块中声明 `reg` 或 `wire`。

| 禁止构造 | 替代方式 |
|---|---|
| `logic` | 按驱动方式使用 `reg` 或 `wire` |
| `always_ff` | `always @(posedge clk or negedge rst_n)` |
| `always_comb` | `always @*` |
| `always_latch` | 完整赋值，避免 latch |
| `interface` / `modport` | 展开为普通端口 |
| `typedef enum` | `localparam` 状态编码 |
| `unique case` / `priority case` | `case` + `default` |
| `'0` | `{WIDTH{1'b0}}` |

## 命名规范

| 对象 | 规则 | 示例 |
|---|---|---|
| 模块 | `lower_snake_case` | `pulse_counter` |
| 实例 | `lower_snake_case`，推荐 `_inst` | `pulse_counter_inst` |
| 普通信号 | `lower_snake_case` | `data_valid` |
| `parameter` / `localparam` | `ALL_CAPS` | `DATA_WIDTH` |
| 输入端口 | `i_` 前缀 | `i_valid` |
| 输出端口 | `o_` 前缀 | `o_ready` |
| 双向端口 | `io_` 前缀 | `io_sda` |
| 寄存器当前值 | `_reg` 后缀 | `valid_reg` |
| 组合下一值 | `_next` 后缀 | `valid_next` |
| 同步器 | `_sync` 后缀 | `req_sync` |
| 低有效信号 | `_n` 后缀 | `rst_n` |

信号名简洁性：

- 不堆叠多个后缀。
- 同步器移位寄存器直接命名为 `req_sync`，不要写成 `req_sync_reg`。
- 端口前缀已经表达方向，不再使用 `_i`、`_o`、`_io` 后缀。
- 名字要表达硬件含义，避免 `tmp1`、`foo`、`data2` 这类弱语义名字。

## Spacing

| 项目 | 规则 |
|---|---|
| 缩进 | 4 个空格 |
| Tab | 禁止 |
| 行尾空白 | 禁止 |
| 最大行宽 | 100 字符 |
| 逗号 | 逗号后 1 个空格 |
| 二元运算符 | 两侧各 1 个空格 |
| 一元运算符 | 运算符和操作数之间不加空格 |
| 函数/task 调用 | 名称和 `(` 之间不加空格 |
| 连续声明 | 类型、位宽、名字按列对齐 |
| 连续 `assign` | 左值和 `=` 按列对齐 |
| 空行 | 逻辑段之间保留 1 个空行 |

示例：

```verilog
wire [DATA_WIDTH-1:0] sum_data;
wire                  sum_valid;

assign sum_data  = data_a + data_b;
assign sum_valid = i_valid & o_ready;
```

## 注释规则

| 对象 | 要求 |
|---|---|
| 文件头 `Description` | 说明模块用途和关键行为 |
| `parameter` | 每个参数必须在定义同一行写注释 |
| `localparam` | 状态、常量、编码需要简短注释 |
| CDC 信号 | 注明跨域方向 |
| 非显然逻辑 | 注释说明硬件意图，不重复代码字面含义 |

参数注释示例：

```verilog
parameter DATA_WIDTH = 8,   // Data bus width in bits
parameter DEPTH      = 16   // Number of storage entries
```

## begin / end

| 情况 | 规则 |
|---|---|
| 控制体只有 1 条语句 | 不使用 `begin/end` |
| 控制体有多条语句 | 使用 `begin/end` |
| `begin` / `end` | 各自独占一行 |
| `begin` 缩进 | 比控制语句多 4 个空格 |
| `end` 对齐 | 与对应 `begin` 对齐 |
| 命名 `generate` | 允许 `begin : block_name` |

单语句：

```verilog
if (enable)
    data_next = i_data;
else
    data_next = data_reg;
```

多语句：

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

## reg / wire 与赋值

| 驱动方式 | 声明类型 | 赋值类型 |
|---|---|---|
| `always @(posedge ...)` | `reg` | 非阻塞 `<=` |
| `always @*` | `reg` | 阻塞 `=` |
| `assign` | `wire` | 连续赋值 |
| 输出端口 | `output wire` | 由内部信号 `assign` 驱动 |

规则：

- 禁止用 `assign` 驱动 `reg`。
- 禁止用 `always` 驱动 `wire`。
- 同一个 `always` 块中禁止混用 `=` 和 `<=`。
- 同一个信号只能有一个驱动源。

输出端口模式：

```verilog
output wire o_valid;

reg valid_reg;

assign o_valid = valid_reg;
```

## case

| 项目 | 规则 |
|---|---|
| 精确匹配 | 使用 `case` |
| 默认分支 | 必须写 `default` |
| 禁止 | `casex`、`full_case`、`parallel_case` |
| 单语句 case item | 不包 `begin/end` |
| 多语句 case item | 使用 `begin/end` |

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

## 数字字面量

| 情况 | 规则 | 示例 |
|---|---|---|
| 常量 | 显式位宽 | `8'd16` |
| 单 bit | 显式位宽 | `1'b0` |
| 参数化清零 | 复制操作符 | `{DATA_WIDTH{1'b0}}` |
| 长常量 | 可用下划线分组 | `16'h12_ab` |
| 窄转宽 | 显式补零或符号扩展 | `{8'd0, byte_data}` |

## 禁止项速查

| 禁止项 | 原因 |
|---|---|
| 占位符代码 | 模块必须完整 |
| 隐式线网 | 易隐藏拼写错误 |
| 前向引用 | 可读性差且容易报错 |
| 多驱动 | 硬件含义不明确 |
| 门控时钟 | FPGA 时钟网络不应由组合逻辑改写 |
| 未同步 CDC | 可能产生亚稳态 |
| `#delay` in RTL | 不可综合 |
| `$display` / `$finish` in RTL | 只用于 testbench 或参数检查 |
