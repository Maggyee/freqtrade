---
description: 将本地代码更改推送到你的 GitHub Fork
---

# 推送代码到 Fork 工作流

此工作流用于将本地代码更改提交并推送到任何 GitHub 仓库。

## 前置条件
- 当前目录是一个 Git 仓库（包含 .git 文件夹）
- 有待提交的代码更改
- 有对远程仓库的 push 权限

## 步骤

### 1. 确认当前目录是 Git 仓库
// turbo
```powershell
git rev-parse --is-inside-work-tree
```
> 应该输出 `true`

### 2. 检查当前分支
// turbo
```powershell
git branch --show-current
```

### 3. 查看文件更改状态
// turbo
```powershell
git status
```
> 确认有需要提交的更改

### 4. 查看更改详情（可选）
// turbo
```powershell
git diff --stat
```

### 5. 暂存所有更改
// turbo
```powershell
git add .
```
> 或者使用 `git add <file>` 只暂存特定文件

### 6. 确认暂存内容
// turbo
```powershell
git status --short
```

### 7. 提交更改
```powershell
git commit -m "<提交信息>"
```
> 请使用有意义的提交信息，推荐格式：
> - `feat: 添加新功能`
> - `fix: 修复问题`
> - `docs: 更新文档`
> - `refactor: 重构代码`
> - `chore: 杂项更新`

### 8. 获取当前分支名
// turbo
```powershell
$branch = git branch --show-current; echo "当前分支: $branch"
```

### 9. 推送到远程仓库
```powershell
git push origin <当前分支名>
```
> 将 `<当前分支名>` 替换为步骤 8 显示的分支名
> 如果是新分支，使用: `git push -u origin <分支名>`

## 完成
代码已成功推送到远程仓库！

## 快捷一行命令
如果确认要提交所有更改：
```powershell
git add . && git commit -m "<提交信息>" && git push origin $(git branch --show-current)
```

## 故障排除

### Push 被拒绝 (rejected)
远程有新的提交，先拉取再推送：
```powershell
git pull origin <分支名> --rebase
git push origin <分支名>
```

### 需要配置用户信息
```powershell
git config user.name "你的用户名"
git config user.email "你的邮箱"
```

### 撤销最后一次 add
```powershell
git reset HEAD
```

### 撤销最后一次 commit（保留更改）
```powershell
git reset --soft HEAD~1
```

### 修改最后一次提交信息
```powershell
git commit --amend -m "新的提交信息"
```

## 可选：创建 Pull Request
如果你想将更改合并到上游仓库：
1. 访问你的 GitHub Fork 页面
2. 点击 "Contribute" -> "Open pull request"
3. 选择源分支和目标分支
4. 填写 PR 标题和描述
5. 提交 PR

## 分支操作参考

### 创建新分支
```powershell
git checkout -b feature/new-feature
```

### 切换分支
```powershell
git checkout main
```

### 查看所有分支
```powershell
git branch -a
```
