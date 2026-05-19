# Awesome Verilog CN

面向 FPGA 开发的中文 Verilog-2005 Skill，用于从自然语言生成可综合 RTL，并对代码进行审查、调试和测试平台指导。

## 功能列表

- 从自然语言需求推导模块名、参数和端口。
- 生成完整的 Verilog-2005 `.v` 模块。
- 强制使用 `i_`、`o_`、`io_` 端口前缀。
- 使用低有效复位 `i_rst_n` / `rst_n`。
- 强制单语句不包 `begin` / `end`，多语句才包且独占一行对齐。
- 强制 spacing：4 空格缩进、运算符留空格、相关声明按列对齐。
- 强制每个 `parameter` 在定义同一行写注释。
- 生成只包含 `File`、`Date`、`Description` 的文件头。
- 生成编译指令、模块主体和结尾 `` `resetall``。
- 审查 `reg` / `wire` 驱动关系。
- 检查前向引用、多驱动、位宽不匹配和未使用信号。
- 检查算术位宽、符号扩展、截断和进位处理。
- 检查 latch 推断、复位覆盖和 CDC 风险。
- 检查 valid/ready 和 AXI 握手稳定性。
- 明确函数、task、例化和 `inout` 的使用边界。
- 支持读取用户风格文件或 `.v` 模板中的非冲突细节。
- 提供 FSM、计数器、移位寄存器、流水线和握手模板。
- 提供 Verilog-2005 测试平台模板。

## 文件结构

```text
awesome-verilog-cn/
├── SKILL.md
├── README.md
├── agents/
│   └── openai.yaml
└── references/
    ├── 01_coding_style.md
    ├── 02_module_structure.md
    ├── 03_design_patterns.md
    └── 04_review_checklist.md
```

## 工作原理

Skill 使用三步工作流：

1. **接口推导**：根据用户描述推导 `lower_snake_case` 模块名、参数和端口。
2. **代码生成**：生成完整 Verilog-2005 模块，包含简化文件头、编译指令和 RTL 实现。
3. **自检摘要**：逐项检查语法、端口命名、复位、位宽、驱动关系、CDC 和 latch 风险。

四个参考文件保持独立职责：01 负责编码风格，02 负责模块结构，03 负责设计模式，04 专门负责 review。

## 核心编码约定

| 项目 | 约定 |
|---|---|
| 语言 | Verilog-2005 |
| 禁止 | SystemVerilog 语法 |
| 输入端口 | `i_` 前缀 |
| 输出端口 | `o_` 前缀，`output wire` |
| 双向端口 | `io_` 前缀 |
| 时钟端口 | `i_clk` |
| 复位端口 | `i_rst_n`，内部 `rst_n` |
| 复位极性 | 低有效 |
| 时序块 | `always @(posedge clk or negedge rst_n)` |
| 组合块 | `always @*` |
| 时序赋值 | 非阻塞 `<=` |
| 组合赋值 | 阻塞 `=` |
| spacing | 4 空格缩进；运算符两侧留空格；相关声明按列对齐 |
| 参数注释 | 每个 `parameter` 同一行写注释 |
| 输出驱动 | 内部 `_reg` 信号通过 `assign` 驱动 |
| 同步器命名 | 使用 `req_sync` 这类简洁 `_sync` 名称 |
| FSM 状态 | `localparam`，`ALL_CAPS` |
| `begin` / `end` | 单语句不包；多语句才包，且独占一行对齐 |
| `case` | 必须有 `default` |
| 数字字面量 | 显式位宽 |
| 模块例化 | 命名端口连接 |
| CDC | 单比特同步器，多比特异步 FIFO 或握手 |
| 握手 | valid 保持到 ready，stall 时数据稳定 |

## 安装 / 使用说明

在 Claude Code 或 Codex 的 skill 目录中使用本文件夹。

推荐方式：

```text
~/.codex/skills/awesome-verilog-cn/
```

也可以在当前工作区中直接引用：

```text
Use $awesome-verilog-cn to generate a UART transmitter in Verilog-2005.
```

典型请求：

```text
使用 $awesome-verilog-cn 生成一个参数化同步 FIFO，深度 16，数据宽度 8 bit，带 valid/ready 接口。
```

审查请求：

```text
使用 $awesome-verilog-cn 审查这个 Verilog 模块的复位、位宽、多驱动和 CDC 风险。
```

测试平台请求：

```text
使用 $awesome-verilog-cn 为这个计数器生成 Verilog-2005 testbench。
```
