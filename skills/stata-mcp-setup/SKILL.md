---
name: stata-mcp-setup
description: 在任意 Windows 机器上把 Stata MCP（stata-mcp）配置到 WorkBuddy 并跑通连接测试；覆盖 uv 安装、Stata 可执行文件自动探测、mcp.json 合并写入、目录白名单与 r(604) 双重日志等排错。当用户要接入 Stata MCP、运行 Stata do-file、或做 auto.dta 连接测试时使用。
version: 2.0
agent_created: true
---

# Stata MCP 跨机型配置与排错（Windows + WorkBuddy）

> 本技能不绑定任何具体机器：所有路径（Stata 安装位置、WorkBuddy 托管 Python、用户名、项目目录）均在执行时**探测得到**，禁止把某一台机器上探测到的路径硬编码给另一台机器。

## 适用场景

- 用户要在 WorkBuddy 里通过 MCP 调用本机 Stata（任意版本：16/17/18/19 × SE/MP/BE）。
- 需要跑 do-file、做最小连接测试（auto.dta）、或排查 `outside allowed directories` / `r(604)` 等错误。

## 第一步：环境探测（先探测，再动手）

### 1.1 探测 Stata 可执行文件

按优先级探测，拿到后**与用户确认**：

```powershell
# A. 注册表（最可靠，无需猜路径）
Get-ItemProperty "HKLM:\SOFTWARE\StataStata*" , "HKLM:\SOFTWARE\WOW6432Node\StataStata*" -ErrorAction SilentlyContinue |
  Select-Object -ExpandProperty StataEXE

# B. 常见安装目录（按版本号枚举）
Get-ChildItem "C:\Program Files\Stata*", "C:\Program Files (x86)\Stata*", "D:\stata*" -Filter "*.exe" -ErrorAction SilentlyContinue |
  Where-Object { $_.Name -match "Stata(MP|SE|BE|IC)?-?64\.exe|Stata\.exe" } |
  Select-Object -ExpandProperty FullName

# C. PATH 上的 stata 命令
where.exe StataMP-64 2>$null; where.exe StataSE-64 2>$null; where.exe Stata-64 2>$null; where.exe Stata 2>$null
```

预期形如 `D:\stata18\StataMP-64.exe` 或 `C:\Program Files\Stata18\StataMP-64.exe`。记录为 `$StataCLI`。

### 1.2 探测 WorkBuddy 托管 Python（uv 的安装载体）

```powershell
# 取版本号最高的托管 Python
Get-ChildItem "$env:USERPROFILE\.workbuddy\binaries\python\versions\*\python.exe" |
  Sort-Object FullName -Descending | Select-Object -First 1 -ExpandProperty FullName
```

若为空则回退 `where.exe python`（系统 Python）。记录为 `$WBPy`。

### 1.3 其他动态值

