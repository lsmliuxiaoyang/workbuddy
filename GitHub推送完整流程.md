# GitHub 推送完整流程指南

> 适用场景：本地有公司 git 账号，想把文件推送到个人 GitHub 仓库
> 作者：lsmliuxiaoyang | 更新日期：2026-06-22

---

## 前置准备

| 需要准备的东西 | 说明 |
|---|---|
| GitHub 个人账号 | 注册地址：[github.com](https://github.com) |
| GitHub 仓库 | 提前在 GitHub 网站上建好 |
| Personal Access Token | 用于命令行认证，替代密码 |

---

## 第一步：在 GitHub 创建仓库

1. 打开 [github.com/new](https://github.com/new)
2. 填写仓库名，例如：`workbuddy`
3. 选择 **Public**（公开，GitHub Pages 免费托管需要）
4. 点击 **Create repository**（其他选项保持默认）

---

## 第二步：生成 Personal Access Token

> Token 是命令行推送代码时的"密码"，比账号密码更安全。

1. 打开 [github.com/settings/tokens/new](https://github.com/settings/tokens/new)
2. **Note**：随便填，例如 `push`
3. **Expiration**：选 `90 days`（或根据需要选择）
4. **Select scopes**：勾选第一个大选项 `repo`（会自动勾上所有子项）
5. 拉到页面底部，点击绿色按钮 **Generate token**
6. ⚠️ **立刻复制生成的 Token**（格式：`ghp_xxxxxxxxxxxx`），离开页面后不再显示！

---

## 第三步：本地初始化 git 仓库

```bash
# 1. 创建并进入项目目录
mkdir ~/my-project && cd ~/my-project

# 2. 把要上传的文件放入目录（示例：复制 HTML 文件并重命名为 index.html）
cp /path/to/你的文件.html index.html

# 3. 初始化 git
git init

# 4. 设置本仓库的个人账号（只影响当前仓库，不影响全局公司配置）
git config user.name "你的GitHub用户名"
git config user.email "你的GitHub邮箱"
```

---

## 第四步：提交文件

```bash
# 1. 将文件添加到暂存区
git add .

# 2. 提交（引号内填写本次提交的说明）
git commit -m "feat: 首次提交"

# 3. 将本地分支命名为 main
git branch -M main
```

---

## 第五步：关联远程仓库并推送

> 把 Token 内嵌到 URL 里，避免弹出账号密码输入框。

```bash
# 1. 关联远程仓库（替换 YOUR_TOKEN、YOUR_USERNAME、YOUR_REPO）
git remote add origin https://YOUR_TOKEN@github.com/YOUR_USERNAME/YOUR_REPO.git

# 2. 推送到远程
git push -u origin main
```

**实际示例：**
```bash
git remote add origin https://ghp_xxxxxxxxxxxx@github.com/lsmliuxiaoyang/workbuddy.git
git push -u origin main
```

### 如果提示"远程仓库有内容，需要先拉取"

```bash
# 先拉取远程内容，再推送
git pull --rebase origin main
git push origin main
```

---

## 第六步：开启 GitHub Pages（生成可访问的网页链接）

> 如果只是上传代码存档，这步可以跳过。如果需要把 HTML 文件变成可访问的网页，需要开启此功能。

### 方式一：网页操作（推荐新手）

1. 打开你的仓库页面：`github.com/YOUR_USERNAME/YOUR_REPO`
2. 点击顶部 **Settings**
3. 左侧菜单找到 **Pages**
4. **Source** 选择 `Deploy from a branch`
5. **Branch** 选择 `main`，路径选 `/`（root）
6. 点击 **Save**
7. 等待约 1-2 分钟，页面顶部会出现访问链接

### 方式二：命令行（一行搞定）

```bash
curl -X POST \
  -H "Authorization: token YOUR_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/YOUR_USERNAME/YOUR_REPO/pages \
  -d '{"source":{"branch":"main","path":"/"}}'
```

开启成功后，访问链接格式为：
```
https://YOUR_USERNAME.github.io/YOUR_REPO/
```

---

## 以后更新文件

每次修改文件后，执行以下命令即可更新：

```bash
cd ~/my-project

# 修改文件后执行：
git add .
git commit -m "更新内容说明"
git push
```

推送后约 1 分钟，GitHub Pages 自动更新。

---

## 多账号场景：公司账号 + 个人账号并存

如果本机已有公司 git 账号，不要改全局配置，只在项目目录内单独设置：

```bash
# 查看全局账号（公司账号）
git config --global user.name
git config --global user.email

# 只给当前仓库设置个人账号（不加 --global）
git config user.name "个人GitHub用户名"
git config user.email "个人GitHub邮箱"

# 验证当前仓库使用的账号
git config user.name
git config user.email
```

这样公司其他仓库完全不受影响。

---

## 常见问题排查

| 错误信息 | 原因 | 解决方法 |
|---|---|---|
| `rejected (fetch first)` | 远程仓库有本地没有的内容 | 先执行 `git pull --rebase origin main` |
| `Authentication failed` | Token 过期或权限不足 | 重新生成 Token，确保勾选了 `repo` |
| `could not lock config file` | git config 文件有 macOS 扩展属性锁 | 执行 `xattr -d com.apple.provenance .git/config` |
| `remote: Repository not found` | 仓库名或用户名拼写错误 | 检查 remote url 是否正确 |
| `Permission denied (publickey)` | 用了 SSH 方式但没配置密钥 | 改用 HTTPS 方式（Token 内嵌到 URL）|

---

## 完整命令速查（复制即用）

```bash
# ====== 首次推送 ======
mkdir ~/my-project && cd ~/my-project
cp /path/to/文件 index.html
git init
git config user.name "YOUR_USERNAME"
git config user.email "YOUR_EMAIL"
git add .
git commit -m "feat: 首次提交"
git branch -M main
git remote add origin https://YOUR_TOKEN@github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main

# ====== 后续更新 ======
cd ~/my-project
git add .
git commit -m "更新内容"
git push

# ====== 开启 GitHub Pages ======
curl -X POST \
  -H "Authorization: token YOUR_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/YOUR_USERNAME/YOUR_REPO/pages \
  -d '{"source":{"branch":"main","path":"/"}}'
```

---

*文档版本 v1.0 · lsmliuxiaoyang · 2026-06-22*
