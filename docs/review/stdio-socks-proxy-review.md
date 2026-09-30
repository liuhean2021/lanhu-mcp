# stdio SOCKS / 本地代理修复 — 提交前审查

- **仓库**：`lanhu-mcp`
- **分支**：`main`（工作区未暂存）
- **审查日期**：2026-09-30
- **原始意图**：修复 Cursor 本地 stdio 调用蓝湖 MCP 时因 SOCKS 代理缺少 `socksio` 导致的 httpx 失败；保持最小改动；文档同步。

---

## 审查前置分析报告

### 🔔 负载告警
- 改动行数：+14 / -1（低负载）
- 安全敏感模块改动：0 行
- 跨模块数：1（启动脚本 + 依赖 + 文档）

### 🔍 Scope Check（范围检查）
Scope（范围）: CLEAN（正常）
Intent（意图）: 让本地 stdio MCP 在 Cursor 注入 SOCKS 代理时不再硬失败，并同步说明。
Delivered（实际改动）: 增加 `socksio` 依赖；`run-stdio.sh` 支持可选清代理；README / LOCAL-STDIO / CHANGELOG 同步。

### 📐 Diff Scope（改动范围）
config / docs（依赖与启动脚本；无 auth/frontend/backend/API/migration）

### 🔐 Secret 扫描
- 命令：`gitleaks git --pre-commit --redact --verbose`
- 范围：未暂存工作区 diff，scanned ~1027 bytes
- 结果：no leaks found

### 🌐 兼容性基线
- 不适用（非前端 UI 改动）

### 🏛️ 架构定性
设计判断: 方案匹配问题
- 改动落在职责所属模块？是（依赖声明 + stdio 启动器 + 文档）
- 依赖方向符合分层？是

---

## 修改总结

### `pyproject.toml`
- **类型**：修改
- **要点**：`dependencies` 增加 `socksio>=1.0.0`
- **目的**：httpx 遇到 `socks://` / `socks5://` 代理时具备 SOCKS 传输能力

### `requirements.txt`
- **类型**：修改
- **要点**：与 `pyproject.toml` 同步增加 `socksio>=1.0.0`
- **目的**：pip / 安装脚本与正式依赖一致，避免只改一侧

### `run-stdio.sh`
- **类型**：修改
- **要点**：当 `LANHU_DISABLE_SYSTEM_PROXY=1` 时 `unset` 继承的 `ALL_PROXY` / `HTTP(S)_PROXY` / `SOCKS_PROXY`（含大小写变体）
- **目的**：Cursor 等 IDE 注入不可用 SOCKS 时，允许强制直连；默认仍尊重系统代理（需已装 socksio）

### `LOCAL-STDIO.md` / `README.md` / `CHANGELOG.md`
- **类型**：修改
- **要点**：说明 SOCKS + `socksio`、可选 `LANHU_DISABLE_SYSTEM_PROXY=1`；CHANGELOG Unreleased 记录
- **目的**：代码与文档一致，使用者知道如何配置与排障

### 仓库外相关（不在本次 lanhu-mcp commit 范围）
- `com.baosight.zhgh.app-repo/.vscode/mcp.json`：HTTP → stdio，并设 `LANHU_DISABLE_SYSTEM_PROXY=1`
- `~/.cursor/mcp.json`：已配 stdio + 同环境变量

---

## 场景矩阵

| 场景 | 预期 | 修改后 | 对抗性检查 |
|------|------|--------|------------|
| 无代理环境，stdio 启动 | 正常启动 | 一致（未设清代理开关） | ✅已查：开关默认 0 |
| 合法 HTTP(S) 代理 | 走代理访问蓝湖 | 一致（默认不清代理） | ✅已查 |
| SOCKS 代理 + 已装 socksio | 走 SOCKS | 依赖已声明；需 `pip install -e .` 生效 | ✅已查 |
| SOCKS 代理 + 缺 socksio（旧 venv） | 原 httpx 硬失败 | 仍可能失败，直到重装依赖；或设清代理开关 | ✅已查 |
| `LANHU_DISABLE_SYSTEM_PROXY=1` | 忽略继承代理直连 | `.sh` 已实现 | ✅已查：空值/未设置不影响 |
| Windows `run-stdio.bat` + 同 env | 文档称 Windows 用 bat | bat **未**实现清代理 | ✅已查：能力缺口 |
| 需代理才能出网却误开清代理 | 可能连不上蓝湖 | 属配置误用；文档为 opt-in | ✅已查 |

---

## 专项清单（适用项）

