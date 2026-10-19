---
title: PRD - 系统消息与多业务到期提醒中心功能说明
aliases: [消息中心PRD, 到期提醒需求]
created: 2026-10-19
updated: 2026-10-19
status: active
tags: [prd, message, reminder, finance, gift, coupon, fullstack]
---
# 📋 PRD - 系统消息与多业务到期提醒中心功能说明

返回首页：[[Home|知识库首页]] | 技术方案：[[02-Features/09-Message-消息与到期提醒中心/TechSpec-系统消息与多业务幂等巡检架构|TechSpec-系统消息与多业务幂等巡检架构]]

---

## 1. 业务背景与产品定位

在个人与家庭财务及人情管理场景中，用户常面临三大痛点：
1. **人情重要事件遗忘**：亲友婚礼、满月、乔迁等礼尚往来事件（`gift_event_info_t`）缺乏提前一天与当天的集中提醒；
2. **消费券过期作废**：已领取但未核销完毕的消费券（`cpn_coupon_info_t`）过期才被发现；
3. **猫超卡与预付卡余额沉睡过期**：天猫超市享淘卡/账户记录（`account_record_info`）与商超预付消费卡（`prepaid_card_info_t`）临期缺乏醒目预警。

为此，系统构建统一的 **「系统消息与多业务到期提醒中心」**，打通移动端底部「消息」Tab、PC 端顶部铃铛通知抽屉与微信公众号模板消息，采用 **100% XXL-Job 后台定时任务全量巡检（`sysMessageRemindHandler`）+ 唯一幂等键（`dedup_key`）落库 + 微信模板消息联动推送 + 业务页面一键直达** 架构。

---

## 2. 核心功能与提醒规则

### 2.1 四大消息分类与触发阈值

| 消息大类 (`msgType`) | 业务来源 | 触发条件与子阶段 (`subType`) | 优先级 (`priority`) | PC 端直达路由 | 移动端直达路由 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`SYSTEM`**<br/>系统通知 | `sys_message_info_t` | • 首次访问自动生成欢迎通知 (`WELCOME`)<br/>• 用户/管理员手动新增备忘或系统公告 (`ANNOUNCE`) | `NORMAL` / `MEDIUM` / `HIGH` | 留在本抽屉或自定义 `jumpUrl` | 留在本页或自定义 `jumpUrl` |
| **`GIFT_EVENT`**<br/>礼尚往来事件 | `gift_event_info_t` | 扫描当前用户/机构可见且未删除的事件：<br/>• **当天举行 (`D_DAY`)**：`eventDate == today`<br/>• **提前1天 (`D_MINUS_1`)**：`eventDate == today + 1` | • 当天：`HIGH`<br/>• 提前1天：`MEDIUM` | `/finance/gift/event` | `/finance/gift/event` |
| **`COUPON_EXPIRE`**<br/>消费券到期 | `cpn_coupon_info_t` | 扫描未删除、未过期且剩余未核销数量 `remainingQuantity > 0`：<br/>• **当天到期 (`D_DAY`)**：剩余 `0` 天<br/>• **提前1天 (`D_MINUS_1`)**：剩余 `1` 天<br/>• **提前2~3天 (`D_MINUS_3`)**：剩余 `2~3` 天 | • 当天：`HIGH`<br/>• 提前1天：`MEDIUM`<br/>• 提前2~3天：`NORMAL` | `/cpn-coupon/cpn-coupon-info` | `/cpn-coupon/cpn-coupon-info` |
| **`CARD_EXPIRE`**<br/>猫超卡/预付卡到期 | `account_record_info`<br/>`prepaid_card_info_t` | 1. **猫超卡/账户记录**：`avli_date` 剩余 `0~3` 天且 `amount > 0`<br/>2. **预付消费卡**：`expire_date` 剩余 `0~3` 天且 `current_balance > 0` | • 当天：`HIGH`<br/>• 提前1天：`MEDIUM`<br/>• 提前2~3天：`NORMAL` | • 猫超卡：`/selfFinance/accountRecordInfo`<br/>• 预付卡：`/selfFinance/prepaidCardInfoT` | • 猫超卡：`/finance/accountRecordInfo`<br/>• 预付卡：`/selfFinance/prepaidCardInfoT` |

### 2.2 移动端交互体验 (`alex_miaosha_mobile`)
- **底部 TabBar 未读角标**：首页与消息页加载时自动同步未读总数，展示在底部导航栏「消息」图标右上角（超过 99 显示 `99+`，0 条时自动隐藏）。
- **分类胶囊与快捷筛选**：支持 `[全部 / 礼金事件 / 消费券 / 猫超/预付卡 / 系统通知]` 横向胶囊切换（附各类别未读数微标）、关键词搜索及「仅未读」一键过滤。
- **左滑手势与业务直达**：卡片支持左滑「标为已读 / 删除」，点击卡片自动消除未读红点并跳转至对应业务管理页面。
- **新增自定义提醒**：右上角「新增提醒」支持随时录入个人备忘或系统通知（已移除手动巡检按钮，提醒由后台 XXL-Job 100% 自动生成）。

### 2.3 PC 端交互体验 (`alex_miaosha_front`)
- **顶栏铃铛未读角标**：`my-right-info` 顶栏铃铛实时展示未读总数 Badge。
- **通知中心抽屉 (`MessageCenterDrawer.vue`)**：点击铃铛滑出 460px 宽通知抽屉，支持分类 Tab 切换、「全部已读」、「新增提醒」及单条「去处理」路由直达（已移除手动「实时巡检」按钮）。

---

## 3. 验收标准 (Acceptance Criteria)

- **[AC1] 提醒准确性与微信联动**：当天/次日举行的礼尚往来事件、剩余 `0~3` 天且未核销完的消费券、剩余 `0~3` 天且余额大于 0 的猫超卡与预付卡均能由 XXL-Job 准确生成对应阶段站内信，并自动触发微信公众号模板消息推送。
- **[AC2] 严格幂等与防复活**：同一业务记录在同一提醒阶段同一天仅生成一条消息、仅推送一次微信模板消息；用户标为已读或主动删除后，再次巡检绝不重复生成。
- **[AC3] 权限隔离与异常降级**：用户仅可查看、标为已读或删除归属本人或本机构的提醒消息，跨用户操作返回 `403` 拒绝；微信推送失败不影响站内信正常落库。
