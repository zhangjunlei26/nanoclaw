---
name: setup
description: 运行初始的NanoClaw设置。当用户想安装依赖、认证消息通道、注册主通道或启动后台服务时使用。触发器包括“设置”、“安装”、“配置NanoClaw”或首次设置请求。
---

# NanoClaw装置

自动运行设置步骤。只有在需要用户操作（通道认证、配置选择）时才会暂停。设置时用 `bash setup.sh`做bootstrap，然后 `npx tsx setup/index.ts --step <name>` 用 bash 做所有步骤。步骤会发出结构化状态块以完成。冗长的日志会变成`log/setup.log`。

**原理：** 当东西坏了或缺失时，就要修复它。除非真的需要用户手动操作（比如认证频道、粘贴秘密令牌），否则不要让用户自己去修。如果缺少依赖，就安装它。如果服务无法启动，就诊断并修理。需要时先征求用户许可，然后开始工作。

**用户体验说明：** 所有面向用户的问题都可以使用 `AskUserQuestion`。

## 0. Git & Fork 设置

检查 git 远程配置，确保用户有 fork 并且upstream配置已完成。

运行：

- `git remote -v`

**情况 A - `origin` 指向 `qwibitai/nanoclaw`（用户直接克隆）：**

用户是克隆而不是分叉。AskUserQuestion：“你直接克隆了NanoClaw。我们建议分叉，这样你可以推动自定义内容。你想fork一个分支吗？”

- 现在就fork（推荐）— 按步骤运行
- 继续使用分支 — 它们只会有局部变化

如果 fork：指示用户在 GitHub 上 fork `qwibitai/nanoclaw`（需要在浏览器中完成），然后要求他们提供 GitHub 用户名。运行：

```
git remote rename origin upstream
git remote add origin https://github.com/<their-username>/nanoclaw.git
git push --force origin main
```

用 `git remote -v` 验证。

如果继续不分叉：添加upstream代码，这样他们仍能拉取更新：

```
git remote add upstream https://github.com/qwibitai/nanoclaw.git
```

**情况 B — `origin`指向用户的分支，没有`upstream`远程：**

添加upstream：

```
git remote add upstream https://github.com/qwibitai/nanoclaw.git
```

**情况 C — 既存在`起源`分支（用户分支）也存在`upstream`分支（qwibitai）：**

已经配置好了。继续。

**Verify：** `git remote -v` 应该显示 `origin` → 用户仓库， `upstream` → `qwibitai/nanoclaw.git` 。

## 1. 引导（Node.js + 依赖）

运行`bash setup.sh` 并解析状态块。

- 如果 NODE_OK=假，→ Node.js 缺失或过于旧。确认后使用`AskUserQuestion: Would you like me to install Node.js 22?`
  - macOS：`brew install node@22`（如果有 brew），或者安装nvm，然后 `nvm install 25`
  - Linux： `curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt-get install -y nodejs` ， 或 nvm
  - 安装 Node 后，重新运行 `bash setup.sh`
- 如果 DEPS_OK=false → 读取`log/setup.log`。试试：删除 `node_modules`，重跑 `bash setup.sh`。如果原生模块构建失败，安装构建工具（macOS上是`xcode-select --install` ，Linux 上是 `build-essential`），然后重新尝试。
- 如果 NATIVE_OK=false → better-sqlite3 未能加载。安装构建工具并重新运行。
- 记录`PLATFORM`和`IS_WSL`以备后续步骤使用。

## 2. 检查环境

运行 `npx tsx setup/index.ts --step environment` 并解析状态块。

- 如果 HAS_AUTH=true→WhatsApp 已经配置好，请注意第 5 步
- 如果 HAS_REGISTERED_GROUPS=true，→已存在配置，可以跳过或重新配置
- 记录第 3 步的 Docker 和 APPLE_CONTAINER 值

## 3. 容器运行时

### 3a. 选择运行时间

查看 `APPLE_CONTAINER` 和 `DOCKER` 的预检结果，以及第一步的 PLATFORM。

- PLATFORM=linux → Docker（唯一选项）
- PLATFORM=macos + APPLE_CONTAINER=已安装 → 使用 `AskUserQuestion: Docker (cross-platform) or Apple Container (native macOS?` If Apple Container，现在运行 `/convert-to-apple-container`，然后跳到 3c。
- PLATFORM=macos + APPLE_CONTAINER=not_found → Docker

