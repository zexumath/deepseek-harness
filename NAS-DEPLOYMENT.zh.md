# DSH 部署笔记 — NAS + Windows 局域网访问

> 记录从零安装 DeepSeek Harness（DSH）到 Windows 浏览器经 SSH 隧道访问 NAS 上 Web UI 的全过程。
> 环境：Synology DS420plus（DSM）、Entware、Node 22.23.2、用户 `xuze`。
> 日期：2026-08-16。

## 1. 最终架构（当前运行态）

```
Windows 浏览器
  └─ http://127.0.0.1:1089
       └─ SSH LocalForward 隧道 (ssh nas)
            └─ NAS 127.0.0.1:3080
                 └─ dsh web（npm 预构建版 @deepseek-ai/dsh@0.1.0-rc.6）
                      ├─ 运行于 tmux session `dsh`
                      ├─ 命令入口：~/.local/bin/dsh（wrapper，带 --expose-internals）
                      ├─ 配置/凭据：~/.dsh/（$DSH_HOME）
                      └─ 模型：DeepSeek 官方 API（deepseek-v4-flash / deepseek-v4-pro）
```

关键路径速查：

| 项 | 路径 |
|---|---|
| dsh wrapper 命令 | `~/.local/bin/dsh`（PATH 优先级最高的 dsh） |
| npm 包 bin | `~/.local/share/pi-node/node-v22.23.2-linux-x64/lib/node_modules/@deepseek-ai/dsh/lib/bin.js` |
| DSH_HOME | `~/.dsh` |
| DeepSeek key | `~/.dsh/.credentials.yaml`（mode 600） |
| 运行 session | tmux `dsh`（`dsh web` 前台跑） |
| source repo（跟踪用，不运行） | `~/dsh`（fork `zexumath/deepseek-harness`，upstream = `deepseek-ai/deepseek-harness`） |

## 2. 环境准备

```bash
# Node 22.23.2 已存在（pi-node 目录）
node -v   # v22.23.2

# 全局装 pnpm（DSH 要求 pnpm@11.x）
npm i -g pnpm@11.7.0

# node-pty 原生模块编译需要 make（Entware）
sudo opkg install make   # 需要 root
```

## 3. 走过的弯路（source build 失败记录）

最初想按 README「Run from source」在 `~/dsh`（fork clone）里跑，遇到三个坑，最终放弃 source 运行：

1. **node-pty 编译失败**：`pnpm install` 在 `subprocess-local` 依赖的 `node-pty` 原生编译处挂掉，因为系统缺 `make`。装 make 后 `pnpm install` 通过。
2. **workspace 链接缺失**：root `node_modules/@deepseek-ai/` 只有 1 个链接（root 无依赖所致），source 模式下 loader 从 root 作用域解析不到 `@deepseek-ai/dsh-*` 包。手工软链 225 个包到 root 后能过这一关，但属 hack。
3. **HMR 需要 `--expose-internals`**（致命）：`cordis-plugin-hmr` 在 `new Hmr()` 时强制要求 Node 带 `--expose-internals`。而：
   - Node **禁止**把 `--expose-internals` 放进 `NODE_OPTIONS`（报错 "not allowed in NODE_OPTIONS"）；
   - `web-app` bundle 和用户 `cordis.patch.yml` 里的 `disabled: true` 对 hmr 条目**不生效**（rc.6 的真实 bug，`--dump-config` 显示已 disabled 但 live boot 仍加载）。

结论：**source 跑法在 NAS 上不可行，改用 npm 预构建版**。source repo 保留仅作 fork 跟踪/读码。

## 4. 最终安装（npm 预构建版）

```bash
npm i -g @deepseek-ai/dsh     # 0.1.0-rc.6
dsh --version                  # 验证
```

### PATH 覆盖 wrapper

npm 全局 bin 在 PATH 位置 6；`~/.local/bin` 在位置 2。写一个 wrapper 让 `dsh` 恒带 `--expose-internals`：

`~/.local/bin/dsh`：
```bash
#!/usr/bin/env bash
exec /volume1/homes/xuze/.local/share/pi-node/node-v22.23.2-linux-x64/bin/node \
  --expose-internals \
  /volume1/homes/xuze/.local/share/pi-node/node-v22.23.2-linux-x64/lib/node_modules/@deepseek-ai/dsh/lib/bin.js \
  "$@"
```
`chmod +x ~/.local/bin/dsh`。此 wrapper 同时覆盖 source repo（`~/dsh` 里只能 `pnpm dsh`）与 npm 原生 bin 的影响。