### 3.8 文档同步
- [x] README / LOCAL-STDIO / CHANGELOG 已同步
- [ ] Windows `run-stdio.bat` 未同步实现（见审查结论）

### 3.4 测试质量
- [ ] 无针对 `LANHU_DISABLE_SYSTEM_PROXY` 的自动化测试（shell 改动可接受，建议手工验证）

### 3.2 / 3.3 / 3.5 / 3.13
- 不适用或跳过（无认证、无热点路径、非前端）

---

## 审查结论

### 亮点
🎉 改动面积极小（6 文件、+14 行），根因（缺 `socksio`）与可选绕过（清代理）分层清晰，默认行为不破坏「需要代理」的既有约定。

### 必须修复（阻塞提交）
无

### 应当修复
🟡 [重要]〔代码审查〕`run-stdio.bat` 未支持 `LANHU_DISABLE_SYSTEM_PROXY`，而 README FAQ / LOCAL-STDIO 将 Windows 启动路径指向 `.bat`，且环境变量说明易被理解为跨平台生效。
  → 建议：在 `run-stdio.bat` 增加等价清理逻辑；或在文档中明确「目前仅 `run-stdio.sh` 支持该变量」。

### 优化建议（不阻塞）
🟢 [建议] 合并后提醒使用者执行一次 `./venv/bin/python -m pip install -e .`（或 `uv pip install -e .`），否则旧 venv 仍无 `socksio`。
🟢 [建议] 提交前用 `LANHU_DISABLE_SYSTEM_PROXY=1` 手工跑一次 `./run-stdio.sh`（Ctrl+C 退出）确认启动横幅正常。
💡 [思路] 仓库外 `app-repo/.vscode/mcp.json` 含本机绝对路径，勿误提交到不相关仓库的共享模板。

### 问题统计
🔴 阻塞: 0 | 🟡 重要: 1 | 🟢 建议: 2 | 💡 思路: 1 | 🎉 亮点: 1

### 审查反思（收尾两问）
🔍 **眼下最没把握的**：未在本审查中实际用 MCP 工具复测「打开蓝湖设计稿」端到端（此前会话曾因 SOCKS 失败；修复后需重载 MCP 再验）。
👁 **可能遗漏的**：`easy-install.sh` / Docker 镜像构建是否会从 `pyproject` 拉到 `socksio`（理论上会）；HTTP 常驻模式是否也受 Cursor SOCKS 影响（本次只修 stdio 启动器）。

### 审查结论
✅ **通过**（无 🔴）— 可提交；建议优先处理 Windows bat 与文档表述一致性（🟡），或合并前在文档写清「仅 sh」。

Self-Test：不触发（改动 < 200 行、非高风险重构）。

---

## 追加：Windows 对齐、NO_PROXY 与多客户端统一配置（2026-09-30）

### 修改总结
- `run-stdio.sh`：开关开启时在 `unset` 之后追加 `export NO_PROXY="*" no_proxy="*"`。
- `run-stdio.bat`：新增等价逻辑（清 `ALL_PROXY`/`HTTP_PROXY`/`HTTPS_PROXY`/`SOCKS_PROXY`，设 `NO_PROXY=*`）。
- `LOCAL-STDIO.md`：第 3 节改为「一份配置适用所有客户端」（Claude Code / Cursor / Windsurf / Gemini / Codex），`env` 只需 `LANHU_USER_ROLE` + `LANHU_DISABLE_SYSTEM_PROXY`。
- `README.md` / `CHANGELOG.md`：示例与说明同步（Linux/Mac 示例改为直接指向脚本，与 Windows 形态一致）。

### 关键发现（本轮实测）
- 干净环境（无任何代理变量、无 `NO_PROXY`）下，httpx 会回落读取 macOS 系统代理（`127.0.0.1:7897`），代理通道数为 2；仅 `unset` 不能保证直连。
- 设置 `NO_PROXY=*` 后 `urllib.getproxies()` 的环境分支非空，不再回落系统/注册表代理，通道数为 0。
- `run-stdio.sh` 用假 `lanhu-mcp` 验证：开关开启时子进程环境仅剩 `NO_PROXY=* no_proxy=*`；开关关闭时环境保持不变。

### 审查结论
- 🔴 阻塞：无
- 🟡 重要：`run-stdio.bat` 未在 Windows 实机验证（本机 macOS）〔代码审查〕
- 🟢 建议：Cursor/Windsurf/Gemini/Codex 的配置路径与格式未实测，文档已标注
- Secret 扫描：`gitleaks git --pre-commit --redact`，scanned ~3.5 KB，no leaks found
- ✅ 通过（未提交）
