# GitHub Actions 部署指南（中文版）

本文档介绍如何使用 GitHub Actions 将 Vue 3 + Vite 项目自动部署到 GitHub Pages。

---

## 第一步：创建源码仓库

1. 登录 [GitHub](https://github.com)，点击右上角 **+** → **新建仓库**。
2. 填写仓库信息：
   - **仓库名称**：项目名，比如 `my-project`（任意命名）。
   - **说明**：可选，项目描述。
   - **公开 / 私有**：建议选 **公开**（GitHub Pages 免费部署要求公开仓库）。
3. 点击 **创建仓库**。
4. 将本地代码推送到远程仓库：

```bash
git init
git add .
git commit -m "初始化项目"
git branch -M main
git remote add origin https://github.com/<你的用户名>/my-project.git
git push -u origin main
```

---

## 第二步：创建 `<用户名>.github.io` 仓库

这是 GitHub Pages 的**部署目标仓库**（也叫用户站点仓库）。

1. 点击右上角 **+** → **新建仓库**。
2. **关键配置**：
   - **仓库名称**：必须填 `<你的用户名>.github.io`
     - 比如用户名是 `zhangsan`，就填 `zhangsan.github.io`
     - 用户名必须和你的 GitHub 账号名**完全一致**
   - **可见性**：必须选 **公开**
3. 点击 **创建仓库**（不要勾选"添加 README 文件"）。

> **说明**：源码仓库放项目源代码，`<用户名>.github.io` 仓库接收构建后的静态文件。访问 `https://<用户名>.github.io` 就能看到部署好的页面。

---

## 第三步：创建个人访问令牌

为了让 GitHub Actions 能把构建产物推送到 `<用户名>.github.io` 仓库，需要一个有写入权限的令牌。

1. 点击右上角头像 → **设置**。
2. 左侧菜单 → **开发者设置** → **个人访问令牌** → **令牌（经典版）**。
3. 点击 **生成新令牌（经典版）**。
4. 配置如下：

| 字段 | 说明 |
|------|------|
| **备注** | 填个说明，比如 `deploy-to-pages` |
| **过期时间** | 建议选 `永不过期` 或按需设置 |
| **选择权限** | 勾选 **workflow** 和 **repo** 全部权限 |

5. 点击 **生成令牌** → **立即复制保存**（离开页面后将无法再查看）。

---

## 第四步：在源码仓库中添加密钥

1. 进入源码仓库（`my-project`）→ **设置** → **密钥和变量** → **操作**。
2. 点击 **新建仓库密钥**，添加以下内容：

| 名称 | 值 |
|------|-----|
| `DEPLOY_TOKEN` | 上一步生成的个人访问令牌 |

---

## 第五步：编写工作流文件

在源码仓库根目录创建 `.github/workflows/deploy.yml`：

```yaml
name: 部署到 GitHub Pages

on:
  push:
    branches: [main]

# 设置令牌权限
permissions:
  contents: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. 检出源码
      - name: 检出源码
        uses: actions/checkout@v4

      # 2. 安装 Node.js
      - name: 安装 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      # 3. 安装依赖
      - name: 安装依赖
        run: npm ci

      # 4. 构建项目（Vite 默认输出到 dist 目录）
      - name: 构建项目
        run: npm run build

      # 5. 部署到 <用户名>.github.io 仓库
      - name: 部署到 GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          # 使用之前创建的密钥
          personal_token: ${{ secrets.DEPLOY_TOKEN }}
          # 目标仓库
          external_repository: <你的用户名>/<你的用户名>.github.io
          # 发布分支
          publish_branch: main
          # 构建产物目录
          publish_dir: ./dist
          # 提交信息
          commit_message: ${{ github.event.head_commit.message }}
          # 允许空提交
          allow_empty_commit: true
```

> **注意**：把 `<你的用户名>` 替换成你实际的 GitHub 用户名。

---

## 第六步：验证部署

1. 把工作流文件推送到源码仓库的 `main` 分支：

```bash
git add .github/workflows/deploy.yml
git commit -m "添加 GitHub Actions 部署配置"
git push origin main
```

2. 进入源码仓库 → **操作** 标签页，查看工作流运行状态。
3. 运行成功后，访问 `https://<你的用户名>.github.io` 就能看到部署好的页面。

---

## 常见问题

### 部署失败，提示权限不足？

- 确认 `DEPLOY_TOKEN` 密钥已添加到**源码仓库**（不是目标仓库）。
- 确认令牌勾选了 `repo` 和 `workflow` 权限。

### 页面空白或资源 404？

- 检查 `vite.config.ts` 中的 `base` 配置：

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  base: '/', // 用户站点用根路径
})
```

### 如何部署到项目站点（非用户站点）？

- 如果不需要 `<用户名>.github.io`，也可以部署到当前仓库的 `gh-pages` 分支，修改 `deploy.yml`：

```yaml
- name: 部署到 GitHub Pages
  uses: peaceiris/actions-gh-pages@v4
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./dist
```

然后在仓库 **设置 → 页面** 中选择 `gh-pages` 分支作为部署源。

---

## 工作流程图

```
┌──────────────────┐     git push      ┌──────────────────┐
│   本地开发环境    │ ────────────────→ │   源码仓库        │
│  (my-project)    │                   │  (my-project)     │
└──────────────────┘                   └────────┬─────────┘
                                                │ 触发工作流
                                                ▼
                                        ┌───────────────┐
                                        │  GitHub Actions │
                                        │  1. npm ci     │
                                        │  2. npm run    │
                                        │     build      │
                                        │  3. 推送 dist   │
                                        └───────┬───────┘
                                                │
                                                ▼
                                        ┌──────────────────┐
                                        │  部署目标仓库      │
                                        │  (用户名.github.io) │
                                        └────────┬─────────┘
                                                 │
                                                 ▼
                                        ┌──────────────────┐
                                        │  GitHub Pages     │
                                        │  用户名.github.io  │
                                        └──────────────────┘
```
