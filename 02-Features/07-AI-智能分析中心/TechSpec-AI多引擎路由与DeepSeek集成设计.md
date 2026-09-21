---
title: TechSpec - AI多引擎路由与DeepSeek集成设计
tags: [techspec, ai, deepseek, feign, rule-engine, architecture]
aliases: [AI技术方案, DeepSeek集成方案, AI架构设计]
created: 2026-09-16
updated: 2026-09-16
status: active
---

# 🛠️ TechSpec - AI多引擎路由与DeepSeek集成设计

返回功能说明：[[02-Features/07-AI-智能分析中心/PRD-智能分析中心功能说明|PRD-智能分析中心功能说明]]

---

## 1. 架构总览与时序流转

`alex_miaosha_ai` 服务由公共契约模块 `ai_api` 与核心运行时 `ai_boot` 构成，采用面向接口的多引擎策略模式：

```mermaid
sequenceDiagram
    autonumber
    actor Caller as 业务微服务 (如 finance_boot)
    participant Feign as AiAnalyzeApi (OpenFeign)
    participant Ctrl as AiAnalyzeController (Port: 30010)
    participant Router as AiEngineRouter
    participant DeepSeek as DeepSeekAiEngine
    participant Rule as RuleBasedAiEngine
    participant Fallback as AiAnalyzeFallbackFactory

    Caller->>Feign: analyze(AiAnalyzeReq)
    Feign->>Ctrl: POST /api/v1/ai/analyze
    Ctrl->>Router: route(engine, req)
    alt 外部配置已启用 DeepSeek 且 API Key 有效
        Router->>DeepSeek: analyze(req)
        DeepSeek->>DeepSeek: HTTP POST https://api.deepseek.com/v1/chat/completions
        DeepSeek-->>Ctrl: 返回模型生成分析结果
    else 降级或未配置 Key
        Router->>Rule: analyze(req)
        Rule-->>Ctrl: 返回基于规则的结构化摘要
    end
    Ctrl-->>Feign: 200 Result.success(AiAnalyzeResp)
    Feign-->>Caller: 业务获得分析结果
    opt 网络故障或 15s 超时
        Feign-xCtrl: 远程连接异常
        Feign->>Fallback: 触发熔断降级
        Fallback-->>Caller: 返回 Fallback 兜底响应
    end
```

---

## 2. 核心类结构与引擎抽象

```mermaid
classDiagram
    class AiEngine {
        <<interface>>
        +name() String
        +analyze(AiAnalyzeReq req) AiAnalyzeResp
    }

    class DeepSeekAiEngine {
        -DeepSeekClient client
        -DeepSeekProperties properties
        +analyze()
    }

    class RuleBasedAiEngine {
        +analyze()
    }

    class AiEngineRouter {
        -Map~String, AiEngine~ engines
        -AiProperties aiProperties
        +route(String requestedEngine) AiEngine
    }

    AiEngine <|.. DeepSeekAiEngine
    AiEngine <|.. RuleBasedAiEngine
    AiEngineRouter --> AiEngine
```

---

## 3. 契约定义与数据模型

### 3.1 请求对象 `AiAnalyzeReq`
```java
public class AiAnalyzeReq implements Serializable {
    private String bizType;             // 业务标识 (finance/coupon/order)
    private String content;             // 待分析正文内容 (必填)
    private Map<String, Object> context;// 结构化上下文参数
    private Integer depth;              // 分析深度 (1~3，默认1)
    private String engine;              // 强制指定引擎 (deepseek / rule-based)
    private String model;               // 指定模型 (deepseek-chat / deepseek-reasoner)
    private Double temperature;         // 采样温度 (默认0.2)
    private Integer maxTokens;          // 最大Token数 (默认1024)
}
```

### 3.2 响应对象 `AiAnalyzeResp`
```java
public class AiAnalyzeResp implements Serializable {
    private String requestId;           // 链路追踪ID
    private String summary;             // 智能分析摘要
    private List<String> keyPoints;     // 关键核心要点列表
    private String engine;              // 实际执行引擎标识
    private Long costMs;                // 执行耗时(毫秒)
}
```

---

## 4. 生产配置与环境变量规约

```yaml
# application-prod.yaml
server:
  port: 30010

spring:
  application:
    name: alex-ai-prod

ai:
  engine: deepseek
  deepseek:
    base-url: https://api.deepseek.com
    chat-completions-path: /v1/chat/completions
    api-key: ${AI_DEEPSEEK_API_KEY:} # 严格通过系统环境变量注入，严禁硬编码提交仓库
    model: deepseek-chat
    temperature: 0.2
    max-tokens: 1024
    timeout-ms: 15000 # 15s 严格超时
```