## 5. DeepSeek 凭据

Key 来源：pi 的 `~/.pi/agent/auth.json`（`deepseek` provider）。写入 DSH 凭据文件：

```bash
umask 077
printf 'DEEPSEEK_API_KEY: sk-...\n' > ~/.dsh/.credentials.yaml
chmod 600 ~/.dsh/.credentials.yaml
```

验证 key 有效：`GET https://api.deepseek.com/v1/models`（Bearer key）→ 返回 `deepseek-v4-flash` / `deepseek-v4-pro`。

DSH 的 `llm-deepseek` 适配器默认 `apiKeyEnv: DEEPSEEK_API_KEY`，经 `ctx.credentials` seam 解析，`.credentials.yaml` 是该 seam 的本地文件后端（mode 600 是惯例）。

## 6. 运行（tmux 常驻）

```bash
tmux new-session -d -s dsh
tmux send-keys -t dsh 'dsh web' Enter
```

- 默认监听 `127.0.0.1:3080`
- 检查：`curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3080/` → 200
- 日志：tmux pane 里（`tmux capture-pane -t dsh -p`）
- 停止：`tmux send-keys -t dsh C-c`

## 7. 局域网访问（SSH 隧道方案）

### 为什么不能直接绑局域网

DSH rc.6 **不支持非 loopback 绑定**：
- `dsh web --host 0.0.0.0` 被 CLI 明确拒绝（"intentionally not supported yet for safety"）
- `--host <具体IP>` 被 webserver 配置 schema 拒绝（只允许 `127.0.0.1` 或 `0.0.0.0`）

DSH 无任何认证层（官方说明等认证层才支持远程访问），**切勿把 3080 端口直接暴露公网**（等于暴露远程代码执行）。

### 方案：SSH LocalForward 隧道

NAS 侧（一次性，需 root）：
```bash
printf '\n# allow xuze tcp forwarding for dsh web ssh tunnel\nMatch User xuze\n    AllowTcpForwarding yes\n' >> /etc/ssh/sshd_config
/usr/bin/sshd -t           # 语法检查，空输出 = OK
kill <sshd listener PID>   # 硬重启 sshd（SIGHUP 在 Synology 定制版上不重读配置）
```
注意：Synology 默认 `AllowTcpForwarding no`（sshd_config 全局），`xuze` 不在任何 Match 块中故被拒（表现为 `channel open failed: administratively prohibited` / 浏览器 ERR_CONNECTION_RESET）。重启后 supervisor 自动拉起新 listener。

Windows 侧 `C:\Users\<用户名>\.ssh\config`：
```
Host nas
    HostName 192.168.1.184
    User xuze
    LocalForward 1089 127.0.0.1:3080
```
使用：`ssh nas` → 浏览器开 `http://127.0.0.1:1089`。隧道只在 SSH 连接存续期间有效。

### 顺带改动：关闭 SSH 登录自动进 tmux

`~/.profile` 中自动 `tmux new-session -A -s pi fish` 的 case 块已删除（保留 `tmux-pi` 别名）。`ssh nas` 现在落普通 shell，`exit` 直接断连。

## 8. 安全须知

- DSH Web UI 无认证。当前仅 loopback + SSH 隧道访问，安全。
- 未来公网访问（Phase 2 情况 B）：推荐 Tailscale/WireGuard 或带认证反代，**禁止**裸端口映射。
- `/api` browser-trust fence：所有 `/api` 请求按 Host authority 校验（loopback 或 `--trusted-host` 白名单），防 DNS rebinding；它是 reachability 策略，不是认证。
- key 文件 `~/.dsh/.credentials.yaml` 权限 600，不入 git。

## 9. 日常运维速查

```bash
# 服务检查
tmux list-sessions | grep dsh
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3080/

# 重启服务
tmux send-keys -t dsh C-c          # 停
tmux send-keys -t dsh 'dsh web' Enter  # 起

# fork 同步上游（低频）
cd ~/dsh && git fetch upstream && git merge upstream/master
```

## 10. TODO / 待办

- [ ] 公网穿透（Tailscale 优先）
- [ ] DSH 开机自启 + 崩溃拉起（Synology 任务计划或 supervisor）
- [ ] pi ↔ DSH 共享 skills：软链 `~/.pi/agent/skills/*` → `~/.agents/skills/`（DSH 默认扫该目录，格式兼容 SKILL.md）
- [ ] pi ↔ DSH 共享 memory：DSH 会话 cwd 指到 `~/butler-brain`（AGENTS.md + memory/ 同为 md 文件）
