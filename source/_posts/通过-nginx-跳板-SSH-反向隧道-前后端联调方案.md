---
title: 通过 nginx 跳板 + SSH 反向隧道实现前后端联调方案
date: 2026-10-23 15:00:00
tags:
---

适用场景：前端、后端开发者处于不同网段，彼此网络不通，但都能访问同一台 nginx 服务器（且有其 SSH 权限），开发机均为 Windows。

## 1. 背景与目标

- 前端开发者与后端开发者处于不同网段，无法直接互访。
- 双方都可以访问并 SSH 登录同一台 nginx 服务器。
- nginx 服务器**不能主动访问**两台开发机（开发机在 NAT/防火墙之后）。
- 目标：以 nginx 为跳板，实现前后端联调（前端调后端接口，后端也能查看前端页面）。

## 2. 方案原理

nginx 只能做反向代理，要求它能主动访问被代理方。由于开发机对 nginx 不可达，因此改由**开发机主动向 nginx 建立 SSH 反向隧道（ssh -R）**，把本机端口"挂"到 nginx 服务器的回环地址上，再由 nginx 代理这些端口。

```
前端开发机 ──ssh -R──▶ nginx 服务器 127.0.0.1:15173（前端页面）
后端开发机 ──ssh -R──▶ nginx 服务器 127.0.0.1:18080（后端接口）

访问方 ──▶ nginx:8000 ┬─ /      → 127.0.0.1:15173
                       └─ /api/  → 127.0.0.1:18080
```

两台开发机都是主动向外连接，因此不需要在开发机上开放任何入站端口。

## 3. 前置条件与端口规划

### 3.1 前置条件

- 两位开发者都有 nginx 服务器的 SSH 账号。
- nginx 服务器 sshd 允许端口转发（`AllowTcpForwarding yes`，默认即是）。
- 开发机为 Win10/11，已自带 OpenSSH 客户端（`ssh -V` 可验证）。
- nginx 服务器防火墙/安全组放行对外访问端口（本文为 8000），建议仅对开发者网段放行。

### 3.2 端口规划（示例）

| 角色 | 本机服务端口 | nginx 上的隧道端口 | nginx 对外路径 |
| --- | --- | --- | --- |
| 前端开发者 | 5173（Vite） | 15173 | `/` |
| 后端开发者 | 8080 | 18080 | `/api/` |

多人并存时，各自使用不同隧道端口，并在 nginx 用不同路径前缀区分（如 `/api-zhang/`、`/api-li/`）。

## 4. 配置步骤

### 4.1 配置 SSH 密钥免密登录（两位开发者各做一次）

在 PowerShell 中执行：

```powershell
# 生成密钥（一路回车）
ssh-keygen -t ed25519

# 将公钥写入 nginx 服务器（此步需输入一次密码）
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh user@nginx服务器 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

之后即可免密建立隧道，脚本中无需出现密码。

### 4.2 前端开发者：启动前端并暴露

1. 启动前端服务：`npm run dev`，本地访问 `http://127.0.0.1:5173` 确认正常。
2. Vite 需放开 Host 校验并（如有热更新问题）指定 HMR 端口：

```js
// vite.config.js
export default defineConfig({
  server: {
    host: '127.0.0.1',
    allowedHosts: true,          // 测试阶段放开，或写具体域名/IP
    hmr: { clientPort: 8000 }    // 通过 nginx 8000 端口访问时保证热更新
  }
})
```

Vue CLI / webpack devServer 对应配置为 `allowedHosts: 'all'`（旧版 `disableHostCheck: true`）。

3. 建立隧道：

```powershell
ssh -N -R 127.0.0.1:15173:127.0.0.1:5173 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes user@nginx服务器
```

窗口保持打开、无报错即成功。

### 4.3 后端开发者：暴露后端接口

```powershell
ssh -N -R 127.0.0.1:18080:127.0.0.1:8080 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes user@nginx服务器
```

### 4.4 在 nginx 服务器上验证隧道

```bash
curl -I http://127.0.0.1:15173    # 前端隧道
curl -I http://127.0.0.1:18080    # 后端隧道
```

返回 HTTP 响应说明隧道已通。不通则不要继续配置 nginx，先回头检查本机服务与隧道命令。

### 4.5 nginx 配置

建议使用独立配置文件，例如 `/etc/nginx/conf.d/dev-test.conf`：

