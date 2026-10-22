---
title: "外部参考 - [文档或技术主题名称]"
aliases: ["[文档简称]", "[外部技术主题]参考"]
source_url: "https://example.com/doc"
author: "[原作者/官方机构]"
clipped_date: YYYY-MM-DD
status: active
tags: [external-doc, reference, template]
---
# 🌐 外部参考 - [文档或技术主题名称]

返回首页：[[Home|知识库首页]]  
关联系统功能：[[02-Features/00-FeatureName/TechSpec-FeatureName|内部技术方案]] | [[01-Standards/后端微服务开发规则与运维规约|开发规约]]

---

## 1. 核心摘要与引入背景

> [!NOTE]
> **引入背景**：简要说明为何引入该外部文档（如：对接微信服务号模板消息推送、学习 SonarQube AES-GCM 安全规则、引入 Midscene AI 自动化测试等）。  
> **核心要点**：提炼外部文档最关键的 2~3 个技术结论或规约。

---

## 2. 核心技术正文 (纯净提取)

> [!TIP]
> 经过清洗剔除广告与边栏后的纯净正文，包含代码块、接口定义与核心参数表。

### 2.1 关键接口/协议定义

| 参数名 | 类型 | 必填 | 说明 |
| :--- | :--- | :---: | :--- |
| `example_param` | String | 是 | 核心参数示例说明 |

### 2.2 核心代码范式

```typescript
// 示例纯净代码
export interface ExamplePayload {
  key: string;
  value: number;
}
```

---

## 3. 本地化适配与避坑指南 (Local Adaptation)

> [!WARNING]
> **原文档潜在缺陷/本地项目特殊约束**：
> 1. 外部文档可能采用 `LocalDate.now()` 或 JS `Number` 传 ID，本地系统必须转换为 `DateUtils.now()` 与 `Long2String`；
> 2. 外部 API 异常状态码与本地系统的 Result 结构映射关系。