| 值 | 获取方式 |
|---|---|
| 用户主目录 | `$env:USERPROFILE` / `Path.home()` |
| mcp.json 路径 | `$env:USERPROFILE\.workbuddy\mcp.json`（**注意：不是 `.workbuddy\.mcp.json`） |
| 项目工作目录 | 默认 `D:\stata_projects`（D 盘存在时），否则 `$env:USERPROFILE\stata_projects`；询问用户后创建 `scripts\` + `results\` |

## 第二步：安装 uv（系统没有时）

```powershell
& $WBPy -m pip install --upgrade uv
# uvx.exe 位于 $env:USERPROFILE\.workbuddy\binaries\python\versions\<版本>\Scripts\uvx.exe
```

把该 `Scripts` 目录加入**用户 PATH**（WorkBuddy 重启后才能 spawn `uvx`）：

```powershell
$dir = "$env:USERPROFILE\.workbuddy\binaries\python\versions\<版本>\Scripts"
$p = [Environment]::GetEnvironmentVariable("Path","User") -split ';'
if ($p -notcontains $dir) { [Environment]::SetEnvironmentVariable("Path", ($p + $dir) -join ';', "User") }
```

## 第三步：写入 mcp.json（合并，不覆盖）

先读现有文件，只合并 `stata-mcp` 条目，保留其他服务器。推荐用 Python 合并：

```python
import json, os
mcp = os.path.join(os.path.expanduser("~"), ".workbuddy", "mcp.json")
cfg = json.load(open(mcp, encoding="utf-8")) if os.path.exists(mcp) else {}
cfg.setdefault("mcpServers", {})["stata-mcp"] = {
    "command": "uvx",
    "args": ["stata-mcp"],
    "env": {
        "STATA_CLI": STATA_CLI,                 # 探测到的 Stata 可执行文件
        "STATA_MCP__CWD": PROJECT_DIR,          # 项目目录（正斜杠）
        "STATA_MCP_CWD": PROJECT_DIR,           # 兼容旧版
        "STATA_MCP__ADDITIONAL_ALLOWED_DIRS": PROJECT_DIR  # 白名单双保险
    },
    "disabled": False
}
json.dump(cfg, open(mcp, "w", encoding="utf-8"), ensure_ascii=False, indent=2)
```

要点：
- 环境变量名是 `STATA_CLI`（**不是** `STATA_PATH`）；旧版 `--usable` 参数已弃用，不要用。
- 路径用**正斜杠**，且不含空格/中文。
- `STATA_MCP__ADDITIONAL_ALLOWED_DIRS` 与 `STATA_MCP__CWD` 同时设——因为 `WORKING_DIR` 写入自检失败会静默回退 `~/Documents` 导致白名单失效，双保险可绕过该回退。

## 第四步：连接测试 do 文件

放入 `<项目目录>\scripts\connection_test.do`：

```
* connection_test.do
clear all
set more off
cd "<项目目录>"
* 关键：capture log close 避免 MCP 已开日志导致 r(604)
capture log close
log using "<项目目录>\results\connection_test.log", text replace
sysuse "auto.dta", clear
describe
summarize price mpg weight
reg price mpg weight, vce(robust)
log close
```

**预期锚点**（Stata 自带数据，任何机器一致）：74 个观测；mpg 系数 -49.51222（稳健 SE 95.8074，p=0.607）；weight 系数 1.746559（p=0.028）；R²=0.2934。

## 第五步：运行与验证

- **直接运行**（批处理）：`& "$StataCLI" /b do "<项目目录>\scripts\connection_test.do"`
- **经 MCP 运行**：工具 `stata_do`，参数：
  ```json
  {"dofile_path": "<项目目录>/scripts/connection_test.do", "enable_smcl": false, "timeout": 120}
  ```
  参数名是 **`dofile_path`**（不是 `file`）。
- **无头沙箱验证法**（stdout 被 Stata 子进程吞掉时）：写一个 Python JSON-RPC 客户端，从 mcp.json 读 env 传给 `uvx stata-mcp` 子进程，依次 `initialize` → `notifications/initialized` → `tools/list` → `tools/call`，把每步响应**写进文件**再读取，不依赖 stdout 回传。

## 常见错误速查

| 症状 | 根因 | 修复 |
|---|---|---|
| `Access denied: Dofile is outside allowed directories` | 目录白名单未放行（WORKING_DIR 回退 `~/Documents`） | mcp.json 加 `STATA_MCP__ADDITIONAL_ALLOWED_DIRS`；确认目录真实存在且可写 |
| `r(604) log file already open` | MCP 已开日志，do 文件又 `log using` | `log using` 前加 `capture log close` |
| doctor 回显被吞 | 无头环境无控制台 | 正常现象；改用落盘法验证 |
| 找不到 `uvx` | Scripts 未入 PATH 或未重启 | 检查 PATH 并重启 WorkBuddy |
| 参数报错 mismatch | 用了 `file` 而非 `dofile_path` | 改参数名 |

## 收尾

交付物：mcp.json（合并后）、`scripts\connection_test.do`、`results\connection_test.log`。提醒用户：**重启 WorkBuddy → 插件 → MCP 服务器 → 信任 `stata-mcp`** 后才激活。连接异常时依次排查：uvx 在 PATH、STATA_CLI 路径、JSON 转义、项目目录存在且可写。