### 3a-docker. 安装 Docker

- DOCKER=运行→继续到 4b
- DOCKER=installed_not_running → start Docker： `open -a Docker` （macOS） 或 `sudo systemctl start docker` （Linux）.等 15 秒，重新核对 `docker 信息 `。
- DOCKER=not_found → 如果确认使用`AskUserQuestion: Docker is required for running agents. Would you like me to install it?`
  - macOS：通过 brew 安装——`cask docker`，然后`打开-a Docker`，等待它启动。如果无法使用 brew，直接下载到 Docker Desktop https://docker.com/products/docker-desktop
  - Linux：用 `curl -fsSL https://get.docker.com | sh && sudo usermod -aG docker $USER` .注意：用户可能需要登出/登录才能获得组员资格。

### 3b. 苹果容器转换门（如有需要）

**如果选择的运行时是 Apple Container**，你必须检查源代码是否已经从 Docker 转换为 Apple 容器。千万不要跳过这一步。运行：

```
grep -q "CONTAINER_RUNTIME_BIN = 'container'" src/container-runtime.ts && echo "ALREADY_CONVERTED" || echo "NEEDS_CONVERSION"
```

**如果 NEEDS_CONVERSION**，源代码仍然使用 Docker 作为运行时。你必须现在就运行 `/convert-to-apple-container` 技能，才能进入构建步骤。

**如果 ALREADY_CONVERTED**，代码已经用了苹果容器。继续看 3c。

**如果选择运行时是 Docker**，则无需转换。继续看 3c。

### 3c. 构建与测试

运行 `npx tsx setup/index.ts --step container -- --runtime <chosen>` 并解析状态块。

**如果 BUILD_OK=假：** 读取`日志/setup.log` 尾部以查找构建错误。

- 缓存问题（过时层）：`docker builder prune -f`（Docker）或 `container builder stop && container builder rm && container builder start` （Apple Container）。重试。
- Dockerfile 语法或缺失文件：从日志中诊断并修复，然后再试。

**如果 TEST_OK=假但 BUILD_OK=真：** 映像能建好但运行不了。检查日志——常见原因是运行时没有完全启动。等一下，再试一次。

## 4. Claude 认证（无脚本）

如果第二步的 HAS_ENV=真，读取 `.env` 并检查 `CLAUDE_CODE_OAUTH_TOKEN` 或 `ANTHROPIC_API_KEY`。如果有，请和用户确认：保留还是重新配置？

AskUserQuestion：Claude 订阅（Pro/Max）与 Anthropic API 密钥的区别？

**订阅：** 告诉用户在另一个终端运行 `claude setup-token`，复制令牌，添加到 `CLAUDE_CODE_OAUTH_TOKEN=<token>` `.env`。千万不要在聊天中领取代币。

**API 密钥：** 告诉用户将 `ANTHROPIC_API_KEY=<key>` 添加到 `.env`。

## 5. 建立频道

AskUserQuestion（多选）：你想启用哪些消息频道？

- WhatsApp（通过二维码或配对码进行身份验证）
- Telegram（通过机器人令牌从@BotFather 认证）
- Slack（通过带套接字模式的 Slack 应用认证）
- Discord（通过 Discord 机器人令牌认证）

**将每个频道的专长委托给每个频道。** 每个通道技能负责其自身的代码安装、认证、注册和 JID 解析。这避免了通道特定逻辑的重复，并确保 JID 始终正确。

对于每个选定的通道，调用其技能：

- **WhatsApp：**Invoke `/add-whatsapp`
- **电报：** 调用 `/add-telegram`
- **Slack：** 调用 `/add-slack`
- **Discord：** 调用 `/add-discord`

每个技能都会：

1. 安装通道代码（通过 `git 合并`技能分支）
2. 收集凭证/令牌并写入 `.env`
3. 认证（WhatsApp 二维码/配对，或验证基于令牌的连接）
4. 请用正确的 JID 格式注册聊天
5. 构建与验证

**完成所有通道技能后** ，安装依赖并重建——通道合并可能会引入新的包：

```
npm install && npm run build
```

如果构建失败，读取错误输出并修复（通常是缺少依赖）。然后继续进行第6步。

## 6. 山准许名单

