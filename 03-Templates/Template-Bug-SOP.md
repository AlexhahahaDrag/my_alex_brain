---
title: "避坑 SOP - [故障/缺陷名称]"
tags: [sop, troubleshooting, bug, defense]
aliases: ["[缺陷名称]SOP", "[缺陷名称]排错指南"]
created: YYYY-MM-DD
updated: YYYY-MM-DD
status: active
---

# 🚨 避坑 SOP - [故障/缺陷名称]

所属功能模块：[[02-Features/XX-FeatureName/PRD-FeatureName|对应PRD]]

---

## 1. 现象与危害 (Symptom & Impact)
- 错误提示、HTTP 状态码、日志堆栈异常、用户直接感知损失。

---

## 2. 根本原因剖析 (Root Cause)
- 为何产生？代码漏洞、并发竞态、引用污染或事务失效？

---

## 3. 防护代码与规范 (Solution & Defense Code)
- 正确的代码实现示例；
- 防护规则与持续监测手段（如单测/拦截器/Midscene断言）。
