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

先在项目目录用 `pwd`（Windows 用 `cd`）得到绝对路径，替换下面的 `<ABS_PATH>`。

**Claude Code（macOS / Linux）：**

```bash
claude mcp add lanhu -s user \
  -e LANHU_USER_ROLE=Developer -e LANHU_USER_NAME=YourName \
  -- /bin/bash <ABS_PATH>/run-stdio.sh
```

**Claude Code（Windows）：**

```bat
claude mcp add lanhu -s user -e LANHU_USER_ROLE=Developer -e LANHU_USER_NAME=YourName -- <ABS_PATH>\run-stdio.bat
```

> Windows 提示：若 Claude Code 无法直接拉起 `.bat`，把命令改为 `-- cmd /c <ABS_PATH>\run-stdio.bat`（此写法未实测）。项目建议放在不含中文和空格的路径下；Python 需从 python.org 安装并勾选 Add to PATH。

**Claude Code（直接编辑配置文件）：** 也可以不用命令，直接编辑 `~/.claude.json`，在顶层 `mcpServers`（全局级）里加入：

```json
{
  "mcpServers": {
    "lanhu": {
      "type": "stdio",
      "command": "<ABS_PATH>/run-stdio.sh",
      "args": [],
      "env": { "LANHU_USER_ROLE": "Developer", "LANHU_USER_NAME": "YourName" }
    }
  }
}
```

- 顶层 `mcpServers` 对所有项目生效；`projects.<路径>.mcpServers` 只对该项目生效。
- 该文件可能还包含其他 MCP 服务的密钥，编辑时只改 `lanhu` 一项，分享或截图前务必脱敏。
- 配置里是本机绝对路径，不适合写进随仓库提交的 `.mcp.json`。

**Cursor 等（JSON）：** 见 [README.md](README.md) 中「按需启动配置示例」。

注册后重启会话，或在客户端里重连 MCP。用 `claude mcp list` 可看到 `lanhu` 状态。

## 注意事项

- 不要移动项目目录或删除 `venv`，否则需要重新注册路径 / 重新安装。
- Cookie 过期后，重新获取并更新项目根目录的 `.env`（见 [GET-COOKIE-TUTORIAL.md](GET-COOKIE-TUTORIAL.md)）。`.env` 已被 git 忽略，不要提交。
- 需要代理的网络环境，请让启动客户端的 shell 带上 `HTTP(S)_PROXY`。
- 更新代码后（`git pull`）如依赖有变化，执行 `./venv/bin/python -m pip install -e .`。
