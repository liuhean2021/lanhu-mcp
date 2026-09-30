# 纯本地按需调用（免 Docker，stdio）

适用于不想使用 Docker 的场景：只需要 Git 和 Python，不需要 Docker，也不需要常驻服务。MCP 客户端（Claude Code、Cursor 等）每次连接时自动拉起 `run-stdio.sh`，会话结束即退出。

## 平台验证情况

| 平台 | 状态 |
|---|---|
| macOS Intel | 已验证 |
| macOS Apple 芯片（M 系列） | 按脚本预期支持，未实测（`cryptography` 有预编译包，一般无需下方 Intel 的特殊处理） |
| Windows | 按脚本预期支持，未实测（使用 `easy-install.bat` / `run-stdio.bat`） |

在其他平台跑通或遇到问题，欢迎回来更新本表。

## 前置要求

- Git
- Python 3.10–3.13（推荐 3.12；macOS 自带 3.9 不满足）
- 能访问蓝湖，并已登录蓝湖网页版（用于获取 Cookie）

## 1. 克隆并安装

```bash
git clone git@github.com:liuhean2021/lanhu-mcp.git
cd lanhu-mcp
bash easy-install.sh        # Windows: easy-install.bat
```

`easy-install.sh` 会创建 `venv`、安装依赖和 Chromium、生成 `.env` 并引导填写 `LANHU_COOKIE`。
到最后一步「是否现在启动服务？」**输入 `n`**（那是常驻 HTTP 模式，按需调用不需要）。

> 仅 Intel Mac：若安装时报 `cryptography` 编译失败：
> `./venv/bin/python -m pip install --only-binary cryptography -e .`，再执行
> `./venv/bin/python -m playwright install chromium`。

## 2. 验证能启动

```bash
./run-stdio.sh     # Windows: run-stdio.bat
```

看到 fastmcp 启动横幅即正常，`Ctrl+C` 退出。

## 3. 注册到 MCP 客户端

所有客户端用**同一份内容**（个别客户端字段名不同，见下）：`command` 指向项目里的启动脚本，`env` 只放两项。先在项目目录用 `pwd`（Windows 用 `cd`）得到绝对路径，替换 `<ABS_PATH>`。

```json
{
  "mcpServers": {
    "lanhu": {
      "command": "<ABS_PATH>/run-stdio.sh",
      "args": [],
      "env": {
        "LANHU_USER_ROLE": "Developer",
        "LANHU_DISABLE_SYSTEM_PROXY": "1"
      }
    }
  }
}
```

- Windows 把 `command` 换成 `<ABS_PATH>\run-stdio.bat`。
- `LANHU_DISABLE_SYSTEM_PROXY=1`：让服务直连蓝湖，不走任何代理（环境变量代理和系统代理都会绕过）。需要走代理时删掉这一项。
- 可选 `LANHU_USER_NAME`：用于协作追踪和 @提醒。

把上面的 `lanhu` 条目放到对应客户端的配置文件里：

| 客户端 | 配置文件 |
|---|---|
| Claude Code | `~/.claude.json` 顶层 `mcpServers`（全局级） |
| Cursor | `~/.cursor/mcp.json` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| Gemini CLI | `~/.gemini/settings.json` |
| Codex CLI | `~/.codex/config.toml`（格式见下） |
| opencode | `~/.config/opencode/opencode.json`（`mcp` 段，格式见下） |

> Claude Code、Cursor、opencode 为作者日常在用的配置；Windsurf、Gemini CLI、Codex 的路径与格式按各家已知约定整理，未实测。

Codex 用 TOML，内容相同：

```toml
[mcp_servers.lanhu]
command = "<ABS_PATH>/run-stdio.sh"
args = []
env = { LANHU_USER_ROLE = "Developer", LANHU_DISABLE_SYSTEM_PROXY = "1" }
```

opencode 的字段名不同（`command` 是数组，环境变量叫 `environment`，需要 `type: local`），内容相同：

```json
{
  "mcp": {
    "lanhu": {
      "type": "local",
      "command": ["<ABS_PATH>/run-stdio.sh"],
      "environment": {
        "LANHU_USER_ROLE": "Developer",
        "LANHU_DISABLE_SYSTEM_PROXY": "1"
      },
      "enabled": true
    }
  }
}
```

Claude Code 也可以用命令注册（等价于编辑配置文件）：

```bash
claude mcp add lanhu -s user -e LANHU_USER_ROLE=Developer -e LANHU_DISABLE_SYSTEM_PROXY=1 -- <ABS_PATH>/run-stdio.sh
```

提示：
- 配置文件里可能还有其他服务的密钥，只改 `lanhu` 一项，分享或截图前务必脱敏。
- 配置里是本机绝对路径，不适合写进随仓库提交的 `.mcp.json`。
- Windows 若客户端无法直接拉起 `.bat`，把命令改为 `cmd /c <ABS_PATH>\run-stdio.bat`（未实测）；项目建议放在不含中文和空格的路径，Python 需勾选 Add to PATH。
- 注册后重启会话，或在客户端里重连 MCP。

## 注意事项

- 不要移动项目目录或删除 `venv`，否则需要重新注册路径 / 重新安装。
- Cookie 过期后，重新获取并更新项目根目录的 `.env`（见 [GET-COOKIE-TUTORIAL.md](GET-COOKIE-TUTORIAL.md)）。`.env` 已被 git 忽略，不要提交。
- 代理：默认沿用环境里的代理（已依赖 `socksio`，SOCKS 可用）；配置了 `LANHU_DISABLE_SYSTEM_PROXY=1` 则一律直连。
- 更新代码后（`git pull`）如依赖有变化，执行 `./venv/bin/python -m pip install -e .`。