```nginx
server {
    listen 8000;
    server_name _;

    # 前端页面（含热更新 WebSocket）
    location / {
        proxy_pass http://127.0.0.1:15173;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 后端接口
    location /api/ {
        proxy_pass http://127.0.0.1:18080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

使配置生效：

```bash
nginx -t && nginx -s reload
```

`proxy_pass http://127.0.0.1:18080/;` 末尾带斜杠会去掉 `/api/` 前缀，不带则保留，需与后端路由约定一致。

### 4.6 联调访问方式

- **后端查看前端页面**：浏览器访问 `http://nginx服务器地址:8000`。
- **前端本地调后端接口**（前端不暴露页面也可）：在 dev server 中配置代理，代码里仍写 `/api/...`，同时避免跨域：

```js
// vite.config.js
server: {
  proxy: {
    '/api': { target: 'http://nginx服务器地址:8000', changeOrigin: true }
  }
}
```

## 5. 最小测试流程（建议先做）

先仅测试"前端页面能否通过 nginx 被后端访问"：

1. 前端开发者启动前端服务，并配置 `allowedHosts`。
2. 前端开发者建立 15173 隧道。
3. 在 nginx 服务器上 `curl -I http://127.0.0.1:15173` 验证返回 200。
4. 配置 nginx 并 reload，放行 8000 端口。
5. 后端开发者浏览器访问 `http://nginx服务器地址:8000`，能看到页面即打通；修改前端代码，验证热更新是否同步。
6. 测试通过后，再按 4.3 建立后端隧道并启用 `/api/` 的 location。

## 6. 命令参数说明

以 `ssh -N -R 127.0.0.1:15173:127.0.0.1:5173 -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes user@nginx服务器` 为例：

| 参数 | 含义 |
| --- | --- |
| `-N` | 不执行远程命令、不打开 shell，仅做端口转发。窗口看起来"卡住"属正常。 |
| `-R 127.0.0.1:15173:127.0.0.1:5173` | 反向端口转发：在 nginx 服务器监听 `127.0.0.1:15173`，流量经隧道转到本机 `127.0.0.1:5173`。前半段绑定回环，仅 nginx 本机可访问，更安全；后半段写 `127.0.0.1` 而非 `localhost`，避免 Windows 解析为 IPv6 `::1`。 |
| `-o ServerAliveInterval=30` | 每 30 秒发送心跳，防止空闲断连，并能及时发现断线。 |
| `-o ServerAliveCountMax=3` | 连续 3 次心跳无响应则判定断线并退出（默认 3）。 |
| `-o ExitOnForwardFailure=yes` | 转发建立失败（如端口被占用）时直接退出，避免"登录成功但隧道实际未通"。 |
| `-o BatchMode=yes` | 不弹出密码提示，认证失败直接退出，适合脚本。 |
| `-p 端口` | SSH 端口非 22 时指定。 |
| `-i 私钥路径` | 指定私钥文件。 |
| `-v` | 输出详细日志，用于排错。 |

## 7. 稳定性：断线自动重连

Windows 没有 autossh，可使用循环脚本。

### 7.1 批处理脚本 tunnel.bat（双击运行）

```bat
@echo off
chcp 65001 >nul
set REMOTE=user@nginx服务器
set REMOTE_PORT=15173
set LOCAL_PORT=5173

:loop
echo [%time%] 正在建立隧道: nginx:%REMOTE_PORT% ^> 本机:%LOCAL_PORT%
ssh -N -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes -o BatchMode=yes -R 127.0.0.1:%REMOTE_PORT%:127.0.0.1:%LOCAL_PORT% %REMOTE%
echo [%time%] 隧道断开，5 秒后重连（Ctrl+C 退出）
timeout /t 5 >nul
goto loop
```

### 7.2 使用 ssh config 简化命令

编辑 `C:\Users\用户名\.ssh\config`：

```
Host devtunnel
    HostName nginx服务器地址
    User user
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ExitOnForwardFailure yes
    RemoteForward 127.0.0.1:18080 127.0.0.1:8080
```

之后只需执行 `ssh -N devtunnel`。

### 7.3 图形化工具

MobaXterm（Tunneling → Remote port forwarding）、Bitvise SSH Client 均支持保存配置与自动重连，可作为免开发的替代方案。

## 8. 关于密码

OpenSSH 没有命令行传密码的参数（`-p` 是端口），Windows 也没有 `sshpass`。处理方式：

1. **推荐**：使用密钥登录（见 4.1），脚本中不出现密码。
2. 必须使用密码时，改用 PuTTY 套件的 `plink.exe`，通过 `-pw` 传入密码；首次需手动连接一次以信任服务器指纹，否则 `-batch` 会失败。
3. 明文密码风险：脚本被分享或提交到 Git 即泄露；命令行参数可能被本机其他进程看到。建议让管理员为联调单独创建低权限账号，并避免将脚本纳入版本库。

