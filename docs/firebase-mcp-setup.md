# Firebase MCP Server 配置与使用指南

> 对话时间: 2026-01-30  
> 相关项目: Freqtrade 加密货币交易机器人

---

## 目录

- [Node.js 版本问题](#nodejs-版本问题)
- [Firebase MCP Server 简介](#firebase-mcp-server-简介)
- [Freqtrade 是否需要 Firebase](#freqtrade-是否需要-firebase)
- [安装与配置过程](#安装与配置过程)
- [Firebase 在 Freqtrade 中的应用场景](#firebase-在-freqtrade-中的应用场景)

---

## Node.js 版本问题

### 错误信息

```
npm warn EBADENGINE Unsupported engine {
  package: 'superstatic@10.0.0',
  required: { node: '20 || 22 || 24' },
  current: { node: 'v25.2.1', npm: '11.6.2' }
}
context deadline exceeded
```

### 原因分析

1. **EBADENGINE 警告**: Node.js v25.2.1 太新，`superstatic@10.0.0` 只支持 Node.js 20/22/24
2. **context deadline exceeded**: npm 下载依赖时网络请求超时

### 解决方案

| 方案 | 命令 | 推荐程度 |
|------|------|----------|
| 降级 Node.js (推荐) | `nvm install 22 && nvm use 22` | ⭐⭐⭐⭐⭐ |
| 使用国内镜像 | `npm config set registry https://registry.npmmirror.com` | ⭐⭐⭐⭐ |
| 忽略引擎检查 (临时) | `npm install --engine-strict=false` | ⭐⭐ |

---

## Firebase MCP Server 简介

Firebase MCP Server 是一个 **Model Context Protocol (MCP) 服务器**，让 AI 编程助手能够直接与 Firebase 服务交互。

### 主要功能

| 功能 | 描述 |
|------|------|
| 🔥 项目管理 | 列出、管理 Firebase 项目 |
| 📦 Firestore | 读写数据库文档 |
| 🔐 Authentication | 管理用户认证 |
| ☁️ Cloud Functions | 部署、查看函数日志 |
| 🗄️ Storage | 文件上传下载 |

### 使用场景示例

- "帮我查看 users 集合中最近 10 条数据"
- "根据我的 Firestore 结构生成 TypeScript 类型定义"
- "检查为什么用户登录失败"
- "部署我的 Cloud Functions"

---

## Freqtrade 是否需要 Firebase

**答案: 不需要** ❌

Freqtrade 是完全独立的系统，使用的技术栈：

| 组件 | 技术 |
|------|------|
| 数据存储 | SQLite（默认）或 PostgreSQL |
| 配置 | JSON 文件 (`config.json`) |
| 交易执行 | 交易所 API (Binance, OKX 等) |
| 数据分析 | Pandas, TA-Lib |
| Web UI | FreqUI (可选) |

---

## 安装与配置过程

### 步骤 1: 安装 Firebase CLI

```powershell
npm install -g firebase-tools
```

### 步骤 2: 登录 Firebase

```powershell
firebase login
```

系统会要求：
1. 是否启用 Gemini in Firebase 功能 → 输入 `Y`
2. 允许收集使用数据 → 输入 `Y`
3. 在浏览器中完成 Google 账号授权

### 步骤 3: 验证登录状态

```powershell
firebase projects:list
```

### 配置完成后的状态

| 检查项 | 状态 |
|--------|------|
| 用户认证 | ✅ 已登录 |
| Gemini 服务条款 | ✅ 已接受 |
| Firebase 项目 | ✅ freqtrade |
| MCP Server | ✅ 可正常使用 |

---

## Firebase 在 Freqtrade 中的应用场景

虽然 Freqtrade 本身不需要 Firebase，但可以用来**扩展功能**：

### 1. Firestore - 情绪数据存储与缓存

适用于 crypto-sentiment-skill 的缓存策略：

```json
{
    "coin": "BTC",
    "timestamp": "2026-01-30T22:00:00Z",
    "score": -35,
    "label": "bearish",
    "confidence": 78,
    "sources": ["reddit", "twitter", "cryptopanic"]
}
```

**好处**：
- 避免重复调用 LLM API（节省成本）
- 跨设备/跨实例共享情绪数据
- 构建历史情绪数据库

### 2. Cloud Messaging - 交易信号推送

当检测到极端情绪时，推送通知：

```
🚨 BTC 情绪警报
情绪分数: -75 (极度恐惧)
FOMO: 1/10 | FUD: 9/10
```

### 3. Firebase Hosting - 实时监控仪表板

部署 Web 仪表板显示：
- 各币种情绪分数趋势图
- FOMO/FUD 指数
- 鲸鱼动向监控
- 交易机器人状态

### 4. Firebase Auth - 多用户访问控制

- 用户登录管理
- 策略配置权限控制
- API Key 安全存储

### 架构示意图

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Social Media  │────▶│  情绪分析服务    │────▶│    Firestore    │
│ Reddit/Twitter  │     │  (LLM + Python)  │     │  (数据存储)     │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │
                        ┌─────────────────────────────────┼─────────────────────────────────┐
                        ▼                                 ▼                                 ▼
               ┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
               │   Freqtrade     │              │   Web Dashboard │              │   Mobile App    │
               │  (交易执行)     │              │  (Firebase Host)│              │  (推送通知)     │
               └─────────────────┘              └─────────────────┘              └─────────────────┘
```

### 推荐程度

| 场景 | 推荐 | 说明 |
|------|------|------|
| 个人单机使用 | ⭐⭐ | SQLite 本地缓存足够 |
| 多设备同步 | ⭐⭐⭐⭐ | Firestore 很方便 |
| 构建监控仪表板 | ⭐⭐⭐⭐⭐ | Firebase Hosting 免费额度充足 |
| 团队协作 | ⭐⭐⭐⭐⭐ | Auth + Firestore 完美搭配 |
| 手机推送通知 | ⭐⭐⭐⭐⭐ | FCM 是最佳选择 |

---

## 相关资源

- [Firebase 官方文档](https://firebase.google.com/docs)
- [Firebase CLI 参考](https://firebase.google.com/docs/cli)
- [Freqtrade 文档](https://www.freqtrade.io/en/stable/)
- [crypto-sentiment-skill](../user_data/strategies/crypto-sentiment-skill/SKILL.md)
