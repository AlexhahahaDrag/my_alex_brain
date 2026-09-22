---
title: PRD - 礼尚往来与人情记账功能说明
tags: [prd, feature, gift, finance, mobile, pc]
aliases: [礼尚往来PRD, 礼尚往来功能说明]
created: 2026-09-16
updated: 2026-09-16
status: active
---

# 🎁 PRD - 礼尚往来与人情记账功能说明

返回导航：[[Home]] | 技术实现：[[02-Features/01-Gift-礼尚往来/TechSpec-礼尚往来业务领域模型与契约|TechSpec-礼尚往来业务领域模型与契约]]

---

## 1. 业务愿景与核心痛点
人情往来（随礼、收礼、还礼）是中国传统社交场景的重要组成。传统纸质记账或微信零散记账存在**“时间一长遗忘还礼金额”、“宴席现场收礼统计繁重混乱”、“查账困难”**等痛点。
本模块提供 PC 管理端与移动端双向协同的礼尚往来数字化解决方案，支撑 10w+ 往来记录的高性能检索与对账。

---

## 2. 核心用例与状态流转

```mermaid
graph TD
    User((亲友/用户)) --> Record[随礼 / 收礼]
    Record --> TypeCheck{是收礼还是随礼?}
    TypeCheck -->|RECEIVE 收礼| PendingReturn[待还礼状态 (计算人情负债)]
    TypeCheck -->|GIVE 随礼| PendingReceive[待收礼状态]
    PendingReturn -->|后续还礼| ReturnAction[RETURN 还礼动作]
    ReturnAction --> Balance[双向冲抵抹平]
```

---

## 3. 三端页面功能清单

### 3.1 移动端（单手极简记账与对账）
- **3秒极简记账**：单手键盘快速切换“随礼(GIVE)”或“收礼(RECEIVE)”，输入金额并智能联想亲友姓名，点击完成触发原生触觉轻微振动反馈；
- **亲友录往来卡片**：展示每位亲友的历史时间轴、收送差额（“他欠我”或“我欠他”）；
- **一键智能还礼**：在亲友时间轴点击待还礼流水，系统自动弹起预填好的还礼记账键盘，关联原流水完成追溯；
- **喜宴礼簿现场助手**：支持婚礼/寿宴等大事件现场快速连环录入。

### 3.2 PC 管理端（后台大盘、对账治理与 Tailwind 现代美学）
- **现代美学全栈升级 (Tailwind Modern UI)**：
  - 画布全面重塑为 Tailwind `bg-slate-50`（`#f8fafc`），容器统一对齐 `rounded-2xl`（`16px`）微圆角、`border border-slate-200` 细微边框与 `shadow-sm`，淘汰老旧生硬的 `7px` 边框与暗沉阴影；
  - 核心 KPI 指标卡升级为双环药丸轻底徽章（Emerald/Rose/Blue/Gold），支持平滑微浮动交互（`hover:-translate-y-0.5 hover:shadow-md`）；
- **总览大盘与人情台账 (`gift-dashboard`, `record`)**：
  - 4 大收支差额指标动态统计，胶囊化快速过滤标签，大额收支醒目对比；
- **亲友档案与人情事件 (`person`, `event`)**：
  - 亲友关系网络画像卡片流，核心/重要/普通/弱关系分层徽标，正负人情净值胶囊；
  - 人情大事件办宴看板、时间胶囊、高频场景推荐卡片与事由词典分组面板；
- **统计报表与 AI 智能解读 (`analysis`, `ai`)**：
  - 月度/年度趋势柱状图、收礼/随礼亲友排行榜、关系构成分布；
  - AI 智能解读面板（`GiftAiInsightPanel`），采用极光微白卡片与流式分析气泡，提供往来健康度建议；
- **Excel 异步导出**：支持按筛选结果导出标准化对账报表。

---

## 4. 字段规则与七点法边界

| 字段名 | 业务含义 | 类型 | 必填 | 校验规则 | 默认值 | 交互说明 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `person_id` | 亲友 ID | String (Long2String) | 是 | 必须有效存在 | - | 下拉联想选择 |
| `amount` | 礼金金额 | Decimal(12,2) | 是 | $> 0$ 且 $\le 10,000,000$ | - | 货币格式化输入 |
| `direction` | 礼金流向 | String | 是 | 枚举: `GIVE`/`RECEIVE`/`RETURN` | `GIVE` | 单选标签切换 |
| `related_record_id` | 关联原礼金ID | String (Long2String) | 否 | direction=RETURN 时校验有效性 | `null` | 还礼时自动反填关联 |
| `pay_time` | 交易时间 | Datetime | 是 | 格式 `YYYY-MM-DD HH:mm:ss` | 当前时间 | 时间选择器 |

---

## 5. 验收标准
- [ ] **[AC1] 还礼关联正确性**：还礼记录保存后，原收礼记录的已还状态必须更新，待还金额精确扣减；
- [ ] **[AC2] ID 精度防篡改**：19 位雪花算法 ID 传输至前端必须保持 string，严禁出现低位变 `00` 错误；
- [ ] **[AC3] 行级数据权限隔离**：跨机构或跨用户禁止通过 URL 传参读取他人礼金明细。
