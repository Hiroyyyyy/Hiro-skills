---
name: awesome-verilog-cn
description: 面向 FPGA 的 Verilog-2005 RTL 生成、审查、调试和测试平台指导。用于从自然语言生成可综合 Verilog 模块、推导接口、执行自检，以及审查 FSM、CDC、综合安全性等问题。
---

# FPGA Verilog Skill

## 概述

将自然语言需求转换为面向 FPGA 的可综合 Verilog-2005 RTL。优先保证硬件行为正确、综合安全、接口清晰，生成代码后必须自检再输出。

## 响应组织

根据任务类型只输出需要的章节：

- 生成新模块：`接口推导` → 完整 RTL → `自检摘要`
- RTL 审查：先列问题和位置，再给修复；若用户要求，直接给出修正代码
- 调试解释：先讲硬件行为，再讲可能原因和最小改动
- 涉及 CDC、FSM、握手或算术任务时，说明关键假设和最可能出错的边界情况

## 交互原则

若用户要求与本 Skill 中的规则（命名、复位、格式等）冲突，**先向用户确认**再继续。说明冲突的规则和用户的要求，询问遵循哪一方，不静默覆盖任何一边。

## 强制规则

1. **仅使用 Verilog-2005 语法**
   - 禁止：`logic`、`always_ff`、`always_comb`、`always_latch`、`interface`、`modport`、`unique case`、`priority case`、`typedef enum`、函数中使用 `return`、未命名 `begin...end` 中声明变量

2. **端口使用前缀命名**
   - 输入 `i_`、输出 `o_`、双向 `io_`；禁止使用后缀 `_i`/`_o`/`_io`
   - 时钟端口 `i_clk`，复位端口 `i_rst_n`；内部可别名为 `clk`/`rst_n`

3. **低有效异步复位 `rst_n`**
   - 敏感列表 `always @(posedge clk or negedge rst_n)`，复位判断 `if (!rst_n)`，复位分支在块开头
   - 若异步复位进入，需按各时钟域同步释放

4. **`begin`/`end` 格式**
   - 单语句不包裹，多语句才用 `begin`/`end`
   - `begin`/`end` 各自独占一行，相对控制语句缩进 4 空格，`end` 与 `begin` 对齐

5. **`reg`/`wire` 驱动规则**
   - `always` 驱动用 `reg`，`assign` 驱动用 `wire`；输出端口用 `output wire`，由内部 `_reg` 通过 `assign` 驱动
   - 禁止 `assign` 驱动 `reg`，禁止 `always` 驱动 `wire`，每个信号只有一个驱动源

6. **赋值规则**
   - 时序逻辑用 `<=`，组合逻辑用 `=`，同一个 `always` 块中禁止混用

7. **无占位符代码**
   - 禁止 `// TODO`、`// placeholder`、`// implement later`；每个模块必须完整可综合

8. **无前向引用**
   - 所有信号在首次使用前声明；使用 `` `default_nettype none`` 防止隐式线网

9. **握手协议稳定**
   - `valid` 保持到对应 `ready` 接收；`valid=1` 且 `ready=0` 时数据和旁路信号保持稳定

10. **层次连接显式**
    - 例化只用命名端口，禁止位置连接；参数覆盖只用 `#(...)`，禁止 `defparam`

11. **自检后输出**
    - 生成代码后运行自检（见下方），`[FAIL]` 项修复后再输出

## 工作流

### Step 1：接口推导

从用户需求推导接口，缺失细节时选择保守默认值并说明假设。至少考虑：时钟域、复位源、握手/反压、位宽/符号性、延迟、CDC 边界。

```text
## 接口推导

模块名: <lower_snake_case_name>
参数:
  parameter DATA_WIDTH = 8   // Data bus width in bits
端口:
  input  wire                  i_clk
  input  wire                  i_rst_n
  input  wire [DATA_WIDTH-1:0] i_data
  output wire [DATA_WIDTH-1:0] o_result
```

### Step 2：代码生成

生成完整 `.v` 模块，结构和风格细节参见 `references/02_module_structure.md` 和 `references/01_coding_style.md`。若模块超过约 500 行，建议分子模块或询问用户是否拆分多文件。

### Step 3：自检

```text
## 自检摘要

[PASS/FAIL] 文件头完整（File/Date/Description 已填写）
[PASS/FAIL] 文件结构（resetall / timescale / default_nettype / endmodule / resetall）
[PASS/FAIL] Verilog-2005 语法（无 SystemVerilog 语法）
[PASS/FAIL] 无仿真专用行为（RTL 中无 #delay/$display/$finish/$monitor）
[PASS/FAIL] spacing（缩进、空格、对齐、行尾空白符合规则）
[PASS/FAIL] 参数注释（每个 parameter 都有同一行注释）
[PASS/FAIL] reg/wire 驱动规则
[PASS/FAIL] 无前向引用
[PASS/FAIL] 无多驱动冲突
[PASS/FAIL] 无占位符或空实现
[PASS/FAIL] 位宽匹配
[PASS/FAIL] 算术位宽安全（进位、截断、符号扩展均显式处理）
[PASS/FAIL] 无未使用信号
[PASS/FAIL] 无锁存器推断（always @* 路径完整赋值，case 有 default）
[PASS/FAIL] 复位覆盖（所有寄存器有复位，或说明数据路径例外）
[PASS/FAIL] 端口命名（使用 i_/o_/io_ 前缀）
[PASS/FAIL] begin/end 格式
[PASS/FAIL/N/A] 跨时钟域检查（CDC 信号经过同步器）
[PASS/FAIL/N/A] AXI/valid-ready 握手检查
[PASS/WARN/N/A] 用户自定义风格已应用且未覆盖强制规则
[PASS/FAIL] 禁止构造检查（无 SystemVerilog 语法泄漏）
```

`[FAIL]` 必须修复后再输出；`[WARN]` 必须说明原因；`N/A` 仅用于任务确实不涉及该项。

## 引用路由

按需加载最小相关引用集：

| 主题领域 | 引用文件 | 内容边界 |
|---------|---------|---------|
| 编码风格 | `references/01_coding_style.md` | 命名、spacing、注释、声明、赋值、case、禁止构造 |
| 模块结构 | `references/02_module_structure.md` | 文件头、编译指令、端口/参数、函数/task、存储器、例化、generate |
| 设计模式 | `references/03_design_patterns.md` | 复位、时钟、CDC、FSM、常用 RTL、握手、测试平台 |
| 审查清单 | `references/04_review_checklist.md` | 审查顺序、反模式、错误速查、自检清单 |
