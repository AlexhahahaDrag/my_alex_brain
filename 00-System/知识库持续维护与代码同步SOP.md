---
title: 知识库持续维护与代码同步 SOP
tags: [overview, standard, sop, sync, maintenance]
aliases: [知识库同步SOP, 维护规范]
created: 2026-08-30
updated: 2026-09-16
status: active
---

# 🔄 知识库持续维护与代码同步 SOP

返回导航：[[Home]]

---

## 1. 核心契约原则 (Core Contract)

> [!IMPORTANT]
> **代码与知识库双向同步原则**：
> 知识库 `my_alex_brain` 是项目全生命周期唯一可信的业务与技术真实之源（Source of Truth）。
> **凡涉及代码中的业务规则、数据模型、API 契约、页面交互逻辑或运维配置修改，必须在交付代码的同时，同步更新 `D:\project\my_alex_brain` 中对应的文档！**

---

## 2. 场景化同步映射指南

```mermaid
graph TD
    Change[代码与业务发生变更] --> CheckType{变更属于哪类?}

    CheckType -->|新增/调整业务功能| SyncPRD[1. 同步更新 02-Features/对应功能的 PRD 与 TechSpec]
    CheckType -->|各端通用架构/框架升级| SyncDev[2. 同步更新 01-Standards/各端开发规则]
    CheckType -->|解决线上重大Bug/踩坑| SyncTrouble[3. 沉淀至 02-Features/对应功能的 避坑与Bug修复SOP]
    CheckType -->|技术选型/重大架构决定| SyncADR[4. 使用 03-Templates/Template-ADR 沉淀架构决策]
```

### 2.1 变更场景与对应更新文件映射表

| 代码变更场景 | 对应模块/端 | 知识库必须同步更新的路径 |
| :--- | :--- | :--- |
| **新增/修改业务字段或表结构** (如 Gift 增加字段) | `alex_miaosha_finance` | • `02-Features/01-Gift-礼尚往来/PRD-*.md`<br>• `02-Features/01-Gift-礼尚往来/TechSpec-*.md` |
| **PC 端新增/调整页面与交互** | `alex_miaosha_front` | • `01-Standards/PC端开发规则与组件范式.md`<br>• `02-Features/对应功能/PRD-*.md` |
| **移动端交互/触觉/视口调整** | `alex_miaosha_mobile` | • `01-Standards/移动端开发规则与适配规约.md`<br>• `02-Features/对应功能/PRD-*.md` |
| **权限规则/数据隔离逻辑调整** | `alex_miaosha_user` | • `02-Features/02-RBAC-用户组织权限/TechSpec-*.md`<br>• `01-Standards/后端微服务开发规则与运维规约.md` |
| **新增微服务或端口变更** | 全局网关/运维 | • `00-System/全栈系统架构总览与交互流.md`<br>• `01-Standards/后端微服务开发规则与运维规约.md` |
| **修复复杂竞态/缓存/安全 Bug** | 全端通用 | • `02-Features/对应功能/避坑与Bug修复SOP/` 沉淀复盘 SOP |

---

## 3. AI Agent 与开发者工作流约束

每次 AI Assistant 或开发团队完成功能迭代时，必须执行 **闭环检查**：
1. **代码与单测实现**（满足功能与测试覆盖率）；
2. **知识库文档同步**（检查 `D:\project\my_alex_brain` 对应需求/开发文档是否同步更新，版本日期置为最新）。
