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

## 11. pi ↔ DSH 共享（skills + memory）

已落地（2026-08-16）。核心机制：DSH 的 `dsh-skill-filesystem` 扫 `~/.agents/skills`（`$DSH_AGENTS_HOME`，默认 `~/.agents`），格式同为 `SKILL.md` + `name`/`description` frontmatter，与 pi 完全兼容；DSH 的 `dsh-agent-instructions` 读 `$DSH_HOME/AGENTS.md` 作为全局指令基线。

```bash
# 1) skills：symlink 到真实源头（butler-brain/skills + pi 的 grill-me）
mkdir -p ~/.agents/skills
ln -sfn ~/butler-brain/skills/* ~/.agents/skills/
ln -sfn ~/.pi/agent/skills/grill-me ~/.agents/skills/grill-me

# 2) memory 规则：DSH 全局 AGENTS.md → butler-brain 的 AGENTS.md
ln -sfn ~/butler-brain/AGENTS.md ~/.dsh/AGENTS.md
```

验证方式（headless 端到端）：
```bash
dsh --profile headless "列出你所有可用的 skills"   # 应列出全部 6 个 skill
dsh --profile headless "家里还有鸡蛋吗？"           # 应读 memory/food.md 回答
```

- skills 内容在 butler-brain/skills 单点维护，pi 和 DSH 共用（DSH watcher 默认跟随 symlink）。
- memory 数据（memory/*.md）本身是纯文件，两个 agent 用同一套 skill + 绝对路径读写，天然一致。
- 写操作（买进/用完）需 DSH 的文件写入权限，Web UI 会话里按需授权。
- 会话工作目录：建议在 Web UI 开新会话时选 `~/butler-brain`（project root 与 pi 一致，AGENTS.md 项目层规则生效）；不选也可，skills 内为绝对路径。

## 12. Sandbox 与权限模式（NAS 无沙箱后端）

本 NAS 内核 4.4（Landlock 需 5.13+）、Entware 无 bubblewrap 包 → DSH 沙箱后端不可用。默认 `workspace-write` 预设会报 `sandbox mode "workspace-write" is requested but no sandbox backend is usable`。

解法：supervisor 脚本 export `DSH_PERMISSION_MODE=danger-full-access`（sandbox-policy 的 config 已支持该环境变量：`mode: !!js process.env.DSH_PERMISSION_MODE ?? 'workspace-write'`）。命令无沙箱直跑，家庭单用户可接受；日后若能装 bubblewrap，删掉这行即恢复。

## 13. 开机自启（supervisor）

`~/.local/bin/dsh-web-supervisor.sh`：循环跑 `dsh web`，退出后 5 秒拉起，日志 `/tmp/dsh-web.log`。
开机注册：`/usr/local/etc/rc.d/S99dsh-web.sh`（DSM 开机自动执行 rc.d 目录），以 root 写：

```sh
#!/bin/sh
case "$1" in
  start)
    /bin/su - xuze -c '/var/services/homes/xuze/.local/bin/dsh-web-supervisor.sh >/dev/null 2>&1 &'
    ;;
  stop)
    pkill -f 'bin.js web'
    pkill -f 'dsh-web-supervisor'
    ;;
esac
```

运行管理：
```bash
# 查看状态
ps aux | grep -E 'supervisor|bin.js web' | grep -v grep
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3080/

# 日志
tail -f /tmp/dsh-web.log

# 重启（停服务由 supervisor 自动拉起）
pkill -f 'bin.js web'
```

（原 tmux `dsh` session 方式已废弃，改为 supervisor 常驻。）

已验证：`/usr/local/etc/rc.d/S99dsh-web.sh stop/start` 两分支均正常（2026-08-16 实测），DSM 开机时将自动以同样方式拉起。

## 14. 公网穿透尝试记录（2026-08-16，已回退）

**方案 2 尝试：Tailscale Funnel + Caddy basic auth（为鸿蒙 NEXT 手机提供公网 + 认证入口）**

- 下载 Caddy v2.11.4 静态二进制至 `~/.local/bin/caddy`；Caddyfile 在 `~/.local/etc/dsh/Caddyfile`（basic auth + 反代 127.0.0.1:3080，监听 127.0.0.1:8080）；密码哈希用 `caddy hash-password` 生成。
- `tailscale serve --bg 8080` + `tailscale funnel --bg 8080` 配置成功，域名 `https://ds420plus.taildfa60f.ts.net/`。
- **失败现象**：所有设备（鸿蒙公网、iPhone/电脑 tailnet 内）访问该域名均为空白页，无认证弹框；Caddy access log 无任何请求记录（请求未到达 Caddy）；重启 Tailscale 套件无效。
- **结论**：DSM 上 tailscaled 为 userspace-networking 模式，serve/funnel 从 3080 改指 8080 后转发失败（回退到 3080 即恢复，说明 tailscaled→127.0.0.1:3080 通、→8080 不通的原因未定位，待 Tailscale 更新后再试）。
- **回退**：`tailscale funnel --https=443 off` + `tailscale serve --bg 3080`；Caddy 已停（supervisor 脚本中 Caddy 启动段注释保留）。
- **当前状态**：ts.net 域名仅 tailnet 内可用（直达 3080，无认证）；鸿蒙 NEXT（无 Tailscale 客户端）仍无访问途径。

恢复认证层的步骤（以后再试）：取消 supervisor 里 Caddy 段注释 → 启动 Caddy → `tailscale serve --bg 8080` → 验证 401 弹框后再开 funnel。

## 15. TODO / 待办

- [ ] 鸿蒙 NEXT 手机访问 DSH（公网 + 认证）：Funnel+Caddy 未通，待 Tailscale 修复或改试 Cloudflare Tunnel+Access / frp+VPS
- [x] 公网穿透（Tailscale 基础组网，Windows/iPhone 已可用）
- [x] DSH 开机自启 + 崩溃拉起（supervisor + /usr/local/etc/rc.d/S99dsh-web.sh）
- [x] pi ↔ DSH 共享 skills（symlink `~/butler-brain/skills/*` → `~/.agents/skills/`）
- [x] pi ↔ DSH 共享 memory（`~/.dsh/AGENTS.md` → butler-brain/AGENTS.md + skills 绝对路径）
- [x] 沙箱问题：切 `DSH_PERMISSION_MODE=danger-full-access`（NAS 无 bwrap/Landlock）