AskUserQuestion：代理访问外部目录？

**不：** `npx tsx setup/index.ts --step mounts -- --empty` **是的：** 收集路径/权限。 `npx tsx setup/index.ts --step mounts -- --json '{"allowedRoots":[...],"blockedPatterns":[],"nonMainReadOnly":true}'`

## 7. 开始服务

如果服务已经运行：先卸货。

- macOS： `launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist`
- Linux： `systemctl --user stop nanoclaw` （或者 `systemctl 如果 root 就停止 nanoclaw`）

运行 `npx tsx setup/index.ts --step service` 并解析状态块。

**如果 FALLBACK=wsl_no_systemd：** 检测到无 systemd 的 WSL。告诉用户他们可以在 WSL 中启用 systemd（ `echo -e "[boot]\nsystemd=true" | sudo tee /etc/wsl.conf` 然后重启 WSL），或者使用生成的 `start-nanoclaw.sh` 包装器。

**如果 DOCKER_GROUP_STALE=真：** 用户在会话开始后被添加到了 docker 组——systemd 服务无法访问 Docker socket。请用户执行以下两个命令：

1. 立即解决办法： `sudo setfacl -m u:$(whoami):rw /var/run/docker.sock`
2. 持久修复（每次重启 Docker 后重新应用）：

```
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/socket-acl.conf << 'EOF'
[Service]
ExecStartPost=/usr/bin/setfacl -m u:USERNAME:rw /var/run/docker.sock
EOF
sudo systemctl daemon-reload
```

把`用户名`替换成真正的用户名（来自 `whoami`）。分别运行两个 `sudo` 命令——先做 `tee` heredoc，然后 `daemon-reload`。用户确认 setfacl 已运行后，重新运行服务步骤。

**如果 SERVICE_LOADED=假：**

- 请阅读`日志/setup.log` 以查找错误。
- macOS：检查 `launchctl list | grep nanoclaw` 。如果 PID=`-` 且状态非零，则读取`日志/nanoclaw.error.log`。
- Linux：已完成 `systemctl --user status nanoclaw` 。
- 修复后重新运行服务步骤。

## 8. 验证

运行 `npx tsx setup/index.ts --step verify` 并解析状态块。

**如果 STATUS=失败，则分别修正：**

- SERVICE=停止→`npm 运行构建 `，然后重启： `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` （macOS）或 `systemctl --user restart nanoclaw` （Linux）或 `bash start-nanoclaw.sh`（WSL nohup）
- SERVICE=not_found → 重跑第 7 步
- CREDENTIALS=缺少→重跑步骤 4
- CHANNEL_AUTH 显示任何频道的 `not_found`，→重新调用该频道的技能（例如 `/add-telegram`）
- REGISTERED_GROUPS=0 →重新调用第 5 步的频道技能
- MOUNT_ALLOWLIST=缺失→ `npx tsx setup/index.ts --step mounts -- --empty`

告诉用户测试：在注册聊天中发送消息。显示：` 尾部 -f logs/nanoclaw.log`

## 故障排除

**服务无法启动：** 查看`日志/nanoclaw.error.log`。常见情况：错误的节点路径（重跑第 7 步）、缺少 `.env`（第 4 步）、缺少信道凭证（重新调用信道技能）。

**容器代理失败（“Claude Code 进程以代码 1 退出”）：** 确保容器运行时正在运行——` 打开 -a Docker`（macOS Docker）、`容器系统启动`（Apple Container），或 `sudo systemctl start docker`（Linux）。请查看容器日志。 `groups/main/logs/container-*.log`

**消息无回复：** 检查触发模式。主频道不需要前缀。查阅 DB： `npx tsx setup/index.ts --step verify` 。查看`日志/nanoclaw.log`。

**频道无法连接：** 确认频道凭证是否设置在 `.env` 中。当信道的凭证存在时，它们会自动启用。关于 WhatsApp：检查商店 `/认证/creds.json`。对于基于令牌的信道：检查 `.env` 中的令牌值。任何更改 `.env` 后重启服务。

**卸载服务：**macOS： `launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist` |Linux： `systemctl --user stop nanoclaw`

## 诊断

请阅读并关注 [diagnostics.md](https://github.com/zhangjunlei26/nanoclaw/blob/main/.claude/skills/setup/diagnostics.md)。
