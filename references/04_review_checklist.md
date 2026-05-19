# 04 Review Checklist

本文件用于执行输出前 review。
它提供检查顺序、反模式和自检清单，帮助结果稳定可交付。

## Review 顺序

| 顺序 | 检查项 | 重点 |
|---|---|---|
| 1 | 语言 | 仅 Verilog-2005，无 SystemVerilog 语法和其他禁止构造 |
| 2 | 文件结构 | 文件头、编译指令、结尾 `` `resetall`` |
| 3 | spacing | 缩进、空格、对齐、行尾空白 |
| 4 | 参数注释 | 每个 `parameter` 都有同一行注释 |
| 5 | 端口 | `i_`、`o_`、`io_` 前缀，输出为 `output wire` |
| 6 | 声明 | 所有信号先声明后使用 |
| 7 | 驱动 | `reg/wire` 驱动关系、单驱动 |
| 8 | 组合逻辑 | 默认值、完整路径、`case default` |
| 9 | 时序逻辑 | 复位覆盖、非阻塞赋值、单语句块风格 |
| 10 | CDC | 跨域信号同步 |
| 11 | 握手 | valid/ready、AXI 通道稳定 |
| 12 | 完整性 | 无占位符、无未使用信号、无空实现 |

## 反模式：组合逻辑环路

| 问题 | 后果 | 修复 |
|---|---|---|
| 组合信号依赖自身 | 硬件含义不稳定 | 使用寄存器或明确下一值 |

错误：

```verilog
assign ready_next = ready_next & i_enable;
```

正确：

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

## 反模式：无意 latch

| 问题 | 后果 | 修复 |
|---|---|---|
| 组合块某些路径未赋值 | 推断 latch | 在块顶部设置默认值 |

错误：

```verilog
always @*
    if (i_enable)
        data_next = i_data;
```

正确：

```verilog
always @*
    begin
        data_next = data_reg;

        if (i_enable)
            data_next = i_data;
    end
```

## 反模式：多驱动输出

| 问题 | 后果 | 修复 |
|---|---|---|
| 输出同时被 `assign` 和 `always` 驱动 | 多驱动冲突 | 输出由内部 `_reg` 统一 `assign` |

错误：

```verilog
assign o_valid = valid_reg;

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        o_valid <= 1'b0;
```

正确：

```verilog
reg valid_reg;

assign o_valid = valid_reg;

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        valid_reg <= 1'b0;
    else
        valid_reg <= valid_next;
```

## 反模式：显式组合敏感列表

| 问题 | 后果 | 修复 |
|---|---|---|
| `always @(a or b)` 漏掉输入 | 行为不一致 | 使用 `always @*` |

错误：

```verilog
always @(a or b)
    y = a & b & c;
```

正确：

```verilog
always @*
    y = a & b & c;
```

## 常见错误速查

| 错误现象 | 根因 | 修复 |
|---|---|---|
| `reg` 被连续赋值驱动 | `assign` 驱动了 `reg` | 改为 `wire`，或改到 `always` 中驱动 |
| 找不到信号 | 前向引用或隐式线网 | 在首次使用前声明 |
| 未命名块声明报错 | 在未命名 `begin...end` 内声明变量 | 移到模块级或命名块外 |
| 多驱动 | 同一信号由多个源驱动 | 保留一个驱动源 |
| latch 推断 | `always @*` 某些路径未赋值 | 顶部设置默认值 |
| 参数缺少说明 | `parameter` 无注释 | 在定义同一行补注释 |
| spacing 混乱 | 缩进、空格或对齐不一致 | 按 4 空格和列对齐整理 |
| valid 毛刺 | ready 接收前撤销 valid | 保持到 `valid && ready` |
| AXI 响应丢失 | `BVALID` / `RVALID` 未保持 | 保持到对应 ready 接收 |
| CDC 亚稳态 | 跨域信号未同步 | 单比特同步器，多比特 FIFO 或握手 |
| 位宽警告 | 连接或运算宽度不匹配 | 显式截断、扩展或补零 |
| 有符号比较异常 | 一侧为 unsigned 或宽度不同 | 显式声明 signed，并扩展到相同宽度 |
| 位置例化误连 | 端口顺序变化或人工错位 | 使用命名端口例化 |
| RTL 中混入仿真控制 | 使用 `force`、`release`、`fork`、`join` | 移到 testbench 或重写为硬件逻辑 |

## 自检清单

```text
## 自检摘要

[PASS/FAIL] 文件头完整（File/Date/Description 均已填写）
[PASS/FAIL] 文件结构（resetall / timescale / default_nettype / endmodule / resetall）
[PASS/FAIL] Verilog-2005 语法（无 SystemVerilog 语法）
[PASS/FAIL] 无仿真专用行为（RTL 中无 #delay/$display/$finish/$monitor）
[PASS/FAIL] spacing（缩进、空格、对齐、行尾空白符合规则）
[PASS/FAIL] 参数注释（每个 parameter 均有同一行注释）
[PASS/FAIL] reg/wire 驱动规则（always 驱动用 reg，assign 驱动用 wire）
[PASS/FAIL] 无前向引用（所有信号在使用前已声明）
[PASS/FAIL] 无多驱动冲突（每个信号只有一个驱动源）
[PASS/FAIL] 无占位符或空实现
[PASS/FAIL] 位宽匹配（所有赋值与端口连接左右宽度一致）
[PASS/FAIL] 算术位宽安全（进位、截断、符号扩展均显式处理）
[PASS/FAIL] 无未使用信号
[PASS/FAIL] 无锁存器推断（所有 always @* 路径完整赋值，case 有 default）
[PASS/FAIL] 复位覆盖（所有寄存器都有复位，或说明数据路径例外）
[PASS/FAIL] 端口命名（使用 i_/o_/io_ 前缀，非后缀）
[PASS/FAIL] begin/end 格式（单语句不包，多语句才包；begin/end 独占一行并对齐）
[PASS/FAIL/N/A] 跨时钟域检查（CDC 信号经过同步器）
[PASS/FAIL/N/A] AXI/valid-ready 握手检查（valid 保持、数据稳定、响应不丢失）
[PASS/WARN/N/A] 用户自定义风格已应用且未覆盖强制规则
[PASS/FAIL] 禁止构造检查（无 SystemVerilog 语法泄漏）
```

处理规则：

- 任一项为 `[FAIL]` 时，先修复再输出最终代码。
- `[WARN]` 必须说明原因。
- `N/A` 只用于任务确实不涉及该项时。
