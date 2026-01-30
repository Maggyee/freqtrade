# 使用 LLM 分析社交媒体情绪 - 完整指南

> 本文档详细介绍如何使用大语言模型 (LLM) 分析社交媒体情绪，并将其集成到 Freqtrade 量化交易策略中。
>
> **参考**: 本文档策略代码基于 [Freqtrade 官方文档](https://www.freqtrade.io/en/develop/strategy-101) 最新 API (`INTERFACE_VERSION = 3`) 编写。

---

## 目录

1. [概述](#概述)
2. [为什么使用 LLM](#为什么使用-llm)
3. [系统架构](#系统架构)
4. [LLM 提供商对比](#llm-提供商对比)
5. [核心代码实现](#核心代码实现)
   - [LLM 情绪分析器](#llm-情绪分析器)
   - [社交媒体数据采集器](#社交媒体数据采集器)
   - [Freqtrade 策略集成](#freqtrade-策略集成)
6. [配置指南](#配置指南)
7. [Prompt Engineering 技巧](#prompt-engineering-技巧)
8. [成本估算](#成本估算)
9. [注意事项与最佳实践](#注意事项与最佳实践)
10. [常见问题](#常见问题)

---

## 概述

市场情绪是影响加密货币价格的重要因素之一。通过分析社交媒体上的讨论，我们可以捕捉市场参与者的情绪状态，从而辅助交易决策。

### 情绪交易的核心逻辑

```
极度恐惧 + 技术面超卖 = 买入机会 (逆向操作)
极度贪婪 + 技术面超买 = 卖出机会 (逆向操作)
```

这与巴菲特的名言一致：**"在别人恐惧时贪婪，在别人贪婪时恐惧"**。

---

## 为什么使用 LLM

### 传统 NLP vs LLM 对比

| 特性 | 传统 NLP (VADER/TextBlob) | LLM (GPT/Gemini/Claude) |
|------|---------------------------|-------------------------|
| 上下文理解 | ❌ 无法理解反讽、暗示 | ✅ 深度理解上下文语义 |
| 专业术语 | ❌ 不懂加密货币术语 | ✅ 理解 "HODL", "diamond hands", "to the moon" |
| 分析方式 | ❌ 简单词汇匹配 | ✅ 理解复杂句子结构 |
| 速度 | ✅ 毫秒级响应 | ❌ 秒级响应 |
| 成本 | ✅ 免费 | ❌ 按 token 计费 |
| API 依赖 | ✅ 本地运行 | ❌ 需要网络连接 |

### LLM 的独特优势

1. **理解加密货币俚语**
   - "Diamond hands" = 坚定持有
   - "Paper hands" = 容易恐慌抛售
   - "WAGMI" = We're All Gonna Make It (看涨)
   - "NGMI" = Not Gonna Make It (看跌)

2. **识别反讽和 FOMO/FUD**
   - 能够识别 "This is definitely going to $100k" 的反讽语气
   - 区分真实恐惧和夸张表达

3. **多语言支持**
   - 可以分析英语、中文等多种语言的社交媒体内容

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     LLM 社交媒体情绪分析系统                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │   数据采集    │    │   数据预处理  │    │      LLM 分析         │   │
│  │              │    │              │    │                      │   │
│  │ • Reddit    │ -> │ • 去重过滤    │ -> │ • OpenAI GPT        │   │
│  │ • Twitter   │    │ • 文本截断    │    │ • Google Gemini     │   │
│  │ • News      │    │ • 噪音清洗    │    │ • Anthropic Claude  │   │
│  │ • Telegram  │    │              │    │                      │   │
│  └──────────────┘    └──────────────┘    └──────────────────────┘   │
│                                                   │                 │
│                                                   ▼                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                       情绪分析结果                             │   │
│  │                                                              │   │
│  │  {                                                           │   │
│  │    "score": -35,        // -100 到 100                       │   │
│  │    "label": "bearish",  // bullish/bearish/neutral           │   │
│  │    "confidence": 78,    // 0 到 100                          │   │
│  │    "summary": "市场情绪偏向悲观，FUD 情绪明显",                  │   │
│  │    "key_topics": ["监管", "ETF", "鲸鱼动向"]                   │   │
│  │  }                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                   │                                 │
│                                   ▼                                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Freqtrade 策略集成                          │   │
│  │                                                              │   │
│  │  情绪分数 + 技术指标 (RSI, MACD, 布林带) = 交易信号             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## LLM 提供商对比

### 主要提供商

| 提供商 | 推荐模型 | 特点 | 价格 (每1M tokens) |
|--------|----------|------|-------------------|
| **OpenAI** | gpt-4o-mini | 性价比最高，效果好 | 输入 $0.15 / 输出 $0.60 |
| **Google** | gemini-1.5-flash | 速度快，支持长文本 | 输入 $0.075 / 输出 $0.30 |
| **Anthropic** | claude-3-haiku | 安全性高，理解能力强 | 输入 $0.25 / 输出 $1.25 |

### 如何获取 API 密钥

#### OpenAI
1. 访问 https://platform.openai.com/
2. 注册/登录账户
3. 进入 API Keys 页面
4. 点击 "Create new secret key"

#### Google Gemini
1. 访问 https://aistudio.google.com/
2. 使用 Google 账户登录
3. 点击 "Get API Key"
4. 创建新的 API 密钥

#### Anthropic Claude
1. 访问 https://console.anthropic.com/
2. 注册/登录账户
3. 进入 API Keys 页面
4. 创建新密钥

---

## 🆕 现代 LLM 技能 (2025+ 最新特性)

> 以下内容基于 Context7 查询的最新官方文档，包含 OpenAI、Claude、Gemini 的最新 API 特性。

### Structured Outputs (结构化输出) - 推荐

**Structured Outputs** 是最新的 LLM 特性，可以**保证输出符合指定的 JSON Schema**，比传统 JSON Mode 更可靠。

| 提供商 | 特性名称 | 参数 | 状态 |
|--------|----------|------|------|
| **OpenAI** | Structured Outputs | `response_format.type = "json_schema"` | ✅ 正式版 |
| **Claude** | JSON Outputs + Strict Tool Use | `output_format.type = "json_schema"` | ✅ Beta |
| **Gemini** | Response JSON Schema | `responseMimeType = "application/json"` | ✅ 正式版 |

### Tool Use / Function Calling (工具调用)

让 LLM 能够调用外部工具获取实时数据，非常适合情绪分析场景：

```python
# 示例：让 LLM 调用工具获取实时加密货币数据
tools = [
    {
        "name": "get_crypto_price",
        "description": "获取加密货币的实时价格",
        "input_schema": {
            "type": "object",
            "properties": {
                "symbol": {"type": "string", "description": "币种符号如 BTC, ETH"},
            },
            "required": ["symbol"]
        }
    },
    {
        "name": "get_social_sentiment",
        "description": "获取社交媒体情绪分数",
        "input_schema": {
            "type": "object", 
            "properties": {
                "coin": {"type": "string"},
                "timeframe": {"type": "string", "enum": ["1h", "24h", "7d"]}
            },
            "required": ["coin"]
        }
    }
]
```

### 三大提供商最新代码对比

#### OpenAI - Structured Outputs (推荐)

```python
# OpenAI 最新 Structured Outputs API (2025)
# 来源: https://platform.openai.com/docs/guides/structured-outputs

response = client.chat.completions.create(
    model="gpt-4o-mini",  # 或 gpt-4o
    messages=[
        {"role": "system", "content": "你是加密货币情绪分析师"},
        {"role": "user", "content": prompt}
    ],
    response_format={
        "type": "json_schema",  # 新版推荐，比 json_object 更可靠
        "json_schema": {
            "name": "sentiment_analysis",
            "strict": True,  # 严格模式：100% 符合 schema
            "schema": {
                "type": "object",
                "properties": {
                    "score": {"type": "integer", "minimum": -100, "maximum": 100},
                    "label": {"type": "string", "enum": ["bullish", "bearish", "neutral"]},
                    "confidence": {"type": "integer", "minimum": 0, "maximum": 100},
                    "summary": {"type": "string"},
                    "key_topics": {"type": "array", "items": {"type": "string"}}
                },
                "required": ["score", "label", "confidence", "summary", "key_topics"],
                "additionalProperties": False
            }
        }
    }
)
```

#### Claude - JSON Outputs + Strict Tool Use (2025 Beta)

```python
# Claude 最新 Structured Outputs API
# 来源: https://platform.claude.com/docs/en/build-with-claude/structured-outputs

import anthropic

client = anthropic.Anthropic()

response = client.beta.messages.create(
    model="claude-sonnet-4-5",  # 最新模型
    betas=["structured-outputs-2025-11-13"],  # 启用 beta 特性
    max_tokens=1024,
    messages=[{"role": "user", "content": prompt}],
    output_format={
        "type": "json_schema",
        "schema": {
            "type": "object",
            "properties": {
                "score": {"type": "integer"},
                "label": {"type": "string"},
                "confidence": {"type": "integer"},
                "summary": {"type": "string"},
                "key_topics": {"type": "array", "items": {"type": "string"}},
                "fomo_level": {"type": "integer"},
                "fud_level": {"type": "integer"}
            },
            "required": ["score", "label", "confidence", "summary", "key_topics"],
            "additionalProperties": False
        }
    }
)
```

#### Gemini - Response JSON Schema

```python
# Gemini 最新 Structured Output API
# 来源: https://ai.google.dev/gemini-api/docs/structured-output

import google.generativeai as genai

model = genai.GenerativeModel("gemini-2.0-flash")  # 或 gemini-1.5-flash

response = model.generate_content(
    prompt,
    generation_config={
        "response_mime_type": "application/json",  # 强制 JSON 输出
        "response_schema": {  # 定义输出结构
            "type": "object",
            "properties": {
                "score": {"type": "integer"},
                "label": {"type": "string"},
                "confidence": {"type": "integer"},
                "summary": {"type": "string"},
                "key_topics": {
                    "type": "array",
                    "items": {"type": "string"}
                }
            },
            "required": ["score", "label", "confidence", "summary"]
        }
    }
)
```

### 新特性 vs 传统方式对比

| 特性 | 传统 JSON Mode | Structured Outputs (新) |
|------|---------------|------------------------|
| 输出格式 | 可能不符合预期 | ✅ **100% 符合 Schema** |
| 字段缺失 | 可能缺少必需字段 | ✅ **必需字段强制输出** |
| 类型安全 | 类型可能错误 | ✅ **类型严格验证** |
| 解析失败 | 需要 try-catch | ✅ **几乎不会失败** |
| 调试难度 | 较高 | ✅ **错误信息清晰** |

---

## 🤖 Agent Skills 架构 (Anthropic 最佳实践)

> 以下内容参考 Anthropic 的 [Agent Skills](https://github.com/anthropics/skills) 开源仓库，采用模块化设计让 AI 代理具备专业能力。

### 什么是 Agent Skills？

**Agent Skills** 是 Anthropic 推出的一种模块化能力扩展系统，核心理念是：

> 💡 把专业知识打包成可复用的技能模块，就像给新员工写入职指南一样。

### 渐进式披露架构

Skills 采用三层加载机制，按需使用，节省 Token：

```
Level 1: 元数据 (始终加载) ──────────────────────────────────────
         name: crypto-sentiment-analyzer
         description: 分析加密货币社交媒体情绪

Level 2: 核心指令 (触发时加载) ──────────────────────────────────
         SKILL.md 正文：分析步骤、使用场景、评分标准

Level 3: 扩展资源 (按需加载) ──────────────────────────────────
         prompts/sentiment_analysis.md - Prompt 模板
         scripts/llm_analyzer.py - LLM 分析器代码
         scripts/collect_social_data.py - 数据采集脚本
```

### 我们的 Skill 结构

本项目已创建符合 Anthropic Skills 规范的技能目录：

```
user_data/strategies/crypto-sentiment-skill/
├── SKILL.md                           # 技能定义文件
├── prompts/
│   └── sentiment_analysis.md          # Prompt 模板和 JSON Schema
└── scripts/
    ├── llm_analyzer.py                # LLM 分析器代码
    └── collect_social_data.py         # 数据采集脚本
```

### SKILL.md 标准格式

```yaml
---
name: crypto-sentiment-analyzer
description: |
  分析加密货币社交媒体情绪的专业技能。
  当需要获取 BTC、ETH 等加密货币的市场情绪时使用此技能。
---

# 加密货币情绪分析技能

## 使用场景
- 在交易策略中获取市场情绪
- 检测 FOMO/FUD 情绪强度

## 操作步骤
1. 采集社交媒体数据
2. 使用 LLM 分析情绪
3. 生成结构化报告

## 评分标准
...
```

### Skills 设计原则

| 原则 | 说明 | 应用 |
|------|------|------|
| **渐进式披露** | 按需加载信息 | Prompt 模板放在单独文件 |
| **代码执行分离** | 脚本在 context 外执行 | 数据采集脚本独立运行 |
| **模块化设计** | 技能可组合复用 | 情绪分析、采集、策略分开 |
| **元数据驱动** | name+description 决定触发 | 清晰的技能描述 |

> 📚 **参考**: [Anthropic Agent Skills 仓库](https://github.com/anthropics/skills) | [Agent Skills 官方博客](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

---

## 核心代码实现

### LLM 情绪分析器

创建文件 `user_data/strategies/llm_sentiment_analyzer.py`:

```python
"""
llm_sentiment_analyzer.py
LLM 社交媒体情绪分析器
支持多种 LLM 提供商: OpenAI, Google Gemini, Anthropic Claude
"""

import json
import logging
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime

import requests

logger = logging.getLogger(__name__)


@dataclass
class SentimentResult:
    """情绪分析结果数据类"""
    score: float              # 情绪分数 -100 到 100
    confidence: float         # 置信度 0 到 100
    label: str                # 标签: bullish/bearish/neutral
    summary: str              # 情绪总结
    key_topics: list          # 关键话题
    sample_size: int          # 分析的样本数量
    timestamp: datetime       # 分析时间


class BaseLLMAnalyzer(ABC):
    """LLM 分析器基类"""
    
    @abstractmethod
    def analyze(self, texts: list, coin: str) -> SentimentResult:
        """分析文本情绪"""
        pass
    
    def _build_prompt(self, texts: list, coin: str) -> str:
        """
        构建分析提示词 (Prompt Engineering 核心)
        这是整个系统最关键的部分！
        """
        # 合并文本，限制长度
        combined = "\n---\n".join(texts[:30])  # 最多30条
        if len(combined) > 8000:
            combined = combined[:8000] + "...[截断]"
        
        prompt = f"""
你是一位专业的加密货币市场情绪分析师。请分析以下关于 {coin} 的社交媒体帖子。

## 分析要求
1. 识别整体市场情绪 (看涨/看跌/中性)
2. 检测是否有反讽、FOMO、FUD 等情绪
3. 识别关键话题和事件
4. 评估社区共识程度

## 社交媒体帖子
{combined}

## 输出格式 (必须是有效的 JSON)
请严格按照以下 JSON 格式输出，不要包含其他内容:
{{
    "score": <整数, -100到100, 负数=看跌, 0=中性, 正数=看涨>,
    "confidence": <整数, 0到100, 分析置信度>,
    "label": "<bullish/bearish/neutral>",
    "summary": "<一句话中文总结当前市场情绪>",
    "key_topics": ["<话题1>", "<话题2>", "<话题3>"],
    "fomo_level": <整数, 0到10, FOMO情绪强度>,
    "fud_level": <整数, 0到10, FUD情绪强度>,
    "whale_mentions": <是否提到大户/机构动向, true/false>
}}

## 评分参考
- -100 到 -60: 极度恐慌, 大量恐惧性抛售讨论
- -60 到 -20: 偏空, 担忧情绪明显
- -20 到 20: 中性, 观望为主
- 20 到 60: 偏多, 乐观情绪占主导
- 60 到 100: 极度贪婪, FOMO 情绪严重
"""
        return prompt


class OpenAIAnalyzer(BaseLLMAnalyzer):
    """OpenAI GPT 分析器"""
    
    def __init__(self, api_key: str, model: str = "gpt-4o-mini"):
        """
        初始化 OpenAI 分析器
        
        参数:
            api_key: OpenAI API 密钥
            model: 模型名称
                - gpt-4o: 最强，但最贵
                - gpt-4o-mini: 性价比高 [推荐]
                - gpt-3.5-turbo: 便宜但效果一般
        """
        self.api_key = api_key
        self.model = model
        self.base_url = "https://api.openai.com/v1/chat/completions"
    
    def analyze(self, texts: list, coin: str) -> SentimentResult:
        """使用 OpenAI API 分析情绪"""
        try:
            prompt = self._build_prompt(texts, coin)
            
            headers = {
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json"
            }
            
            payload = {
                "model": self.model,
                "messages": [
                    {
                        "role": "system",
                        "content": "你是专业的加密货币情绪分析师。"
                    },
                    {
                        "role": "user", 
                        "content": prompt
                    }
                ],
                "temperature": 0.3,  # 低温度 = 更稳定的输出
                "max_tokens": 500,
                # 🆕 使用最新的 Structured Outputs API (2025)
                # 来源: https://platform.openai.com/docs/guides/structured-outputs
                "response_format": {
                    "type": "json_schema",  # 新版推荐，比 json_object 更可靠
                    "json_schema": {
                        "name": "sentiment_analysis",
                        "strict": True,  # 严格模式：100% 符合 schema
                        "schema": {
                            "type": "object",
                            "properties": {
                                "score": {"type": "integer"},
                                "label": {"type": "string", "enum": ["bullish", "bearish", "neutral"]},
                                "confidence": {"type": "integer"},
                                "summary": {"type": "string"},
                                "key_topics": {"type": "array", "items": {"type": "string"}},
                                "fomo_level": {"type": "integer"},
                                "fud_level": {"type": "integer"},
                                "whale_mentions": {"type": "boolean"}
                            },
                            "required": ["score", "label", "confidence", "summary", "key_topics"],
                            "additionalProperties": False
                        }
                    }
                }
            }
            
            response = requests.post(
                self.base_url,
                headers=headers,
                json=payload,
                timeout=30
            )
            response.raise_for_status()
            
            result = response.json()
            content = result['choices'][0]['message']['content']
            data = json.loads(content)
            
            return SentimentResult(
                score=data.get('score', 0),
                confidence=data.get('confidence', 50),
                label=data.get('label', 'neutral'),
                summary=data.get('summary', ''),
                key_topics=data.get('key_topics', []),
                sample_size=len(texts),
                timestamp=datetime.now()
            )
            
        except Exception as e:
            logger.error(f"OpenAI 分析失败: {e}")
            return self._default_result(str(e))
    
    def _default_result(self, error_msg: str = '') -> SentimentResult:
        """返回默认结果"""
        return SentimentResult(
            score=0,
            confidence=0,
            label='neutral',
            summary=f'分析失败: {error_msg}' if error_msg else '无数据',
            key_topics=[],
            sample_size=0,
            timestamp=datetime.now()
        )


class GeminiAnalyzer(BaseLLMAnalyzer):
    """Google Gemini 分析器"""
    
    def __init__(self, api_key: str, model: str = "gemini-1.5-flash"):
        """
        初始化 Gemini 分析器
        
        参数:
            api_key: Google AI API 密钥
            model: 模型名称
                - gemini-1.5-pro: 最强
                - gemini-1.5-flash: 快速且便宜 [推荐]
        """
        self.api_key = api_key
        self.model = model
        self.base_url = f"https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent"
    
    def analyze(self, texts: list, coin: str) -> SentimentResult:
        """使用 Gemini API 分析情绪"""
        try:
            prompt = self._build_prompt(texts, coin)
            
            payload = {
                "contents": [{
                    "parts": [{"text": prompt}]
                }],
                "generationConfig": {
                    "temperature": 0.3,
                    "maxOutputTokens": 500,
                    "responseMimeType": "application/json"
                }
            }
            
            response = requests.post(
                f"{self.base_url}?key={self.api_key}",
                json=payload,
                timeout=30
            )
            response.raise_for_status()
            
            result = response.json()
            content = result['candidates'][0]['content']['parts'][0]['text']
            data = json.loads(content)
            
            return SentimentResult(
                score=data.get('score', 0),
                confidence=data.get('confidence', 50),
                label=data.get('label', 'neutral'),
                summary=data.get('summary', ''),
                key_topics=data.get('key_topics', []),
                sample_size=len(texts),
                timestamp=datetime.now()
            )
            
        except Exception as e:
            logger.error(f"Gemini 分析失败: {e}")
            return self._default_result(str(e))
    
    def _default_result(self, error_msg: str = '') -> SentimentResult:
        """返回默认结果"""
        return SentimentResult(
            score=0, confidence=0, label='neutral',
            summary=f'分析失败: {error_msg}' if error_msg else '无数据',
            key_topics=[], sample_size=0, timestamp=datetime.now()
        )


class ClaudeAnalyzer(BaseLLMAnalyzer):
    """Anthropic Claude 分析器"""
    
    def __init__(self, api_key: str, model: str = "claude-3-haiku-20240307"):
        """
        初始化 Claude 分析器
        
        参数:
            api_key: Anthropic API 密钥
            model: 模型名称
                - claude-3-opus: 最强
                - claude-3-sonnet: 平衡
                - claude-3-haiku: 快速便宜 [推荐]
        """
        self.api_key = api_key
        self.model = model
        self.base_url = "https://api.anthropic.com/v1/messages"
    
    def analyze(self, texts: list, coin: str) -> SentimentResult:
        """使用 Claude API 分析情绪"""
        try:
            prompt = self._build_prompt(texts, coin)
            
            headers = {
                "x-api-key": self.api_key,
                "anthropic-version": "2023-06-01",
                "Content-Type": "application/json"
            }
            
            payload = {
                "model": self.model,
                "max_tokens": 500,
                "messages": [
                    {"role": "user", "content": prompt}
                ]
            }
            
            response = requests.post(
                self.base_url,
                headers=headers,
                json=payload,
                timeout=30
            )
            response.raise_for_status()
            
            result = response.json()
            content = result['content'][0]['text']
            
            # 提取 JSON
            import re
            json_match = re.search(r'\{[\s\S]*\}', content)
            if json_match:
                data = json.loads(json_match.group())
            else:
                raise ValueError("无法提取 JSON")
            
            return SentimentResult(
                score=data.get('score', 0),
                confidence=data.get('confidence', 50),
                label=data.get('label', 'neutral'),
                summary=data.get('summary', ''),
                key_topics=data.get('key_topics', []),
                sample_size=len(texts),
                timestamp=datetime.now()
            )
            
        except Exception as e:
            logger.error(f"Claude 分析失败: {e}")
            return self._default_result(str(e))
    
    def _default_result(self, error_msg: str = '') -> SentimentResult:
        """返回默认结果"""
        return SentimentResult(
            score=0, confidence=0, label='neutral',
            summary=f'分析失败: {error_msg}' if error_msg else '无数据',
            key_topics=[], sample_size=0, timestamp=datetime.now()
        )


def create_analyzer(provider: str, api_key: str, model: str = None) -> BaseLLMAnalyzer:
    """
    工厂函数：创建 LLM 分析器
    
    参数:
        provider: 提供商名称 (openai/gemini/claude)
        api_key: API 密钥
        model: 可选，模型名称
    """
    if provider.lower() == 'openai':
        return OpenAIAnalyzer(api_key, model or 'gpt-4o-mini')
    elif provider.lower() == 'gemini':
        return GeminiAnalyzer(api_key, model or 'gemini-1.5-flash')
    elif provider.lower() == 'claude':
        return ClaudeAnalyzer(api_key, model or 'claude-3-haiku-20240307')
    else:
        raise ValueError(f"不支持的提供商: {provider}")
```

---

### 社交媒体数据采集器

创建文件 `user_data/strategies/social_data_collector.py`:

```python
"""
social_data_collector.py
社交媒体数据采集器
"""

import requests
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional, List


@dataclass
class SocialPost:
    """社交媒体帖子数据类"""
    source: str          # reddit/twitter/news
    title: str           # 标题
    content: str         # 内容
    score: int           # 点赞/upvotes
    comments: int        # 评论数
    created_at: datetime # 发布时间
    url: str             # 原始链接


class RedditCollector:
    """
    Reddit 数据采集器
    
    注册 Reddit App 步骤:
    1. 登录 https://www.reddit.com/prefs/apps
    2. 滚动到底部，点击 "create app" 或 "create another app"
    3. 填写表单:
       - name: FreqtradeBot (任意名称)
       - 选择 "script"
       - redirect uri: http://localhost:8080
    4. 点击 "create app"
    5. 记录 client_id (在应用名称下方) 和 secret
    """
    
    def __init__(self, client_id: str, client_secret: str, user_agent: str = "FreqtradeBot/1.0"):
        """
        初始化 Reddit API
        
        参数:
            client_id: Reddit App 的 client_id
            client_secret: Reddit App 的 secret
            user_agent: 用户代理字符串
        """
        try:
            import praw
            self.reddit = praw.Reddit(
                client_id=client_id,
                client_secret=client_secret,
                user_agent=user_agent
            )
            self.enabled = True
        except ImportError:
            print("警告: 请安装 praw 库: pip install praw")
            self.enabled = False
    
    def collect(self, coin: str, limit: int = 50, hours: int = 24) -> List[SocialPost]:
        """
        采集 Reddit 帖子
        
        参数:
            coin: 币种名称，如 'BTC', 'Bitcoin', 'ETH'
            limit: 最大采集数量
            hours: 时间范围 (小时)
        
        返回:
            SocialPost 列表
        """
        if not self.enabled:
            return []
        
        posts = []
        
        # 相关 subreddits
        subreddits = [
            'CryptoCurrency',    # 综合加密货币
            'Bitcoin',           # 比特币专区
            'ethereum',          # 以太坊专区
            'altcoin',           # 山寨币
            'CryptoMarkets',     # 市场讨论
            'binance',           # 币安
            'defi'               # DeFi
        ]
        
        # 搜索关键词映射
        keyword_map = {
            'BTC': ['Bitcoin', 'btc', 'BTC'],
            'ETH': ['Ethereum', 'eth', 'ETH'],
            'SOL': ['Solana', 'sol', 'SOL'],
            'BNB': ['Binance', 'bnb', 'BNB'],
            'XRP': ['Ripple', 'xrp', 'XRP'],
        }
        
        keywords = keyword_map.get(coin.upper(), [coin])
        cutoff_time = datetime.utcnow() - timedelta(hours=hours)
        
        for subreddit_name in subreddits:
            try:
                subreddit = self.reddit.subreddit(subreddit_name)
                
                for keyword in keywords:
                    # 搜索帖子
                    for submission in subreddit.search(
                        keyword, 
                        limit=limit // len(subreddits), 
                        time_filter='day',
                        sort='hot'
                    ):
                        created = datetime.utcfromtimestamp(submission.created_utc)
                        
                        if created < cutoff_time:
                            continue
                        
                        posts.append(SocialPost(
                            source='reddit',
                            title=submission.title,
                            content=submission.selftext[:1000] if submission.selftext else '',
                            score=submission.score,
                            comments=submission.num_comments,
                            created_at=created,
                            url=f"https://reddit.com{submission.permalink}"
                        ))
                        
            except Exception as e:
                print(f"采集 r/{subreddit_name} 失败: {e}")
                continue
        
        # 按热度排序 (点赞 + 评论数)
        posts.sort(key=lambda x: x.score + x.comments * 2, reverse=True)
        
        # 去重 (基于标题)
        seen_titles = set()
        unique_posts = []
        for post in posts:
            if post.title not in seen_titles:
                seen_titles.add(post.title)
                unique_posts.append(post)
        
        return unique_posts[:limit]


class CryptoPanicCollector:
    """
    CryptoPanic 新闻采集器
    
    官网: https://cryptopanic.com/developers/api/
    免费版每月 5000 次请求
    """
    
    def __init__(self, api_key: Optional[str] = None):
        """
        初始化 CryptoPanic 采集器
        
        参数:
            api_key: CryptoPanic API 密钥 (可选，无密钥也能用但有限制)
        """
        self.api_key = api_key
        self.base_url = "https://cryptopanic.com/api/v1/posts/"
    
    def collect(self, coin: str, limit: int = 20) -> List[SocialPost]:
        """
        采集加密货币新闻
        
        参数:
            coin: 币种符号，如 'BTC', 'ETH'
            limit: 最大采集数量
        """
        posts = []
        
        try:
            params = {
                'currencies': coin.upper(),
                'filter': 'hot',      # hot/rising/bullish/bearish
                'public': 'true',
                'kind': 'news'        # news/media
            }
            
            if self.api_key:
                params['auth_token'] = self.api_key
            
            response = requests.get(self.base_url, params=params, timeout=10)
            response.raise_for_status()
            data = response.json()
            
            for item in data.get('results', [])[:limit]:
                # 解析时间
                created_str = item.get('created_at', '')
                if created_str:
                    created_at = datetime.fromisoformat(created_str.replace('Z', '+00:00'))
                else:
                    created_at = datetime.now()
                
                posts.append(SocialPost(
                    source='cryptopanic',
                    title=item.get('title', ''),
                    content=item.get('title', ''),  # CryptoPanic 只有标题
                    score=item.get('votes', {}).get('positive', 0) - item.get('votes', {}).get('negative', 0),
                    comments=item.get('comments', 0),
                    created_at=created_at,
                    url=item.get('url', '')
                ))
                
        except Exception as e:
            print(f"CryptoPanic 采集失败: {e}")
        
        return posts


class MultiSourceCollector:
    """多源数据采集器 - 聚合多个数据源"""
    
    def __init__(
        self,
        reddit_client_id: str = None,
        reddit_client_secret: str = None,
        cryptopanic_api_key: str = None
    ):
        """初始化多源采集器"""
        self.collectors = []
        
        # 添加 Reddit
        if reddit_client_id and reddit_client_secret:
            self.collectors.append(
                ('reddit', RedditCollector(reddit_client_id, reddit_client_secret))
            )
        
        # 添加 CryptoPanic
        self.collectors.append(
            ('cryptopanic', CryptoPanicCollector(cryptopanic_api_key))
        )
    
    def collect(self, coin: str, limit: int = 50) -> List[SocialPost]:
        """从所有数据源采集数据"""
        all_posts = []
        
        for name, collector in self.collectors:
            try:
                posts = collector.collect(coin, limit=limit // len(self.collectors))
                all_posts.extend(posts)
                print(f"从 {name} 采集了 {len(posts)} 条帖子")
            except Exception as e:
                print(f"{name} 采集失败: {e}")
        
        # 按时间排序
        all_posts.sort(key=lambda x: x.created_at, reverse=True)
        
        return all_posts[:limit]
```

---

### Freqtrade 策略集成

创建文件 `user_data/strategies/LLMSentimentStrategy.py`:

```python
"""
LLMSentimentStrategy.py
使用 LLM 分析社交媒体情绪的 Freqtrade 策略

配置步骤:
1. 将此文件放到 user_data/strategies/ 目录
2. 将 llm_sentiment_analyzer.py 和 social_data_collector.py 也放到同一目录
3. 在 config.json 中配置 API 密钥
4. 运行: freqtrade trade --strategy LLMSentimentStrategy
"""

import logging
from datetime import datetime, timedelta
from typing import Optional

import numpy as np
import pandas as pd
from pandas import DataFrame

from freqtrade.strategy import IStrategy, IntParameter, DecimalParameter
import talib.abstract as ta

# 导入自定义模块
try:
    from llm_sentiment_analyzer import create_analyzer, SentimentResult
    from social_data_collector import MultiSourceCollector
    LLM_AVAILABLE = True
except ImportError:
    LLM_AVAILABLE = False
    print("警告: 未找到 LLM 模块，将使用模拟情绪")

logger = logging.getLogger(__name__)


class LLMSentimentStrategy(IStrategy):
    """
    基于 LLM 社交媒体情绪分析的交易策略
    
    核心逻辑:
    1. 定期采集社交媒体数据 (Reddit, 新闻)
    2. 使用 LLM 分析市场情绪
    3. 结合技术指标生成交易信号
    
    情绪交易逻辑 (逆向操作):
    - 极度恐惧 + 技术超卖 = 买入机会
    - 极度贪婪 + 技术超买 = 卖出机会
    """
    
    INTERFACE_VERSION = 3
    
    # ========== 基础配置 ==========
    timeframe = '1h'           # 1小时K线
    can_short = False          # 不做空
    startup_candle_count = 50  # 需要50根K线预热
    
    # ========== 风险管理 ==========
    minimal_roi = {
        "0": 0.05,     # 立即: 5% 止盈
        "60": 0.03,    # 60分钟后: 3% 止盈
        "120": 0.01,   # 120分钟后: 1% 止盈
        "240": 0       # 240分钟后: 保本出场
    }
    
    stoploss = -0.08           # 8% 止损
    
    trailing_stop = True                      # 启用移动止损
    trailing_stop_positive = 0.02             # 盈利达到2%后启动
    trailing_stop_positive_offset = 0.03      # 从3%开始追踪
    trailing_only_offset_is_reached = True    # 只在达到offset后追踪
    
    # ========== 可优化参数 ==========
    # 情绪阈值 (使用 Hyperopt 优化)
    sentiment_buy_threshold = IntParameter(
        -80, -20, default=-40, space='buy', optimize=True,
        load=True
    )
    sentiment_sell_threshold = IntParameter(
        20, 80, default=50, space='sell', optimize=True,
        load=True
    )
    
    # RSI 阈值
    rsi_buy = IntParameter(20, 40, default=30, space='buy', optimize=True)
    rsi_sell = IntParameter(60, 80, default=70, space='sell', optimize=True)
    
    # 情绪权重 (技术面 vs 情绪面)
    sentiment_weight = DecimalParameter(
        0.2, 0.8, default=0.4, space='buy', optimize=True,
        load=True
    )
    
    def __init__(self, config: dict) -> None:
        super().__init__(config)
        
        # 情绪缓存
        self._sentiment_cache = {}
        self._cache_duration = timedelta(hours=1)  # 缓存1小时
        
        # 初始化 LLM 和数据采集器
        if LLM_AVAILABLE:
            self._init_components(config)
        else:
            self.llm_analyzer = None
            self.data_collector = None
    
    def _init_components(self, config: dict):
        """初始化 LLM 分析器和数据采集器"""
        
        # 从 config.json 读取配置
        llm_config = config.get('llm_sentiment', {})
        
        # LLM 分析器
        provider = llm_config.get('provider', 'gemini')
        api_key = llm_config.get('api_key', '')
        model = llm_config.get('model', '')
        
        if api_key:
            try:
                self.llm_analyzer = create_analyzer(provider, api_key, model)
                logger.info(f"已初始化 {provider} LLM 分析器")
            except Exception as e:
                logger.error(f"初始化 LLM 失败: {e}")
                self.llm_analyzer = None
        else:
            logger.warning("未配置 LLM API 密钥")
            self.llm_analyzer = None
        
        # 数据采集器
        self.data_collector = MultiSourceCollector(
            reddit_client_id=llm_config.get('reddit_client_id'),
            reddit_client_secret=llm_config.get('reddit_client_secret'),
            cryptopanic_api_key=llm_config.get('cryptopanic_api_key')
        )
    
    def get_sentiment(self, coin: str) -> dict:
        """
        获取币种的社交媒体情绪
        
        参数:
            coin: 币种符号，如 'BTC'
        
        返回:
            {
                'score': -100 到 100 的情绪分数,
                'label': 'bullish'/'bearish'/'neutral',
                'confidence': 0 到 100 的置信度,
                'summary': 情绪总结文本
            }
        """
        # 默认值
        default = {
            'score': 0,
            'label': 'neutral',
            'confidence': 0,
            'summary': '无数据'
        }
        
        if not self.llm_analyzer or not self.data_collector:
            return default
        
        # 检查缓存
        cache_key = f"{coin}_{datetime.now().strftime('%Y%m%d%H')}"
        if cache_key in self._sentiment_cache:
            logger.info(f"使用缓存的 {coin} 情绪数据")
            return self._sentiment_cache[cache_key]
        
        try:
            # 采集数据
            posts = self.data_collector.collect(coin, limit=30)
            
            if not posts:
                logger.warning(f"未采集到 {coin} 的社交媒体数据")
                return default
            
            # 准备文本
            texts = [f"{p.title}\n{p.content}" for p in posts]
            
            # LLM 分析
            result = self.llm_analyzer.analyze(texts, coin)
            
            sentiment = {
                'score': result.score,
                'label': result.label,
                'confidence': result.confidence,
                'summary': result.summary
            }
            
            # 缓存结果
            self._sentiment_cache[cache_key] = sentiment
            
            logger.info(
                f"{coin} 情绪: {sentiment['label']} "
                f"(分数={sentiment['score']}, 置信度={sentiment['confidence']}%)"
            )
            
            return sentiment
            
        except Exception as e:
            logger.error(f"情绪分析失败: {e}")
            return default
    
    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        添加技术指标和情绪指标
        
        注意: 根据 Freqtrade 官方文档 (https://www.freqtrade.io/en/develop/hyperopt),
        indicators 只会在每个交易对开始时计算一次，因此所有可能用到的指标都应该在这里计算。
        """
        # ========== 技术指标 (官方推荐) ==========
        
        # ADX - 趋势强度指标 (官方文档推荐)
        dataframe['adx'] = ta.ADX(dataframe)
        
        # RSI - 相对强弱指标
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        
        # EMA - 指数移动平均
        dataframe['ema_20'] = ta.EMA(dataframe, timeperiod=20)
        dataframe['ema_50'] = ta.EMA(dataframe, timeperiod=50)
        dataframe['ema_200'] = ta.EMA(dataframe, timeperiod=200)
        
        # MACD - 官方文档标准写法
        macd = ta.MACD(dataframe)
        dataframe['macd'] = macd['macd']
        dataframe['macd_signal'] = macd['macdsignal']
        dataframe['macd_hist'] = macd['macdhist']
        
        # 布林带 - 官方文档标准写法
        bollinger = ta.BBANDS(dataframe, timeperiod=20, nbdevup=2, nbdevdn=2)
        dataframe['bb_lowerband'] = bollinger['lowerband']
        dataframe['bb_middleband'] = bollinger['middleband']
        dataframe['bb_upperband'] = bollinger['upperband']
        dataframe['bb_percent'] = (dataframe['close'] - dataframe['bb_lowerband']) / \
                                  (dataframe['bb_upperband'] - dataframe['bb_lowerband'])
        
        # ATR (波动率)
        dataframe['atr'] = ta.ATR(dataframe, timeperiod=14)
        
        # 成交量指标
        dataframe['volume_sma'] = dataframe['volume'].rolling(window=20).mean()
        dataframe['volume_ratio'] = dataframe['volume'] / dataframe['volume_sma']
        
        # ========== 情绪指标 ==========
        
        if self.dp and self.dp.runmode.value in ('live', 'dry_run'):
            # 实盘/模拟盘: 获取实时情绪
            coin = metadata['pair'].split('/')[0]
            sentiment = self.get_sentiment(coin)
            
            # 归一化到 0-100
            dataframe['sentiment_score'] = (sentiment['score'] + 100) / 2
            dataframe['sentiment_confidence'] = sentiment['confidence']
        else:
            # 回测: 使用技术指标模拟情绪
            # RSI 反转 + 成交量作为代理
            dataframe['sentiment_score'] = (
                (100 - dataframe['rsi']) * 0.7 +  # RSI 越低=情绪越恐惧
                np.clip(dataframe['volume_ratio'] * 30, 0, 100) * 0.3
            )
            dataframe['sentiment_confidence'] = 50
        
        return dataframe
    
    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        入场信号 (INTERFACE_VERSION 3 标准格式)
        
        根据官方文档 (https://www.freqtrade.io/en/develop/strategy-customization):
        - 使用 'enter_long' 和 'enter_short' 列标记入场信号
        - 使用 'enter_tag' 列标记入场原因，便于分析
        
        买入条件:
        1. 情绪偏恐惧 (逆向操作)
        2. RSI 超卖
        3. 价格在布林带下轨附近
        4. 趋势支撑 (价格在 EMA200 上方)
        """
        # 情绪阈值 (归一化后的)
        sentiment_threshold = (self.sentiment_buy_threshold.value + 100) / 2
        
        # 官方推荐格式: 同时设置 enter_long 和 enter_tag
        dataframe.loc[
            (
                # 情绪条件: 市场恐惧
                (dataframe['sentiment_score'] < sentiment_threshold) &
                
                # 技术条件: RSI 超卖
                (dataframe['rsi'] < self.rsi_buy.value) &
                
                # 价格条件: 接近布林带下轨 (超卖区)
                (dataframe['bb_percent'] < 0.2) &
                
                # 趋势条件: 保持在长期均线上方 (大趋势向上)
                (dataframe['close'] > dataframe['ema_200']) &
                
                # ADX 趋势确认 (官方推荐)
                (dataframe['adx'] > 20) &
                
                # 成交量确认 (官方文档: Make sure Volume is not 0)
                (dataframe['volume'] > 0) &
                (dataframe['volume_ratio'] > 0.5)  # 成交量不能太低
            ),
            ['enter_long', 'enter_tag']
        ] = (1, 'fear_oversold')  # 标签: 恐惧+超卖
        
        return dataframe
    
    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        出场信号 (INTERFACE_VERSION 3 标准格式)
        
        根据官方文档:
        - 使用 'exit_long' 和 'exit_short' 列标记出场信号
        - 使用 'exit_tag' 列标记出场原因
        
        卖出条件:
        1. 情绪过于乐观/贪婪
        2. RSI 超买
        3. 价格在布林带上轨附近
        """
        # 情绪阈值 (归一化后的)
        sentiment_threshold = (self.sentiment_sell_threshold.value + 100) / 2
        
        dataframe.loc[
            (
                # 情绪条件: 市场贪婪
                (dataframe['sentiment_score'] > sentiment_threshold) &
                
                # 技术条件: RSI 超买
                (dataframe['rsi'] > self.rsi_sell.value) &
                
                # 价格条件: 接近布林带上轨 (超买区)
                (dataframe['bb_percent'] > 0.8) &
                
                # 成交量确认 (官方文档标准)
                (dataframe['volume'] > 0)
            ),
            ['exit_long', 'exit_tag']
        ] = (1, 'greed_overbought')  # 标签: 贪婪+超买
        
        return dataframe
```

---

## 配置指南

### config.json 配置示例

```json
{
    "strategy": "LLMSentimentStrategy",
    "strategy_path": "user_data/strategies",
    
    "llm_sentiment": {
        "provider": "gemini",
        "api_key": "YOUR_GEMINI_API_KEY",
        "model": "gemini-1.5-flash",
        
        "reddit_client_id": "YOUR_REDDIT_CLIENT_ID",
        "reddit_client_secret": "YOUR_REDDIT_CLIENT_SECRET",
        
        "cryptopanic_api_key": "YOUR_CRYPTOPANIC_API_KEY"
    },
    
    "exchange": {
        "name": "binance",
        "key": "YOUR_EXCHANGE_API_KEY",
        "secret": "YOUR_EXCHANGE_SECRET"
    },
    
    "pairlists": [
        {"method": "StaticPairList"}
    ],
    
    "exchange": {
        "pair_whitelist": [
            "BTC/USDT",
            "ETH/USDT"
        ]
    }
}
```

### 环境变量方式 (更安全)

```bash
# 设置环境变量
export LLM_API_KEY="your-api-key"
export REDDIT_CLIENT_ID="your-reddit-id"
export REDDIT_CLIENT_SECRET="your-reddit-secret"
```

```json
// config.json 中使用环境变量
{
    "llm_sentiment": {
        "api_key": "${LLM_API_KEY}",
        "reddit_client_id": "${REDDIT_CLIENT_ID}",
        "reddit_client_secret": "${REDDIT_CLIENT_SECRET}"
    }
}
```

---

## Prompt Engineering 技巧

### 好的 Prompt 应该包含

1. **角色设定**: 明确告诉 LLM 它是什么角色
2. **任务描述**: 清晰的分析目标
3. **输出格式**: 严格的 JSON 格式要求
4. **评分标准**: 具体的评分参考
5. **边界条件**: 处理边缘情况

### 示例 Prompt 结构

```
你是一位专业的加密货币市场情绪分析师。  <- 角色设定

请分析以下关于 {coin} 的社交媒体帖子。   <- 任务描述

## 分析要求                              <- 具体要求
1. 识别整体市场情绪
2. 检测反讽、FOMO、FUD
3. 识别关键话题

## 社交媒体帖子                          <- 输入数据
{文本内容}

## 输出格式                              <- 输出格式
{JSON 结构}

## 评分参考                              <- 评分标准
-100 到 -60: 极度恐慌
...
```

### 常见优化技巧

| 技巧 | 说明 |
|------|------|
| 低温度 | `temperature: 0.3` 使输出更稳定 |
| 强制 JSON | 使用 `response_format` 选项 |
| 限制长度 | 截断输入文本，控制 token 消耗 |
| 示例输出 | 提供示例 JSON 格式 |

---

## 成本估算

### 每次分析的 Token 估算

| 组成部分 | 大约 Token 数 |
|----------|--------------|
| 系统提示 | ~100 tokens |
| 用户提示 (Prompt) | ~500 tokens |
| 社交媒体内容 (30条) | ~2000 tokens |
| 输出 (JSON) | ~200 tokens |
| **总计** | **~2800 tokens** |

### 成本对比

| 提供商 | 模型 | 每次分析成本 | 每天成本 (24次) | 每月成本 |
|--------|------|-------------|----------------|----------|
| **Google** | gemini-1.5-flash | ~$0.0014 | ~$0.034 | **~$1** |
| **OpenAI** | gpt-4o-mini | ~$0.0006 | ~$0.015 | **~$0.45** |
| **Anthropic** | claude-3-haiku | ~$0.0012 | ~$0.029 | **~$0.87** |
| OpenAI | gpt-4o | ~$0.022 | ~$0.53 | ~$16 |

**推荐**: 使用 **gpt-4o-mini** 或 **gemini-1.5-flash**，月成本不到 1 美元！

---

## 注意事项与最佳实践

### ⚠️ 风险与应对

| 风险 | 应对策略 |
|------|----------|
| **API 失败** | 设置默认值，不依赖单一数据源 |
| **延迟问题** | 使用缓存，每小时更新一次 |
| **回测困难** | 使用技术指标作为情绪代理 |
| **成本失控** | 限制请求频率，设置每日预算 |
| **情绪噪音** | 结合技术指标双重确认 |
| **假消息** | 多源验证，权威来源优先 |

### ✅ 最佳实践

1. **不要过度依赖情绪信号**
   - 情绪只是辅助指标
   - 必须与技术面结合使用
   - 设置合理的权重 (建议 30-50%)

2. **缓存策略**
   - 每小时更新一次即可
   - 缓存失败时使用默认值
   - 避免频繁 API 调用

3. **错误处理**
   - 所有 API 调用都要有 try-catch
   - 失败时返回中性情绪
   - 记录详细日志

4. **回测注意事项**
   - 无法获取历史情绪数据
   - 使用技术指标模拟
   - 回测结果仅供参考

---

## 常见问题

### Q1: 回测时如何处理情绪数据？

回测时无法获取历史社交媒体数据，建议：
- 使用 RSI 反转作为情绪代理
- 或购买历史情绪数据 (Santiment, LunarCrush)
- 或仅在实盘中使用情绪指标

### Q2: 每小时分析一次够用吗？

对于 1h 及以上时间框架的策略，每小时一次足够。社交媒体情绪通常是缓慢变化的，过于频繁的分析反而会增加噪音。

### Q3: 哪个 LLM 效果最好？

根据测试：
- **GPT-4o**: 最准确，但成本高
- **GPT-4o-mini**: 性价比最高，推荐
- **Gemini-1.5-flash**: 速度快，效果不错
- **Claude-3-haiku**: 理解能力强，安全性高

### Q4: 情绪分析的准确率是多少？

这取决于多个因素：
- 数据质量
- Prompt 设计
- 市场情况

建议不要追求绝对准确，而是将情绪作为概率权重使用。

### Q5: 如何获取历史情绪数据用于回测？

可以考虑：
- **Santiment**: 提供历史社交指标
- **LunarCrush**: 提供历史 Galaxy Score
- **The TIE**: 机构级历史数据

---

## 相关资源

- [Freqtrade 官方文档](https://www.freqtrade.io/)
- [OpenAI API 文档](https://platform.openai.com/docs)
- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs) - 🆕 结构化输出
- [Claude API 文档](https://platform.claude.com/docs) - Anthropic 官方
- [Claude Structured Outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) - 🆕 JSON 输出
- [Google Gemini API](https://ai.google.dev/)
- [Gemini Structured Output](https://ai.google.dev/gemini-api/docs/structured-output) - 🆕 JSON Schema
- [Reddit API 文档](https://www.reddit.com/dev/api/)
- [CryptoPanic API](https://cryptopanic.com/developers/api/)

---

## 更新日志

- **2026-01-29 v1.3**: 根据 Anthropic Agent Skills 仓库添加模块化架构
  - 🆕 添加 **Agent Skills 架构** 章节（渐进式披露、模块化设计）
  - 🆕 创建 `crypto-sentiment-skill/` 技能目录结构
  - 🆕 创建 `SKILL.md` 技能定义文件
  - 🆕 创建 `prompts/sentiment_analysis.md` Prompt 模板
  - 🆕 添加加密货币俚语参考表（HODL, Diamond hands, WAGMI 等）
- **2026-01-29 v1.2**: 根据 Context7 查询的最新 LLM 官方文档更新
  - 🆕 添加 **现代 LLM 技能** 章节（Structured Outputs、Tool Use）
  - 🆕 OpenAI 分析器改用 **Structured Outputs API** (`json_schema`)
  - 🆕 添加 Claude (Sonnet 4.5) 和 Gemini (2.0 Flash) 最新代码示例
  - 🆕 添加新特性 vs 传统方式对比表
  - 🆕 更新相关资源链接（添加各提供商 Structured Outputs 文档）
- **2026-01-29 v1.1**: 根据 Context7 获取的 Freqtrade 官方文档更新
  - 添加 ADX 趋势强度指标 (官方推荐)
  - 使用 `enter_tag` / `exit_tag` 标签标记入场出场原因
  - 布林带变量名改为官方标准 (`bb_lowerband`, `bb_middleband`, `bb_upperband`)
  - 添加 `populate_indicators` 方法的官方说明注释
  - 确认 `INTERFACE_VERSION = 3` 为当前标准版本
- **2026-01-29 v1.0**: 初始版本发布

---

> 💡 **提示**: 情绪分析是量化交易的高级技术，建议先在模拟盘 (dry_run) 中测试，验证有效后再用于实盘。
>
> 📚 **Freqtrade 文档参考**:
> - [Strategy 101](https://www.freqtrade.io/en/develop/strategy-101) - 策略基础
> - [Strategy Customization](https://www.freqtrade.io/en/develop/strategy-customization) - 策略定制
> - [Hyperopt](https://www.freqtrade.io/en/develop/hyperopt) - 参数优化
>
> 🤖 **LLM 文档参考** (Context7 查询):
> - [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs) - 推荐
> - [Claude JSON Outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)
> - [Gemini Structured Output](https://ai.google.dev/gemini-api/docs/structured-output)
>
> 🧩 **Agent Skills 参考**:
> - [Anthropic Skills 仓库](https://github.com/anthropics/skills) - 官方示例
> - [Agent Skills 博客](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) - 设计理念
