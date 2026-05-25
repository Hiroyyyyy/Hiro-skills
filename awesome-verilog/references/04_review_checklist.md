# 04 Review Checklist

本文件用于输出前的结构化质量门。包含分组自检清单和反模式速查。

## 自检清单

按强制规则的维度分组检查。

处理规则：

- `[FAIL]` 必须修复后再输出最终代码
- `[WARN]` 必须说明原因
- `N/A` 仅用于任务确实不涉及该项

### 文件结构

```text
[PASS/FAIL] 文件头完整（File/Date/Description 已填写）
[PASS/FAIL] 编译指令完整（timescale / default_nettype）
```

### 语法边界

```text
[PASS/FAIL] Verilog-2005 语法（无 SystemVerilog 语法泄漏）
[PASS/FAIL] RTL 中无 initial 块（testbench 除外）
[PASS/FAIL] RTL 中无 #delay / $display / $finish / $monitor（testbench 除外）
[PASS/FAIL] 无占位符或空实现
```

### 命名与格式

```text
[PASS/FAIL] 端口命名（使用 i_/o_/io_ 前缀）
[PASS/FAIL] begin/end 格式（单语句不包，多语句才包；独占一行并对齐）
[PASS/FAIL] spacing（缩进、空格、对齐、行尾空白符合规则）
[PASS/FAIL] 参数注释（每个 parameter 均有同一行注释）
[PASS/FAIL] 注释使用中文（仅专用术语/变量/公式符号可保留英文）
[PASS/FAIL] 段分隔注释顶格书写
```

### 信号与赋值

```text
[PASS/FAIL] reg/wire 驱动规则（always 驱动用 reg，assign 驱动用 wire）
[PASS/FAIL] 无前向引用（所有信号在使用前已声明）
[PASS/FAIL] 无多驱动冲突（每个信号只有一个驱动源）
[PASS/FAIL] 位宽匹配（所有赋值与端口连接左右宽度一致）
[PASS/FAIL] 算术位宽安全（进位、截断、符号扩展均显式处理）
[PASS/FAIL] 无未使用信号
```

### 设计约束

```text
[PASS/FAIL] 复位覆盖（所有寄存器都有复位，或说明数据路径例外）
[PASS/FAIL] 无锁存器推断（所有 always @* 路径完整赋值，case 有 default）
[PASS/FAIL/N/A] 跨时钟域检查（CDC 信号经过同步器）
[PASS/FAIL/N/A] AXI/valid-ready 握手检查（valid 保持、数据稳定、响应不丢失）
```

### 附加

```text
[PASS/WARN/N/A] 用户自定义风格已应用且未覆盖强制规则
```

## 反模式速查

### 反模式：组合逻辑环路

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

### 反模式：无意 latch

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

### 反模式：多驱动输出

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

### 反模式：显式组合敏感列表

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

### 诊断速查

综合器或仿真中出现的实际症状，按现象定位根因：

| 错误现象 | 根因 | 修复 |
|---|---|---|
| 未命名块声明报错 | 在未命名 `begin...end` 内声明变量 | 移到模块级或命名块外 |
| valid 毛刺 | ready 接收前撤销 valid | 保持到 `valid && ready` |
| AXI 响应丢失 | `BVALID` / `RVALID` 未保持 | 保持到对应 ready 接收 |
| CDC 亚稳态 | 跨域信号未同步 | 单比特同步器，多比特 FIFO 或握手 |
| 位宽警告 | 连接或运算宽度不匹配 | 显式截断、扩展或补零 |
| 有符号比较异常 | 一侧为 unsigned 或宽度不同 | 显式声明 signed，并扩展到相同宽度 |
| RTL 中混入仿真控制 | 使用 `force`、`release`、`fork`、`join` | 移到 testbench 或重写为硬件逻辑 |
