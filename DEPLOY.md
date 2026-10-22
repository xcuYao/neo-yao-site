# 部署到 https://neoyao.site

静态文件由 GitHub Actions 在推送到 `main` 时构建，再通过 SSH rsync 到腾讯云 nginx 的站点目录。服务器上的 nginx 与 Let's Encrypt 证书保持现状，本仓库不修改服务器配置，也不在服务器上执行 Hexo。

- 站点：`https://neoyao.site`（`_config.yml` 的 `url`）
- 远程目录：`/var/www/blog/`
- 主机：`124.222.200.82`
- 工作流：`.github/workflows/deploy.yml`
- 触发：push 到 `main`，或手动 `workflow_dispatch`

`rsync` 的源和目标都以 `/` 结尾，因此同步的是 `public/` 里面的文件，而不是再套一层 `public` 目录。`--delete` 会删掉远程目录里本次构建没有的旧页面。

## 本地

```bash
npm ci
npm run build    # hexo generate，输出到 public/（已被 gitignore）
npm run serve    # 本地预览，等同 npm run server
```

`public/` 和 `node_modules/` 不要提交。

## GitHub Actions 密钥

在仓库 Settings → Secrets and variables → Actions 中添加：

| Secret | 必填 | 含义 |
| --- | --- | --- |
| `SSH_PRIVATE_KEY` | 是 | 部署用私钥全文（OpenSSH 格式，无 passphrase） |
| `SSH_HOST` | 是 | `124.222.200.82` 或 `neoyao.site` |
| `SSH_USER` | 是 | 能写入 `/var/www/blog` 的 SSH 用户 |
| `SSH_PORT` | 否 | SSH 端口，不设则为 `22` |
| `SSH_KNOWN_HOSTS` | 否 | `ssh-keyscan` 得到的主机公钥。不设时工作流使用 `StrictHostKeyChecking=accept-new` |

远程路径写死为 `/var/www/blog/`，不是密钥。

在本机生成一对仅用于部署的密钥，把公钥放进服务器对应用户的 `~/.ssh/authorized_keys`，把私钥原文粘贴到 `SSH_PRIVATE_KEY`（包含 `BEGIN` / `END` 行）。建议使用 ed25519。

可选，把主机公钥钉死，避免每次连接信任当次扫描结果：

```bash
ssh-keyscan -p 22 124.222.200.82
```

把输出整段存为 `SSH_KNOWN_HOSTS`。

## 服务器目录权限

nginx 的 `root` 应为 `/var/www/blog`（已有配置，不要为这次部署去改 server 块）。部署用户需要能在该目录里增删文件，nginx 运行用户需要能读取。例如部署用户是 `deploy`：

```bash
sudo mkdir -p /var/www/blog
sudo chown -R deploy:www-data /var/www/blog
sudo chmod -R u+rwX,g+rX,o+rX /var/www/blog
```

工作流上传时使用 `--chmod=D755,F644`，保证新文件对 nginx 可读。`--delete` 会清掉该目录中不属于本次 `public/` 的文件，不要把证书、nginx 配置或其他站点放在 `/var/www/blog` 里。

## CI 里的 Node

工作流使用 Node.js 20。Hexo 6.3 要求 Node `>=12.13.0`，主题 Stellar 1.27 文档写明在 Node 20 这一代 LTS 上使用。不要改成在服务器上 `npm install` 或 `hexo generate`。
