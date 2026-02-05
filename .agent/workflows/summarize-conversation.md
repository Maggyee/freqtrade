---
description: 总结当前对话为学习笔记 Markdown 文件
---

# 总结对话工作流

此工作流用于将当前的 AI 对话总结为一份结构化的 Markdown 学习笔记。

## 前置条件
- 已完成一个技术任务或学习主题的对话
- 确定笔记的保存位置

## 步骤

### 1. 确定笔记主题和标题
请用户确认：
- 笔记的主要主题是什么？
- 希望使用什么标题？

### 2. 确定保存位置
询问用户希望保存到哪里：
- 当前项目目录（如 `./docs/notes/`）
- 用户文档目录（如 `~/Documents/LearningNotes/`）
- 其他自定义位置

### 3. 回顾对话关键内容
// turbo
分析当前对话，提取以下内容：
- 解决的问题或完成的任务
- 使用的技术和工具
- 执行的关键命令和代码
- 遇到的问题和解决方案
- 学到的重要概念

### 4. 生成笔记文档
创建 Markdown 文件，包含以下结构：

```markdown
# [主题标题]

**日期**: [当前日期]
**标签**: [相关标签]

## 📋 概述
[本次对话的简要描述]

## 🎯 目标
[列出本次任务的目标]

## 🔧 操作步骤
[详细的操作步骤，包含代码示例]

## 💻 关键代码
[整理后的关键代码和命令]

## 📚 学习要点
[核心概念和最佳实践]

## ✅ 总结
[主要收获和结论]
```

### 5. 保存文件
使用命名规范: `YYYY-MM-DD_主题关键词.md`

### 6. 确认完成
告知用户：
- 文件保存位置
- 文件名称
- 简要内容概述

## 输出示例

文件名: `2026-02-05_git-sync-fork.md`

```markdown
# Git Fork 同步操作指南

**日期**: 2026-02-05
**标签**: #Git #GitHub #Fork #版本控制

## 📋 概述
本文档记录了如何将 GitHub 上游仓库的最新代码同步到自己的 Fork 中。

## 🎯 目标
- 配置上游仓库 (upstream)
- 获取上游最新代码
- 合并到本地并推送到 Fork

## 🔧 操作步骤

### 步骤 1: 添加上游仓库
\`\`\`powershell
git remote add upstream https://github.com/原作者/项目.git
git remote -v  # 验证配置
\`\`\`

### 步骤 2: 获取上游更新
\`\`\`powershell
git fetch upstream
\`\`\`

### 步骤 3: 合并更新
\`\`\`powershell
git checkout main
git merge upstream/main --no-edit
\`\`\`

### 步骤 4: 推送到 Fork
\`\`\`powershell
git push origin main
\`\`\`

## 💻 关键代码

### 一键同步脚本
\`\`\`powershell
git fetch upstream
git checkout main
git merge upstream/main --no-edit
git push origin main
\`\`\`

## 📚 学习要点
- `origin` 是你的 Fork 仓库
- `upstream` 是原始仓库
- 使用 `--no-edit` 避免打开编辑器

## ✅ 总结
通过配置 upstream 远程仓库，可以方便地保持 Fork 与上游同步。
```

## 自定义选项

用户可以指定：
- **详细程度**: 简洁版 / 标准版 / 详细版
- **代码风格**: 只保留关键代码 / 包含所有代码
- **额外章节**: 是否包含"相关资源"、"后续计划"等

## 故障排除

### 对话内容太长
- 只提取最重要的部分
- 分多个文件记录不同主题

### 不确定重点
- 询问用户最想记录哪些内容
- 按时间顺序整理关键节点
