# Stata连接Workbuddy自动配置

> 一个 WorkBuddy 专家包 + 一套通用化技能：在**任意 Windows 机器**上自动打通 Stata MCP，编写 / 运行 / 调试 do 文件，并解读 OLS、面板固定效应、DID 等回归结果。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#许可证)
![Platform](https://img.shields.io/badge/Platform-Windows-lightgrey.svg)
![Stata](https://img.shields.io/badge/Stata-16%20%7C%2017%20%7C%2018%20%7C%2019-1f6f78.svg)
![WorkBuddy](https://img.shields.io/badge/WorkBuddy-Expert-00b4b4.svg)

花名 **Stata 分析助手（小斯塔）**，Agent 型单角色专家，分类 `04-DataAI`。

---

## 这个项目解决什么问题

把 Stata 接进 WorkBuddy 的 MCP 通道，实际会踩到不少文档里没写的坑：环境变量名换过、`--usable` 已弃用、目录白名单会静默回退、MCP 自己开了日志导致 `r(604)`、工具参数名不是直觉里的 `file`……

本项目把**一次真实跑通的排错过程**固化成可复用的专家与技能：

- 所有路径（Stata 安装位置、WorkBuddy 托管 Python、用户目录、项目目录）都在执行时**探测得到**，不绑定任何一台机器。
- 兼容 Stata 16/17/18/19 × SE / MP / BE，兼容不同盘符、不同 Windows 用户名。
- 写 `mcp.json` 时**合并而非覆盖**，不会破坏你已有的其他 MCP 服务器配置。
- 附带 `auto.dta` 最小连接测试与预期数值锚点，用来判断整条链路是否真的通了。

## 目录结构

```
stata-analysis-assistant/
├── README.md                          # 本文件
├── LICENSE                            # MIT
├── .gitignore
├── .codebuddy-plugin/
│   └── plugin.json                    # 专家清单（展示信息、分类、推荐问法）
├── agents/
│   └── stata-analysis-assistant.md    # Agent 人设、工作流程、输出规范
├── skills/
│   └── stata-mcp-setup/
│       └── SKILL.md                   # 核心：跨机型配置 + 排错速查手册
└── avatars/
    ├── expert.svg                     # 矢量头像（纯文本，可直接预览）
    └── README.md                      # 头像规范与 PNG 获取说明
```

## 快速开始

### 用法 A：作为 WorkBuddy 专家安装（推荐）

1. 把本仓库 clone / 解压到本机专家目录：

   ```
   %USERPROFILE%\.workbuddy\plugins\marketplaces\my-experts\plugins\stata-analysis-assistant\
   ```

2. 运行注册命令写入 `marketplace.json`（WorkBuddy 自带脚本）：

   ```bash
   python3 scripts/register_expert.py <expert-dir>
   ```

3. 重启 WorkBuddy，在「专家中心」找到 **Stata连接Workbuddy自动配置** 并开始对话，例如：
   > 帮我在这台电脑上配置 Stata MCP 并跑通连接测试

### 用法 B：只安装技能

把 `skills/stata-mcp-setup/` 整个目录复制到 `~/.workbuddy/skills/` 即可，之后对话里提到「配置 Stata MCP / 跑 do 文件」会自动触发。

### 用法 C：不使用 WorkBuddy，手动配置 Stata MCP

按 `skills/stata-mcp-setup/SKILL.md` 的五步走。核心配置（`~/.workbuddy/mcp.json`，注意**不是** `.workbuddy\.mcp.json`）：

```json
{
  "mcpServers": {
    "stata-mcp": {
      "command": "uvx",
      "args": ["stata-mcp"],
      "env": {
        "STATA_CLI": "<Stata 可执行文件，使用正斜杠>",
        "STATA_MCP__CWD": "<项目工作目录，使用正斜杠>",
        "STATA_MCP__ADDITIONAL_ALLOWED_DIRS": "<同一个项目工作目录>"
      },
      "disabled": false
    }
  }
}
```

要点：环境变量是 `STATA_CLI`（**不是** `STATA_PATH`）；旧版 `--usable` 参数已弃用；路径避免空格与中文；保存后重启客户端并在 MCP 服务器列表里「信任」`stata-mcp`。

## 已踩过的坑（速查）

| 症状 | 根因 | 修复 |
|---|---|---|
| `Access denied: Dofile is outside allowed directories` | `WORKING_DIR` 写入自检失败后静默回退 `~/Documents`，白名单失效 | 同时设置 `STATA_MCP__CWD` 与 `STATA_MCP__ADDITIONAL_ALLOWED_DIRS`，双保险 |
| `r(604) log file already open` | MCP 服务端已打开日志，do 文件里又 `log using` | 在 `log using` 前加 `capture log close` |
| 调用工具报参数不匹配 | 参数名是 `dofile_path`，不是 `file` | 改用 `dofile_path` |
| `doctor` 输出为空 | 无头环境没有真实控制台 | 正常现象；改用 JSON-RPC 客户端把响应落盘验证 |
| 找不到 `uvx` | `Scripts` 目录未加入用户 PATH / 未重启 | 加 PATH 后重启客户端 |
| 中文 / 空格路径引发异常 | Stata 与 MCP 对路径敏感 | 项目目录换成纯 ASCII 路径 |

完整的探测命令、合并脚本、do 文件模板都在 `skills/stata-mcp-setup/SKILL.md`。

## 连接测试基线

用 Stata 自带 `auto.dta` 做最小验证（任何机器结果一致）：

| 指标 | 预期值 |
|---|---|
| 样本量 | 74 个观测 |
| `mpg` 系数 | -49.51222 |
| `mpg` 稳健标准误 | 95.8074 |
| `mpg` 的 p 值 | 0.607 |
| `weight` 系数 | 1.746559（p=0.028） |
| R² | 0.2934 |

命令序列：`sysuse "auto.dta", clear` → `describe` → `summarize price mpg weight` → `reg price mpg weight, vce(robust)`。回归设定是否合理仍需研究者自行判断。

## 跨机型设计

| 需要知道的值 | 获取方式 |
|---|---|
| Stata 可执行文件 | 注册表 `HKLM:\SOFTWARE\StataStata*` → 常见安装目录枚举 → `where.exe StataMP-64` 三级兜底 |
| 托管 Python | `~/.workbuddy/binaries/python/versions/*/python.exe` 取最高版本，回退系统 Python |
| 用户主目录 | `$env:USERPROFILE` / `Path.home()` |
| 项目目录 | 优先数据盘，否则用户主目录下；由用户确认后创建 `scripts/` + `results/` |

## 头像

专家包规范要求 `avatars/expert.png`（512×512 PNG / JPG，≤ 500KB）。本仓库以纯文本形式提供 `avatars/expert.svg` 作为矢量预览；若需要可用的 PNG，见 `avatars/README.md`。

## 许可证

[MIT](LICENSE) — 自由使用、修改、分发，保留版权声明即可。

## 致谢

基于 [stata-mcp](https://pypi.org/project/stata-mcp/) 与 WorkBuddy 的 MCP 机制；连接测试方法参考 Stata 自带 `auto.dta` 的最小验证思路。
