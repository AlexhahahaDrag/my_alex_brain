---
title: TechSpec - 零花钱预算与分类消费契约
aliases: [零花钱预算, 个人财务预算设计, 动态分类提取]
created: 2026-10-08
updated: 2026-10-08
status: active
tags: [techspec, finance, budget, ponytail, tailwind, vant]
---
# 🛠️ TechSpec - 零花钱预算与分类消费契约

返回功能说明：[[02-Features/05-SelfFinance-个人资产财务/PRD-个人资产与收支记账功能说明|PRD-个人资产与收支记账功能说明]]

---

## 1. 业务背景与架构动机

在日常财务收支管理中，用户需要设定每月特定分类（如餐饮、零食、娱乐、服饰等个性化分类）的消费额度上限（即“零花钱月度预算”）。系统需满足以下工程目标：
1. **0 定时任务极简继承 (Ponytail 原则)**：坚决不引入每月1号跑批的定时任务。当月未设置预算时，查询读链路自动向上回溯最近一个已配置月份并标记 `isInherited = true`；当月编辑保存后仅持久化当前月记录。
2. **已有类别动态提取**：取消静态写死的预设分类，由后端自近两月（上月 1 号至本月末）实际记账明细中动态提取去重的 `type_code`（排除转账）。
3. **多端设计语言统一**：
   - PC 端：一体化紧凑卡片（收支概览与预算合并至一行 ~68px）+ Tailwind 风格卡片与 Chip 交互弹窗。
   - 移动端：微概览卡片 + Vant 4 触控胶囊药丸（Chip）与轻微触感震动（Haptics）。

---

## 2. 数据库设计与持久化模型

```sql
CREATE TABLE `finance_budget_info` (
  `id` bigint(20) NOT NULL COMMENT '主键ID',
  `belong_to` varchar(64) NOT NULL COMMENT '归属人ID (关联用户)',
  `year_month` varchar(7) NOT NULL COMMENT '预算月份 (格式 YYYY-MM)',
  `budget_amount` decimal(12,2) NOT NULL DEFAULT '0.00' COMMENT '预算金额 (元)',
  `category_codes` text DEFAULT NULL COMMENT '纳入统计的分类列表 (逗号分隔)',
  `remark` varchar(255) DEFAULT NULL COMMENT '备注',
  `create_user` bigint(20) DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_user` bigint(20) DEFAULT NULL COMMENT '更新人',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `is_delete` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否删除 0否 1是',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_belong_month` (`belong_to`, `year_month`, `is_delete`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='个人财务月度零花钱预算配置表';
```

---

## 3. 后端接口契约与极简查询实现

### 3.1 核心 API 清单

| 方法 | 路径 | 说明 | 参数 |
| --- | --- | --- | --- |
| `GET` | `/finance-budget/status` | 查询指定月份的零花钱预算与消费进度 | `yearMonth` (YYYY-MM), `belongTo` (可空) |
| `POST` | `/finance-budget/save` | 保存/调整指定月份零花钱预算 | `FinanceBudgetSaveReq` |
| `GET` | `/finance-budget/categories` | 动态提取近两月已有记账分类池 | `yearMonth` (YYYY-MM), `belongTo` (可空) |

### 3.2 动态分类提取逻辑 (`selectRecentCategories`)

```xml
<select id="selectRecentCategories" resultType="java.lang.String">
    SELECT DISTINCT type_code
    FROM finance_info
    WHERE is_delete = 0
      AND is_valid = '1'
      AND type_code IS NOT NULL
      AND type_code != ''
      AND type_code != '转账'
      <if test="belongTo != null and belongTo != ''">
          AND belong_to = #{belongTo}
      </if>
      <if test="startDate != null">
          AND order_date &gt;= #{startDate}
      </if>
      <if test="endDate != null">
          AND order_date &lt;= #{endDate}
      </if>
    ORDER BY type_code ASC
</select>
```
时间窗口采用：`startDate = 上月1号 00:00:00`，`endDate = 本月最后一天 23:59:59`。

---

## 4. 前端组件范式与多端落地

### 4.1 PC 端 (`alex_miaosha_front`)
1. **一体化概览卡片 (`.finance-overview-card`)**：
   - 将原「全量账单统计」与「零花钱月度预算」合并在同一横幅中；
   - 纵向占用从双行 ~100px 压缩至 ~68px 单行，高度释放超 32px，最大化表格有效视野；
   - 响应式弹性自适应，宽屏平铺，窄屏自然换行。
2. **Tailwind 风格配置弹窗**：
   - 额度卡片（`bg-slate-50/80`）+ 分类配置卡片（`rounded-xl border border-slate-200`）；
   - 大类「支出、收入」与动态提取的类别全转为 Tailwind Chip；
   - 支持一键「全选 / 清空」并展示实时计数徽标。

### 4.2 移动端 (`alex_miaosha_mobile`)
1. **触控药丸 Chip**：
   - 采用圆角 Pill 胶囊形状替换冗长 Checkbox 列表；
   - 选中高亮浅蓝背景（`#e6f4ff`）与主题色边框（`#91caff` / `#1677ff`）；
   - 轻触带 `transform: scale(0.96)` 微缩放与 `navigator.vibrate(10)` 触感震动反馈；
   - 超长分类支持内部独立滚动。
2. **契约保障**：
   - 金额与 ID 保持严格类型安全，日期调用 `@/utils/dayjs` 格式化。
