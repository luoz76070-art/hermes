# 博客搭建说明

这个项目是一套 `Hugo + PaperMod` 的个人博客，现在采用的推荐部署方式是：

- 本地或 CI 构建静态文件到 `public/`
- 用 `rsync` 把构建结果上传到服务器的 `releases/<release_id>/`
- 由服务器上的 `current` 符号链接切换线上版本
- `Nginx` 只负责服务 `current/` 下的静态文件

这样保留了 Hugo 静态站的简单和稳定，同时把部署从“手工上传目录”升级成了更稳的“原子发布 + 可回滚”。

## 目录说明

- `hugo.yaml`: Hugo 主配置
- `content/posts/`: 博客文章
- `themes/PaperMod/`: PaperMod 主题
- `bin/hugo`: 本地开发可选的 Hugo 二进制，占位保留但默认不提交到 git
- `assets/css/extended/custom.css`: 自定义样式
- `scripts/build.sh`: 生产构建脚本
- `scripts/preview.sh`: 本地预览脚本
- `scripts/deploy_to_aliyun.sh`: 原子发布脚本
- `scripts/rollback_release.sh`: 回滚脚本
- `deploy/nginx/blog.conf.example`: Nginx 配置示例
- `deploy/github-actions/deploy.yml.example`: GitHub Actions 自动发布工作流模板

## 先改这几个地方

在 `hugo.yaml` 里把下面内容改成你自己的：

- `baseURL`
- `title`
- `author`
- `socialIcons`

Nginx 示例文件里也要把域名和证书路径改成你自己的：

- `server_name`
- `ssl_certificate`
- `ssl_certificate_key`

当前项目里已经按你的线上信息预填好了：

- 域名：`zyzlz.xin`
- 服务器 IP：`8.153.100.129`

## 本地预览

```bash
./scripts/preview.sh
```

打开 <http://localhost:1313> 即可预览。

## 生产构建

```bash
./scripts/build.sh
```

这个脚本默认会带上：

- `--environment production`
- `--cacheDir ./.tmp/hugo_cache`
- `--gc`
- `--minify`
- `--cleanDestinationDir`

构建结果会出现在 `public/`。

## 服务器目录结构

推荐把服务器目录整理成下面这样：

```text
/var/www/blog/
├── current -> /var/www/blog/releases/20260417093015
└── releases/
    ├── 20260417090001/
    ├── 20260417093015/
    └── ...
```

Nginx 的 `root` 应指向 `/var/www/blog/current`。

## 首次部署前准备

1. 确保服务器已安装 `nginx`
2. 确保本机已安装 `ssh` 和 `rsync`
3. 把 [deploy/nginx/blog.conf.example](deploy/nginx/blog.conf.example) 改成你的域名后放到服务器上
4. 让站点目录对部署用户可写，例如 `/var/www/blog`
5. 执行 `nginx -t && systemctl reload nginx`

## 手动发布

```bash
./scripts/deploy_to_aliyun.sh 8.153.100.129 /var/www/blog root
```

这个脚本会自动：

- 先执行本地构建
- 在服务器创建新的 `releases/<release_id>/`
- 用 `rsync` 上传 `public/`
- 把 `current` 原子切换到新版本
- 自动清理旧版本，默认保留最近 `5` 个

常用参数：

```bash
./scripts/deploy_to_aliyun.sh 8.153.100.129 /var/www/blog root --keep 7 --port 22
./scripts/deploy_to_aliyun.sh 8.153.100.129 /var/www/blog root --skip-build
```

## 回滚

回滚到上一个版本：

```bash
./scripts/rollback_release.sh 8.153.100.129 /var/www/blog root
```

回滚到指定版本：

```bash
./scripts/rollback_release.sh 8.153.100.129 /var/www/blog root --release 20260417093015
```

## 自动发布

仓库里已经放了一份 GitHub Actions 工作流模板：

- [deploy/github-actions/deploy.yml.example](deploy/github-actions/deploy.yml.example)

它会在 `main` 分支有新提交时自动：

- 安装 Hugo extended
- 构建站点
- 用仓库里的部署脚本发布到服务器

你需要在 GitHub 仓库里配置：

- `Secret`: `BLOG_DEPLOY_SSH_KEY`
- `Variable`: `BLOG_DEPLOY_HOST`
- `Variable`: `BLOG_DEPLOY_PATH`
- `Variable`: `BLOG_DEPLOY_USER`
- `Variable`: `BLOG_DEPLOY_PORT`
- `Variable`: `BLOG_RELEASES_TO_KEEP`

启用时，把它复制到仓库根目录的 `.github/workflows/deploy.yml` 即可生效。

如果你用的是 HTTPS 凭据推送 GitHub，而当前凭据没有 `workflow` scope，保留成模板形式会更省心；等你后面换成有对应权限的 token 或 SSH 推送，再启用它就行。

如果你已经把 `boke/` 初始化成独立仓库，推荐仓库根目录就直接放在当前目录：

```bash
cd /Users/rorance/workspace/boke
git init -b main
```

## 补充说明

- `public/` 仍然应该继续忽略，不建议提交生成产物
- 仓库里的 `bin/hugo` 是本地开发用的，但它超过 GitHub 单文件限制，默认不会提交；CI 会在 Linux runner 上单独安装 Hugo
- 相比原来的 `scp -r public/* ...`，现在这套流程不会残留旧文件，也更适合回滚