## 9. 常见问题排查

| 现象 | 原因与处理 |
| --- | --- |
| 浏览器打不开/超时 | nginx 服务器防火墙或安全组未放行 8000 端口 |
| 502 Bad Gateway | 隧道断开或本机服务未启动，在 nginx 服务器上 `curl http://127.0.0.1:隧道端口` 验证 |
| Blocked request / Invalid Host header | 前端 dev server 未配置 `allowedHosts` |
| 页面能开但样式/JS 加载失败 | 前端 `base` 配置了子路径，或 nginx location 路径与之不匹配 |
| 热更新无效、控制台 WebSocket 报错 | 缺少 `Upgrade`/`Connection` 头，或 Vite 未设置 `hmr.clientPort` |
| `remote port forwarding failed` | 隧道端口被占用或旧连接未释放，`ss -lntp \| grep 端口` 查看，换端口或清理旧连接 |
| `Permission denied` | 密钥或账号问题，先用 `ssh user@nginx服务器` 确认可登录 |
| 隧道通但转发到本机失败 | `localhost` 解析为 IPv6；命令中统一使用 `127.0.0.1`；或服务只监听了特定 IP，需将目标改为该 IP |
| `bad permissions`（私钥） | 在私钥文件属性 → 安全中仅保留自己账户的权限 |
| 电脑休眠后断线 | 唤醒后循环脚本会自动重连；联调期间建议关闭睡眠 |

补充：

- **WebSocket**：接口或热更新使用 WebSocket 时，必须设置 `proxy_http_version 1.1` 及 `Upgrade`/`Connection` 头。
- **超时与大文件**：必要时调大 `proxy_read_timeout`、`client_max_body_size`。
- **Cookie/登录态**：统一通过 nginx 同一地址访问可避免跨域；必要时使用 `proxy_cookie_domain`。

## 10. 安全建议

- 隧道端口只绑定 `127.0.0.1`，不对外暴露。
- nginx 的 8000 端口限制来源 IP，或增加 Basic Auth：

```nginx
allow 10.1.0.0/16;
deny all;
# 或
auth_basic "dev";
auth_basic_user_file /etc/nginx/.htpasswd;
```

- 优先使用密钥认证，必要时在服务器 `authorized_keys` 中为联调密钥加 `restrict,port-forwarding,permitlisten="127.0.0.1:15173"` 限制其只能做端口转发。
- 联调结束后关闭隧道（Ctrl+C），并删除 nginx 测试配置后 reload。

## 11. 工具化与替代方案

### 11.1 封装为 Windows 程序（可选）

| 方案 | 形态 | 开发量 | 适用 |
| --- | --- | --- | --- |
| 批处理/PowerShell 脚本 | 双击运行 | 极低 | 先跑通，自用 |
| 脚本 + 托盘图标 | 常驻托盘，右键启停 | 低 | 日常联调 |
| 图形界面程序 | 填端口、点按钮、看状态 | 中 | 给不熟命令行的同事 |
| 打包 exe（PyInstaller / C#） | 单文件或安装包 | 中 | 分发给多人 |

核心逻辑是启动并监控一个 `ssh.exe` 子进程，并在退出时自动重连。建议路线：先用脚本跑通，稳定后再按需升级为图形界面并打包。

### 11.2 其他替代方案

- **frp**：在 nginx 服务器部署 frps，两位开发者运行 frpc，配置文件化、支持 token 认证与自动重连，适合长期使用。
- **Tailscale / ZeroTier / WireGuard**：组建虚拟局域网，两人直接互通，如允许安装通常更省事。
- **公司已有 VPN/内网穿透**：优先使用现成方案。

## 12. 实施清单

- [ ] 双方确认可 SSH 登录 nginx 服务器
- [ ] 双方配置密钥免密登录
- [ ] 约定并登记各自的隧道端口，避免冲突
- [ ] 前端配置 `allowedHosts` 与 `hmr.clientPort`
- [ ] 建立隧道，并在 nginx 服务器上 curl 验证
- [ ] 配置 nginx，`nginx -t` 后 reload
- [ ] 放行 nginx 的 8000 端口（限制来源网段）
- [ ] 后端访问 `http://nginx服务器地址:8000` 验证页面与热更新
- [ ] 后端建立 `/api/` 隧道，前端通过代理联调接口
- [ ] 联调结束后关闭隧道并清理 nginx 配置
