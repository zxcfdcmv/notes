---
tags:
    - 工具
---

> [!tip]
> - 网页走 Cloudflare 托管
> - 图片走 Cloudflare 静态图片站托管
> - 全站实现无死角的国内 CDN 秒开加速
> - 完全不消耗 Cloudflare Pages 每月 500 次的构建额度上限

---
# 准备工作（只需配置一次）
## 图片自动上传代码仓
==obsidian== 中安装 ==notepix== 插件，配好==代码仓分支路径token==等参数
笔记中==粘贴图片==会自动上传代码仓，并且会直接将图片修改为 ==github== 的 ==raw 链接==

## 配置笔记项目
> 为了让 GitHub 有权限把文件上传到 Cloudflare 账户，需要去两边后台拿一下密钥并配置到 GitHub 仓库里

1. 获取 Cloudflare 账户 ID (Account ID)

- 登录 Cloudflare 控制台，点击右侧的 **"Workers & Pages" (Workers 和 Pages)**。
- 在右侧边栏，你会看到一串由 32 位字母和数字组成的 **Account ID**，把它复制下来。

2. 创建 Cloudflare 创建 API 令牌 (API Token)

- 点击 Cloudflare 右上角的用户头像 -> **My Profile (我的个人资料)** -> **API Tokens (API 令牌)**。
- 点击 **Create Token (创建令牌)** -> 选择最下方的 **Create Custom Token (创建自定义令牌)**。
- **令牌名称**：比如叫 `GitHub-Actions-Pages`。

- **权限配置 (Permissions)**：
    - 选择：`Account` (账户) -> `Cloudflare Pages` -> `Edit` (编辑)
- 点击下一步并生成，**复制生成的这一长串 API 令牌**（它只会出现一次，注意保存）。

3. 将密钥填入 GitHub 仓库

- 打开你存放笔记的 GitHub 仓库，进入 **Settings** -> **Secrets and variables** -> **Actions**。
- 点击 **New repository secret**，分别添加以下两个变量：
    - 名字填 `CLOUDFLARE_ACCOUNT_ID`，内容填你的账户 ID。
    - 名字填 `CLOUDFLARE_API_TOKEN`，内容填你的 API 令牌。

4. 在 Cloudflare 创建一个空白项目

- 回到 Cloudflare 的 **Workers & Pages** 页面。
- 点击 **Create** -> **Pages** -> 选择 **Upload assets (上传资源)**。
- 给项目起一个名字（比如 `my-quartz-blog`），然后点击创建即可（不需要手动上传任何文件，点击完成后直接退出）。
- 将这个名字填入上面 YAML 脚本第 9 步的 `projectName: "你的CF项目名称"` 处。


## 配置图片加速项目

1. **创建并选择仓库**：
    - 登录 Cloudflare，进入 **Workers & Pages** -> 点击 **Create** -> 选择 **Pages** -> 点击 **Connect to Git**。
    - 选中 **`notes`** 仓库。
2. **构建设置（微调这里）**：
    - **Production branch (生产分支 / 默认分支)**：**必须从 `main` 改成 `images`**。这样 Cloudflare 才会去读取存放图片的分支。
    
    - **Framework preset (框架预设)**：选择 **None**。
    - **Build command (构建命令)**：**留空**（什么都不用填）。
    - **Build output directory (输出目录)**：**填入 `assets`**（注意前面有个斜杠）
3. **点击保存并部署 即可部署成功**

---

# github actions
> [!tip] 通过 ==Github Actions== 配置自动推送 ==Cloudflare Pages==

`.github/workflows/deploy_CF.yml`：
```yml
name: Deploy Quartz site to Cloudflare Pages

on:
  push:
    branches:
      - main

permissions:
  contents: read
  deployments: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # 1. 拉取纯笔记仓库（保持所有 Git 历史以确保日期准确）
      - name: Checkout Notes
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          path: notes

      # 2. 拉取 Quartz v5 框架
      - name: Checkout Quartz v5 Framework
        uses: actions/checkout@v4
        with:
          repository: jackyzha0/quartz
          ref: v5
          path: quartz

      # 3. 组装内容与配置文件
      - name: Prepare Content and Config
        run: |
          # 清空默认内容
          rm -rf quartz/content/*
          
          # 复制笔记，过滤掉不需要编译的仓库文件
          rsync -av \
            --exclude='.git' \
            --exclude='.github' \
            --exclude='README.md' \
            --exclude='.gitignore' \
            --exclude='quartz.config.yaml' \
            --exclude='ppconfig.json' \
            notes/ quartz/content/
          
          # 覆盖配置文件
          if [ -f notes/quartz.config.yaml ]; then
            cp notes/quartz.config.yaml quartz/
          else
            echo "Error: quartz.config.yaml not found!"
            exit 1
          fi

      # 4. 设置 Node 环境
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: 'npm'
          cache-dependency-path: quartz/package-lock.json

      # 5. 安装核心依赖
      - name: Install dependencies
        run: |
          cd quartz
          npm ci --legacy-peer-deps

      # 6. 提前显式安装默认主题包，防止 build 过程中动态加载死锁
      - name: Pre-install Default Theme
        run: |
          cd quartz
          npm install @quartz-themes/default --no-save

      # 7. 根据 YAML 配置安装其他社区插件
      - name: Install Quartz plugins
        run: |
          cd quartz
          npx quartz plugin install --from-config

      # 7.5 自动将所有笔记中的 GitHub 原始图床链接，替换为专用 Cloudflare Pages 图片站域名
      - name: Optimize Image Links for Cloudflare CDN
        run: |
          find quartz/content -type f -name "*.md" -exec sed -i 's|https://raw.githubusercontent.com/zxcfdcmv/notes/images/assets/|https://notes-beh.pages.dev/|g' {} +

      # 8. 执行编译
      - name: Build Quartz
        run: |
          cd quartz
          npx quartz build

      # 9. 将编译好的静态文件上传到 Cloudflare Pages
      - name: Publish to Cloudflare Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: "zxcfdcmv-notes" # 填入在 Cloudflare Pages 创建的项目名
          directory: "quartz/public"  # Quartz 编译出来的静态文件目录
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}

```

- 注意修改其中的 ==代码仓库链接== 与 ==cloudflare pages域名==

