# 同步上游 GitHub 仓库到 Fork

## 概述
此 Skill 用于将 GitHub 上游仓库的最新更改同步到你自己的 Fork 仓库中。适用于任何 Fork 的开源项目。

## 适用场景
- 当你 Fork 了一个开源项目
- 需要定期获取上游仓库的最新更新
- 想要将上游更新同步到你的 Fork 中

## 前提条件
1. 已经 Fork 了上游仓库到你的 GitHub 账户
2. 已将 Fork 克隆到本地
3. 有对 Fork 仓库的 push 权限

## 操作流程

### 1. 进入项目目录
```powershell
# 进入你的项目根目录（包含 .git 文件夹的目录）
cd <你的项目路径>
```

### 2. 检查当前远程仓库配置
```powershell
# 查看现有的远程仓库
git remote -v
```

预期输出应该包含 `origin`（你的 Fork）:
```
origin  https://github.com/<你的用户名>/<项目名> (fetch)
origin  https://github.com/<你的用户名>/<项目名> (push)
```

### 3. 添加上游仓库（如果尚未添加）
```powershell
# 添加官方仓库作为 upstream
# 将 <原作者> 和 <项目名> 替换为实际值
git remote add upstream https://github.com/<原作者>/<项目名>.git

# 验证添加成功
git remote -v
```

添加后应该看到：
```
origin    https://github.com/<你的用户名>/<项目名> (fetch)
origin    https://github.com/<你的用户名>/<项目名> (push)
upstream  https://github.com/<原作者>/<项目名>.git (fetch)
upstream  https://github.com/<原作者>/<项目名>.git (push)
```

### 4. 获取上游仓库的最新更新
```powershell
# 拉取上游所有分支和标签
git fetch upstream
```

### 5. 确定主分支名称
```powershell
# 查看上游仓库的默认分支
git remote show upstream | Select-String "HEAD branch"

# 或者查看所有远程分支
git branch -r
```

常见的主分支名：`main`, `master`, `develop`

### 6. 切换到主分支
```powershell
# 切换到主分支（根据项目使用 main/master/develop）
git checkout <主分支名>
```

### 7. 合并上游更改
```powershell
# 将上游的主分支合并到本地
git merge upstream/<主分支名> --no-edit
```

如果有冲突：
```powershell
# 查看冲突文件
git status

# 解决冲突后
git add .
git commit -m "Merge upstream/<主分支名>"
```

### 8. 推送到你的 Fork
```powershell
# 将同步后的代码推送到 GitHub
git push origin <主分支名>
```

## 完整通用脚本

```powershell
# 通用的同步上游仓库脚本
# 用法: .\sync-upstream.ps1 -UpstreamUrl "https://github.com/owner/repo.git" -Branch "main"

param(
    [Parameter(Mandatory=$false)]
    [string]$UpstreamUrl,
    
    [Parameter(Mandatory=$false)]
    [string]$Branch = "main"
)

# 检查是否在 Git 仓库中
if (-not (Test-Path ".git")) {
    Write-Host "错误: 当前目录不是 Git 仓库" -ForegroundColor Red
    exit 1
}

# 检查是否已添加 upstream
$remotes = git remote
if ($remotes -notcontains "upstream") {
    if (-not $UpstreamUrl) {
        Write-Host "错误: 未配置 upstream，请提供 -UpstreamUrl 参数" -ForegroundColor Red
        exit 1
    }
    git remote add upstream $UpstreamUrl
    Write-Host "已添加 upstream: $UpstreamUrl" -ForegroundColor Green
}

# 获取上游更新
Write-Host "正在获取上游更新..." -ForegroundColor Cyan
git fetch upstream

# 切换到目标分支
git checkout $Branch

# 合并上游更改
Write-Host "正在合并 upstream/$Branch..." -ForegroundColor Cyan
git merge upstream/$Branch --no-edit

# 推送到 Fork
Write-Host "正在推送到 origin/$Branch..." -ForegroundColor Cyan
git push origin $Branch

Write-Host "同步完成！" -ForegroundColor Green
```

## 常见问题

### Q: 如何找到上游仓库的 URL？
A: 访问你 Fork 的 GitHub 页面，点击 "forked from xxx/xxx" 链接

### Q: 合并时出现冲突怎么办？
A: 手动解决冲突文件，然后 `git add .` 和 `git commit`

### Q: 不同项目使用不同的主分支怎么办？
A: 
- 大多数新项目使用 `main`
- 老项目可能使用 `master`
- 某些项目使用 `develop`（如 freqtrade）
- 使用 `git remote show upstream` 查看

### Q: upstream 已存在怎么办？
A: 可以用 `git remote set-url upstream <new-url>` 更新地址

## 相关命令速查

| 命令 | 说明 |
|------|------|
| `git remote -v` | 查看远程仓库列表 |
| `git remote add <name> <url>` | 添加远程仓库 |
| `git remote remove <name>` | 删除远程仓库 |
| `git remote set-url <name> <url>` | 更新远程仓库地址 |
| `git fetch <remote>` | 获取远程仓库更新 |
| `git merge <remote>/<branch>` | 合并远程分支 |
| `git push <remote> <branch>` | 推送到远程仓库 |
| `git remote show <remote>` | 查看远程仓库详情 |
