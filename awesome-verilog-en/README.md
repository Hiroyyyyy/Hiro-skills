# Awesome Verilog EN

An English Verilog-2005 skill for FPGA development. It generates synthesizable RTL from natural language and supports code review, debugging, and testbench guidance.

## Features

- Derive module names, parameters, and ports from natural-language requirements.
- Generate complete Verilog-2005 `.v` modules.
- Enforce `i_`, `o_`, and `io_` port prefixes.
- Use active-low reset through `i_rst_n` / `rst_n`.
- Do not wrap single statements in `begin` / `end`; wrap only multi-statement blocks with aligned own-line delimiters.
- Enforce spacing: 4-space indentation, spaces around operators, and aligned related declarations.
- Require every `parameter` to have a same-line comment.
- Generate a file header containing only `File`, `Date`, and `Description`.
- Generate compiler directives, module bodies, and final `` `resetall``.
- Review `reg` / `wire` driver relationships.
- Check forward references, multiple drivers, width mismatches, and unused signals.
- Check arithmetic widths, sign extension, truncation, and carry handling.
- Check latch inference, reset coverage, and CDC risks.
- Check valid/ready and AXI handshake stability.
- Define boundaries for functions, tasks, instantiation, and `inout`.
- Apply non-conflicting details from user style files or `.v` templates.
- Provide FSM, counter, shift-register, pipeline, and handshake templates.
- Provide Verilog-2005 testbench templates.

## File Structure

```text
awesome-verilog-en/
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

## How It Works

The skill uses a three-step workflow:

1. **Interface Derivation**: infer the `lower_snake_case` module name, parameters, and ports.
2. **Code Generation**: generate a complete Verilog-2005 module with a simplified header, directives, and RTL.
3. **Self-Check Summary**: check syntax, port naming, reset, widths, drivers, CDC, and latch risks.

The four reference files have separate responsibilities: 01 coding style, 02 module structure, 03 design patterns, and 04 review only.

## Core Coding Conventions

| Item | Convention |
|---|---|
| Language | Verilog-2005 |
| Prohibited | SystemVerilog syntax |
| Input ports | `i_` prefix |
| Output ports | `o_` prefix, `output wire` |
| Bidirectional ports | `io_` prefix |
| Clock port | `i_clk` |
| Reset port | `i_rst_n`, internal `rst_n` |
| Reset polarity | Active low |
| Sequential block | `always @(posedge clk or negedge rst_n)` |
| Combinational block | `always @*` |
| Sequential assignment | Nonblocking `<=` |
| Combinational assignment | Blocking `=` |
| Spacing | 4-space indentation; spaces around operators; align related declarations |
| Parameter comments | Every `parameter` has a same-line comment |
| Output driving | Internal `_reg` signal through `assign` |
| Synchronizer naming | Use concise `_sync` names such as `req_sync` |
| FSM states | `localparam`, `ALL_CAPS` |
| `begin` / `end` | Do not wrap single statements; wrap only multi-statement blocks, own-line and aligned |
| `case` | Must include `default` |
| Number literals | Explicit width |
| Module instantiation | Named port connections |
| CDC | Single-bit synchronizer, multi-bit async FIFO or handshake |
| Handshake | Hold valid until ready, keep data stable during stalls |

## Installation / Usage

Use this folder inside a Claude Code or Codex skill directory.

Recommended location:

```text
~/.codex/skills/awesome-verilog-en/
```

It can also be referenced directly from the current workspace:

```text
Use $awesome-verilog-en to generate a UART transmitter in Verilog-2005.
```

Typical generation request:

```text
Use $awesome-verilog-en to generate a parameterized synchronous FIFO with depth 16, 8-bit data, and a valid/ready interface.
```

Review request:

```text
Use $awesome-verilog-en to review this Verilog module for reset, width, multiple-driver, and CDC risks.
```

Testbench request:

```text
Use $awesome-verilog-en to generate a Verilog-2005 testbench for this counter.
```
