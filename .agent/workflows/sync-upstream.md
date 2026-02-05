---
description: 同步上游 GitHub 仓库到你的 Fork
---

# 同步上游仓库工作流

此工作流用于将任何 GitHub 上游仓库的最新代码同步到你的 Fork 中。

## 前置条件
- 当前目录是一个 Git 仓库（包含 .git 文件夹）
- 该仓库是从 GitHub Fork 而来

## 步骤

### 1. 确认当前目录是 Git 仓库
// turbo
```powershell
git rev-parse --is-inside-work-tree
```
> 应该输出 `true`

### 2. 检查远程仓库配置
// turbo
```powershell
git remote -v
```
> 确认 origin 指向你的 Fork

### 3. 检查是否已配置 upstream
// turbo
```powershell
git remote | Select-String "upstream"
```
> 如果没有输出，需要执行步骤 4；如果有输出，跳到步骤 5

### 4. 添加上游仓库（如果需要）
```powershell
git remote add upstream <上游仓库URL>
```
> 将 `<上游仓库URL>` 替换为原始仓库的 Git URL
> 例如: `https://github.com/freqtrade/freqtrade.git`

### 5. 获取上游最新代码
// turbo
```powershell
git fetch upstream
```

### 6. 查看当前分支
// turbo
```powershell
git branch --show-current
```

### 7. 查看上游默认分支
// turbo
```powershell
git remote show upstream | Select-String "HEAD branch"
```
> 记住输出的分支名（如 main, master, develop）

### 8. 切换到主分支
```powershell
git checkout <主分支名>
```
> 将 `<主分支名>` 替换为步骤 7 中获取的分支名

### 9. 合并上游更改到本地
```powershell
git merge upstream/<主分支名> --no-edit
```
> 将 `<主分支名>` 替换为实际的分支名

### 10. 推送到你的 Fork
```powershell
git push origin <主分支名>
```
> 将 `<主分支名>` 替换为实际的分支名

## 完成
同步完成后，你的 Fork 将与上游仓库保持一致。

## 故障排除

### "remote upstream already exists" 错误
这是正常的，说明已经配置过 upstream，跳过步骤 4 即可。

### 合并冲突
如果步骤 9 出现冲突：
1. 运行 `git status` 查看冲突文件
2. 手动编辑解决冲突（搜索 `<<<<<<<` 标记）
3. 运行 `git add .`
4. 运行 `git commit -m "Resolve merge conflicts"`
5. 继续步骤 10

### Push 被拒绝
可能是因为远程有你的提交，使用 force push（谨慎）：
```powershell
git push origin <主分支名> --force
```

## 常用项目的 upstream URL

| 项目 | Upstream URL | 主分支 |
|------|--------------|--------|
| freqtrade | `https://github.com/freqtrade/freqtrade.git` | develop |
| React | `https://github.com/facebook/react.git` | main |
| Vue | `https://github.com/vuejs/vue.git` | main |
| Next.js | `https://github.com/vercel/next.js.git` | canary |
