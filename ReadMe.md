#### 换源
npm config set registry https://registry.npmmirror.com/
npm config get registry

#### 清缓存
npm cache clean --force

#### 安装
npm install

#### 启动
npm run serve

`npm run server` 与 `npm run serve` 相同。生成静态文件用 `npm run build`（即 `hexo generate`）。

#### 部署
推送到 `main` 后由 GitHub Actions 构建，并用 rsync 同步到服务器 `/var/www/blog`。不要在服务器上构建，也不要使用 `hexo deploy`。密钥和目录说明见 [DEPLOY.md](DEPLOY.md)。

#### 文档
https://hexo.io/zh-cn/docs/writing

#### 安装hexo
npm install -g hexo-cli

#### 创建文档
hexo new post hello
