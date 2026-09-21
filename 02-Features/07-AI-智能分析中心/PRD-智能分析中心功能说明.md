---
title: PRD - 智能分析中心功能说明
tags: [prd, ai, deepseek, llm, rule-engine, analysis, fullstack]
aliases: [AI智能分析中心, AI微服务需求, AI分析PRD]
created: 2026-09-16
updated: 2026-09-16
status: active
---

# 🤖 PRD - 智能分析中心功能说明 (`alex_miaosha_ai`)

返回首页：[[Home|知识库首页]] | 架构总览：[[00-System/全栈系统架构总览与交互流|系统架构总览]]

---

## 1. 业务全景与服务定位 (WHAT & WHY)

`alex_miaosha_ai`（端口 `30010`，Nacos 注册名 `alex-ai-dev`）是全栈系统的通用 AI 智能化中枢。它基于大语言模型（LLM）与规则引擎双模驱动，为财务记账、人情往来、营销策划与电商运营提供轻量、低延迟、高可用的智能分析与决策建议能力。

```mermaid
graph TD
    Client[业务调用方: 财务/秒杀/营销/用户中心] -->|OpenFeign: AiAnalyzeApi| Gateway[API 网关 / 内部服务调用]
    Gateway --> AISvc[AI分析微服务: alex_miaosha_ai (Port: 30010)]
    
    subgraph 引擎路由与容灾
        AISvc --> Router{AiEngineRouter 动态路由}
        Router -->|优先选用 (已配置 API-Key)| DeepSeek[DeepSeek 大模型引擎 (deepseek-chat / reasoner)]
        Router -->|平滑降级 (未配 Key 或网络异常)| RuleEngine[RuleBased 规则引擎兜底]
    end
    
    DeepSeek -->|流式/结构化响应| Formatter[AiAnalyzeResp 统一响应封装]
    RuleEngine -->|本地即时生成| Formatter
    Formatter -->> Client
```

---

## 2. 端侧功能清单与应用场景

### 2.1 核心智能化赋能场景
1. **财务与礼金收支智能诊断**：
   - 结合用户月度/年度收支数据，自动输出财务健康度摘要（如“餐饮支出占比 42%，超出健康阈值”）；
   - 针对礼尚往来人情账本，智能识别“未还礼亲友”并生成还礼礼金预估建议。
2. **营销与优惠券策略辅助**：
   - 根据用户历史客单价与核销偏好，分析券模板投放效果，生成营销优化文案。
3. **订单履约与售后舆情分析**：
   - 提取订单退款原因与用户评价中的关键负向词汇，提供售后处置建议。

### 2.2 交互与调用规格 (`POST /api/v1/ai/analyze`)
- **分析深度控制 (`depth: 1~3`)**：
  - `depth=1`：基础摘要与提炼（默认快速响应）；
  - `depth=2`：归纳核心关键点与成因；
  - `depth=3`：深度归因与前瞻性建议。
- **动态模型覆盖**：调用方可在请求体中灵活指定 `engine`（`deepseek` / `rule-based`）与 `model`（如 `deepseek-chat` 或深度思考模型 `deepseek-reasoner`）。

---

## 3. 验收标准 (Acceptance Criteria)

- **[AC1] 离线与无 Key 平滑保活**：在未配置外部 `AI_DEEPSEEK_API_KEY` 或断网环境下，接口必须自动降级到 `RuleBasedAiEngine`，返回状态码 200 与结构化兜底摘要，严禁抛出 500 导致业务上游熔断。
- **[AC2] 超时与熔断边界**：大模型远程调用严格限制在 15 秒超时（`timeout-ms: 15000`），超时后自动触发 `AiAnalyzeFallbackFactory` 熔断降级。
- **[AC3] 统一链路追踪**：每次分析响应必须包含唯一 `requestId`、使用的实际引擎标签 `engine` 与精确调用耗时 `costMs`。
