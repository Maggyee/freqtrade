# 推送代码到 Fork 仓库

## 概述
此 Skill 用于将本地代码更改提交并推送到你自己的 GitHub Fork 仓库中。适用于任何 Git 项目。

## 适用场景
- 完成代码修改后需要保存到 GitHub
- 添加了新的功能或文档
- 完成了一阶段的开发工作需要备份
- 在多台设备之间同步代码

## 前提条件
1. 当前目录是一个 Git 仓库
2. 有对远程仓库的 push 权限
3. 已配置 Git 用户信息

## 操作流程

### 1. 进入项目目录
```powershell
# 进入你的项目根目录（包含 .git 文件夹的目录）
cd <你的项目路径>
```

### 2. 检查当前状态
```powershell
# 查看当前分支
git branch --show-current

# 查看文件更改状态
git status
```

预期输出示例：
```
On branch main
Changes not staged for commit:
  modified:   src/app.js

Untracked files:
  src/new-feature.js
```

### 3. 查看具体更改内容（可选）
```powershell
# 查看已修改文件的详细差异
git diff

# 查看某个具体文件的更改
git diff src/app.js

# 查看更改的统计信息
git diff --stat
```

### 4. 暂存更改
```powershell
# 暂存所有更改
git add .

# 或者只暂存特定文件
git add src/app.js
git add src/new-feature.js

# 交互式选择要暂存的更改
git add -p
```

### 5. 提交更改
```powershell
# 提交并添加描述信息
git commit -m "feat: 添加新功能"

# 或者更详细的提交（多行）
git commit -m "feat: 添加用户认证功能

- 实现登录逻辑
- 添加 JWT token 验证
- 更新 API 接口"
```

### 6. 推送到远程仓库
```powershell
# 获取当前分支名
$branch = git branch --show-current

# 推送到 origin
git push origin $branch

# 如果是新分支，需要设置上游追踪
git push -u origin $branch
```

## 提交信息规范

建议使用 [Conventional Commits](https://www.conventionalcommits.org/) 格式：

| 类型 | 说明 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat: 添加用户登录` |
| `fix` | 修复 bug | `fix: 修复数据加载错误` |
| `docs` | 文档更新 | `docs: 更新 README` |
| `style` | 代码格式 | `style: 格式化代码` |
| `refactor` | 代码重构 | `refactor: 优化数据处理` |
| `test` | 测试相关 | `test: 添加单元测试` |
| `chore` | 构建/工具 | `chore: 更新依赖` |
| `perf` | 性能优化 | `perf: 优化查询速度` |

## 完整通用脚本

```powershell
# 通用的代码推送脚本
# 用法: .\push-changes.ps1 -Message "feat: 添加新功能"

param(
    [Parameter(Mandatory=$false)]
    [string]$Message = "chore: 更新代码",
    
    [Parameter(Mandatory=$false)]
    [string]$Remote = "origin"
)

# 检查是否在 Git 仓库中
if (-not (Test-Path ".git")) {
    Write-Host "错误: 当前目录不是 Git 仓库" -ForegroundColor Red
    exit 1
}

# 获取当前分支
$branch = git branch --show-current
Write-Host "当前分支: $branch" -ForegroundColor Cyan

# 显示当前状态
Write-Host "`n=== 当前更改 ===" -ForegroundColor Yellow
git status --short

# 检查是否有更改
$changes = git status --porcelain
if (-not $changes) {
    Write-Host "没有需要提交的更改" -ForegroundColor Yellow
    exit 0
}

# 暂存所有更改
Write-Host "`n=== 暂存更改 ===" -ForegroundColor Yellow
git add .

# 提交
Write-Host "`n=== 提交: $Message ===" -ForegroundColor Yellow
git commit -m $Message

# 推送
Write-Host "`n=== 推送到 $Remote/$branch ===" -ForegroundColor Yellow
git push $Remote $branch

Write-Host "`n=== 完成 ===" -ForegroundColor Green
```

### 使用方法
```powershell
# 使用默认消息
.\push-changes.ps1

# 使用自定义消息
.\push-changes.ps1 -Message "feat: 添加新功能"

# 推送到不同的远程
.\push-changes.ps1 -Message "fix: 修复bug" -Remote "upstream"
```

## 在功能分支上开发

```powershell
# 创建并切换到新分支
git checkout -b feature/my-new-feature

# 进行开发...

# 提交更改
git add .
git commit -m "feat: 实现新功能"

# 推送新分支
git push -u origin feature/my-new-feature
```

## 常见问题

### Q: 如何配置 Git 用户信息？
```powershell
# 全局配置
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"

# 仅当前项目
git config user.name "你的用户名"
git config user.email "你的邮箱"
```

### Q: push 被拒绝怎么办？
A: 可能是远程有新的提交，先拉取再推送：
```powershell
git pull origin <branch> --rebase
git push origin <branch>
```

### Q: 如何撤销最后一次提交？
```powershell
# 撤销提交但保留更改
git reset --soft HEAD~1

# 撤销提交并丢弃更改（危险）
git reset --hard HEAD~1
```

### Q: 如何修改最后一次提交信息？
```powershell
git commit --amend -m "新的提交信息"
```

### Q: 如何忽略某些文件？
A: 编辑项目根目录的 `.gitignore` 文件：
```
# 忽略 node_modules
node_modules/

# 忽略日志
*.log

# 忽略环境配置
.env
.env.local

# 忽略构建产物
dist/
build/
```

### Q: 如何查看提交历史？
```powershell
# 查看最近 5 次提交
git log --oneline -5

# 查看详细历史
git log -5

# 查看图形化历史
git log --oneline --graph --all
```

## 相关命令速查

| 命令 | 说明 |
|------|------|
| `git status` | 查看工作区状态 |
| `git diff` | 查看未暂存的更改 |
| `git diff --staged` | 查看已暂存的更改 |
| `git add .` | 暂存所有更改 |
| `git add -p` | 交互式暂存 |
| `git commit -m "msg"` | 提交更改 |
| `git push origin <branch>` | 推送到远程 |
| `git branch --show-current` | 查看当前分支 |
| `git log --oneline -5` | 查看最近5次提交 |
| `git stash` | 临时保存更改 |
| `git stash pop` | 恢复临时保存的更改 |
| `git reset HEAD` | 取消暂存 |
| `git checkout -- <file>` | 丢弃文件更改 |
