# REA 逆向分析实战教程

> **REA（Reverse Engineer Anything）**：一个 MCP + CLI，让 AI Agent 直接逆向二进制、APK、Electron / Windows 应用。
> 整理自 [morluto/rea](https://github.com/morluto/rea)（MIT 协议，GitHub ⭐ 44.8 万+，npm 包 `rea-agents`）。

---

## 0. 先说清楚：合法使用边界

REA 是**分析工具**，不是"注册机 / 破解器"。本教程只讲一件事：**如何看懂一个程序**——它的结构、逻辑、接口、数据格式、运行时行为。

请只分析你**有权分析**的软件：

- 你自己开发 / 拥有的应用
- 你获得授权测试的目标（渗透测试、众测、CTF）
- 恶意软件样本、漏洞研究样本
- 需要互操作 / 数据迁移的第三方格式

**绕过他人软件的授权与付费保护、制作盗版、篡改他人应用**——不在本教程范围，我也不协助。

---

## 1. REA 是什么

REA 把逆向工程能力（反编译、反汇编、动态插桩、格式解析）封装成 **MCP 工具 + CLI**，让 AI Agent 能"边问边逆"：

- 给它一个二进制 / APK / Electron 应用，它能给出**伪代码、汇编、字符串、符号、调用关系、引用链**
- 每条结论都带 **Evidence（证据）** 和**置信度**（`observed` / `derived` / `inferred`）+ `limitations`（边界说明）
- 既能静态分析，也能跑起来动态观察（进程、UI、Frida、ADB、浏览器）

分析全部**本地**完成，不上传目标文件。

---

## 2. 版本与热度

| 项目 | 值 |
| --- | --- |
| 仓库 | [morluto/rea](https://github.com/morluto/rea) |
| Star | 448,493（Fork 11 万+） |
| 协议 | MIT |
| 最新发布 | `rea-agents-6.8.0`（2026-10-25） |
| npm | `rea-agents@6.8.0` |
| 发版节奏 | 6.x 系列，约每周一发版（6.1 → 6.8 在 2026-10 内完成） |

**选哪个版本？用最新的 `6.8.0`（`npx rea-agents@latest`）。**
旧版本没有留存价值——除非你要复现某个历史分析结果，或上游引擎 API 不兼容。

---

## 3. 能分析什么

| 目标 | REA 产出 | 依赖 |
| --- | --- | --- |
| 原生二进制（ELF / PE / Mach-O） | 伪代码、汇编、字符串、符号、调用与引用 | Hopper / Ghidra / IDA / Kuna |
| APK / Android | 解码后的 Manifest、类/方法、权限、资源 | JADX（REA 内置 headless） |
| .NET 程序集 | 类型/方法/字段/CIL、P/Invoke 边界 | 内置 |
| Electron / JS 应用 | 模块图、依赖、IPC、源码恢复 | 内置（无需原生引擎） |
| 网页 / HAR 抓包 | DOM、网络、源码、WebMCP | 内置（浏览器） |
| 固件 | 分区、文件系统、内核 | Binwalk / Unblob |
| WASM / EVM 字节码 | 导入导出、WAT、函数选择器 | WABT / 本地 carrier |
| Go / Rust 二进制 | 模块、依赖、版本、构建信息 | 内置 |
| 数据库 / 包格式 | SQLite 结构、归档清单、PE 资源 | 内置 |

---

## 4. 环境要求

| 项 | 要求 |
| --- | --- |
| **Node.js** | 22.x（>=22.23.4）/ 24.x（>=24.22.1）/ 26.12.1+。**不要用 23 / 25**，要配 npm |
| **Linux** | 需要 proccs 兼容的 `ps`（进程归属检查） |
| **原生分析引擎**（任选一） | Hopper / Ghidra 12.1（需 JDK 21+）/ IDA / Kuna |
| **Android** | JADX 由 REA 内置；动态分析要 `adb` |
| **Windows** | Ghidra 的 Windows 支持官方标注为 experimental |

---

## 5. 安装

### 5.1 一键 setup（推荐）

```bash
npx rea-agents setup
```

流程：

1. 多选要接入的 Agent（Claude Code / Claude Desktop / Codex / Cursor / Gemini CLI / Windsurf / VS Code / GitHub Copilot CLI …）
2. 会安装 **MCP server** + **工作流说明（skill）**，并对已有配置**做备份**
3. 展示将要改动的路径，**审批后**才写入
4. 完成后**重启 Agent**
5. Claude Desktop 要**完全退出**再审批（运行中的 app 会覆盖注册）

### 5.2 Claude 插件方式

```bash
/plugin marketplace add morluto/rea
/plugin install rea
```

插件自带 skill 和它自己的 `rea` MCP server（不走 `rea setup` 的状态管理）。

### 5.3 手动注册 MCP

`.claude-plugin/plugin.json` 的官方配置：

```json
{
  "mcpServers": {
    "rea": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/scripts/plugin-mcp-launcher.mjs", "6.8.0"]
    }
  }
}
```

### 5.4 诊断与更新

```bash
rea doctor --json    # 诊断（注册、引擎、工具是否就绪）
rea update           # 更新
```

---

## 6. Skill（工作流说明）

`.agents/skills/` 下共 5 个：

| Skill | 用途 |
| --- | --- |
| `reverse-engineer-anything` | **主工作流**（核心，必读） |
| `rea-analysis-optimization` | 分析优化 |
| `rea-tool-design` | 工具设计 |
| `rea-ci-tests-audit` | CI / 测试审计 |
| `rea-changelog-update` | 更新日志 |

> 装 skill 只是**给 Agent 指令**；它**不会**注册 MCP、不会装分析引擎。MCP 和引擎要单独装。

---

## 7. 常用工具与命令

### 7.1 MCP 工具（按场景）

**原生二进制**
`open_binary` → `list_procedures` / `search_procedures` / `analyze_function` / `procedure_pseudo_code` / `procedure_assembly` / `inspect_native_instruction` / `procedure_callees` / `procedure_callers` / `xrefs` / `search_strings` / `list_strings` / `trace_call_path` / `trace_native_values` / `get_call_graph`

**Android**
`inspect_android_package`（解码 Manifest + 类/权限清单）、`inspect_android_class`、`inspect_android_method`、`trace_android_references`、`search_android_classes`、`list_adb_devices` / `install_adb_package` / `start_adb_app` / `capture_adb_screen` / `read_adb_logcat`

**Electron / JS**
`analyze_javascript_application`、`inspect_artifact`、`inspect_electron_page`、`inspect_electron_fuses`、`recover_javascript_sources`、`trace_javascript_semantics`

**动态 / 运行时**
`capture_process_scenario`、`capture_electron_scenario`、`capture_native_ui_scenario`、`capture_browser_scenario`、`observe_native_calls`、`instrument_with_frida`、`observe_javascript_runtime`

**证据查看（大结果必读）**
`inspect_analysis_view`（分页读保留的 Evidence）、`get_evidence_bundle`、`export_evidence_bundle`

**引导工作流 / 文档**
`guided_workflows`（列出 / 渲染 6 个官方工作流）、`search_guidance`、`read_guidance_section`

### 7.2 CLI 速查

```bash
# 列出过程（函数）
rea inspect /abs/path --provider ghidra --json
rea search /abs/path entry --kind procedures --provider ghidra --json

# 函数级分析
rea function PATH PROCEDURE --provider ghidra --json
rea decompile PATH PROCEDURE --provider ghidra --json
rea instructions PATH PROCEDURE --provider ghidra --json
rea xrefs PATH --address=<name-or-addr> --provider ghidra --json

# 引导工作流
rea guided-workflows --action render --workflow-id <id> --arguments-json '{"feature":"..."}'

# 查官方指引
rea guidance --query "关键词"
```

---

## 8. 实战一：逆向分析一个 APK

### 流程

1. **静态结构** — MCP `inspect_android_package`
   - 拿到**解码后的 AndroidManifest**：组件（activity/service/receiver/provider）、权限、`exported`、SDK 版本
   - 类 / 方法清单、资源统计
2. **定位入口** — `inspect_android_class` / `inspect_android_method`
   - 看关键类的方法体（带 smali 回退）
   - `trace_android_references` 查"谁引用了它"
3. **找特征** — `search_strings` / `list_strings`
   - 密钥、URL、协议特征、错误串
4. **看逻辑** — `analyze_function` 出伪代码
5. **动态验证**（可选）
   - `list_adb_devices` → `install_adb_package` → `start_adb_app`
   - `capture_adb_screen` 截图、`read_adb_logcat` 看日志
   - `instrument_with_frida` hook 关键方法
6. **产出** — 结论 + Evidence id；大结果用 `inspect_analysis_view` 分页引用

### 示例思路

> 目标：搞清某个 APK 的网络请求是怎么构造的。
>
> 1. `inspect_android_package` 拿 Manifest → 找到网络相关的 Activity / Service
> 2. `inspect_android_class` 看它的方法 → 定位请求构造函数（如 `buildRequest()`）
> 3. `analyze_function` 看伪代码 → 理解签名算法、加密方式、请求头
> 4. 需要时用 Frida hook 该方法，观察真实入参 / 返回值

---

## 9. 实战二：逆向分析一个 Windows 程序

### 流程

1. **打开** — `open_binary`（PE）
2. **看结构** — `binary_overview`（段、导入表）、`inspect_pe_resources`（资源 / 图标 / 版本信息）
3. **找入口** — `list_procedures`，从 `entry` / 导出函数 / 字符串引用入手
4. **反编译** — `analyze_function` / `procedure_pseudo_code`（伪代码）、`procedure_assembly`（汇编）
5. **追调用** — `trace_call_path` / `get_call_graph` / `xrefs`
6. **抓特征** — `search_strings`（错误信息、URL、密钥）、`search_procedures`（按名字找函数）
7. **动态** — `capture_process_scenario`（跑起来看行为）、`observe_native_calls`（API 断点）、`instrument_with_frida`
8. **.NET 程序** — `inspect_managed_artifact` + `inspect_managed_members` + `verify_managed_native_boundaries`

### 示例思路

> 目标：搞清某个 Windows 工具的配置 / 协议处理逻辑。
>
> 1. `open_binary` → `list_procedures`
> 2. `search_strings` 找 "config" / "license" / "expired" 之类的串 → `xrefs` 定位引用它的函数
> 3. `analyze_function` 看那个函数的伪代码 → 理解它读了哪些配置、走哪条分支
> 4. 需要时 `capture_process_scenario` 跑一下，观察它读的文件 / 注册表 / 网络

---

## 10. 证据与结论规范

REA 的每条结论都带：

- **Evidence id**：可复查、可导出
- **置信度**：`observed`（直接观察）/ `derived`（推导）/ `inferred`（推断）
- **limitations**：明确说明"没验证什么"

实践要点：

- 别把 `inferred` 当事实，需要时补动态验证
- 大结果用 `detail: "summary"` + `inspect_analysis_view` 分页读，**不要**反复全量拉
- 一次性 CLI 输出默认完整；要留存就用 `--evidence_output /path/capture.json`

---

## 11. 常见问题

| 现象 | 处理 |
| --- | --- |
| 工具没出现 / 连接失败 | 读 skill 里的 `references/connection-and-recovery.md`；跑 `rea doctor` |
| Linux 报进程归属 | 装 procps（提供 `ps`） |
| Ghidra 相关报错 | 用 Ghidra 12.1 + JDK 21 |
| Claude Desktop 里没生效 | **完全退出** app 再重新审批 setup |
| Evidence 太大 | 存文件 + 投影摘要（summary view），别内联 |
| 引擎选型 | 静态 JS/Electron 分析**不需要**原生引擎；原生二进制才要 Hopper/Ghidra/IDA/Kuna |

---

## 12. 参考资料

- 官网：<https://rea.tools/> ｜ 指南：<https://rea.tools/guides/>
- 仓库：<https://github.com/morluto/rea> ｜ npm：`rea-agents`
- 上游文档：`docs/getting-started/installation.md`、`docs/guides/android/android-analysis.md`、`docs/guides/native/native-investigation.md`
- 社区：Discord（见仓库 README）
