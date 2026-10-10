---
title: TechSpec - 零花钱预算与分类消费契约
aliases: [零花钱预算, 个人财务预算设计, 动态分类提取]
created: 2026-10-08
updated: 2026-10-09
status: active
tags: [techspec, finance, budget, ponytail, tailwind, vant, org_id]
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
  `org_id` bigint(20) NOT NULL DEFAULT 20 COMMENT '家庭组/机构ID (预算隔离核心维度)',
  `belong_to` bigint(20) DEFAULT NULL COMMENT '属于(用户ID，预留家庭组下特定个人)',
  `budget_month` varchar(7) NOT NULL COMMENT '预算月份 (格式 YYYY-MM, 避开 MySQL 8 保留字 YEAR_MONTH)',
  `income_and_expenses` varchar(32) NOT NULL DEFAULT 'expense' COMMENT '收支类型(expense:支出, income:收入)',
  `budget_amount` decimal(12,2) NOT NULL DEFAULT '0.00' COMMENT '预算金额 (元)',
  `category_codes` text DEFAULT NULL COMMENT '纳入统计的业务分类列表 (逗号分隔，纯净类别如餐饮、水电，不与收支混淆)',
  `remark` varchar(255) DEFAULT NULL COMMENT '备注',
  `create_user` bigint(20) DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_user` bigint(20) DEFAULT NULL COMMENT '更新人',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `is_delete` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否删除 0否 1是',
  PRIMARY KEY (`id`),
  UNIQUE KEY `finance_budget_org_month_IDX` (`org_id`, `budget_month`, `is_delete`),
  KEY `finance_budget_belong_to_IDX` (`belong_to`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='财务月度零花钱预算配置表';
```

> [!NOTE] 家庭组组织架构归属与领域职责分离 (Ponytail 规约)
> - **组织架构维度隔离 (`org_id`)**：零花钱预算按家庭组/机构维度全局统一核算，默认通过当前登录用户所属 `org_id` 进行读写隔离与历史月份继承；唯一索引升级为 `(org_id, budget_month, is_delete)`。
> - **个人字段预留兼容 (`belong_to`)**：保留 `belong_to` 字段且置为可空，为将来家庭组下配置个人专属子预算留出扩展槽，无须后续二次变更表结构。
> - **流水表免物理加列 (Ponytail 零 DDL 架构)**：财务流水主表 `finance_info` 保持原有表结构，**免加 `org_id` 物理列**，避免大表 DDL 与历史流水清洗回填。
> - **数据权限全员共享 (`ORG_SHARED` 机构成员子查询)**：`FinanceInfoMapper` 统一配置 `@DataPermission(table = "finance_info", field = "belong_to", orgField = "", scope = DataPermissionScope.ORG_SHARED)`，自动降级为 `alex_user.t_org_user_info` 机构成员子查询匹配归属人 `belong_to`，实现同家庭组内全员实时共享流水与分类池，跨家庭组严格物理隔离。
> - **Mapper 规约 (mapper.xml 替代内存注解)**：月度查询 `selectByMonth` 与历史继承回溯 `selectLatestBefore` 统一迁入 `FinanceBudgetInfoMapper.xml` 维护，杜绝 Java 注解写死硬编码 SQL。
> - **方向与类别正交**：`income_and_expenses` 独立承载预算方向，支持单选（`expense` 为支出、`income` 为收入）以及多选（`expense,income` 支出+收入均选）。
> - **多选聚合计算**：当同时勾选支出与收入时，后端服务层自动解开收支方向限定，计算 `totalExpense - totalIncome (净支出: 支出 - 收入)`，使同时追踪收支流水的分类预算（如兼顾消费与退款）得以准确统计。
> - **老数据兼容与自愈**：后端服务层在读取与保存时，自动剔除 `categoryCodes` 中混杂的 `支出`、`收入`、`expense`、`income`，使老配置无缝平滑自愈。
> - **家庭组全员聚合流水 (Ponytail 极简模式)**：服务层核算实际开销时，`queryVo.belongTo` 保持为 `null`，复用现成 `@DataPermission` JSqlParser 插件自动汇总所属家庭组内所有家庭成员（如小袋子、臭屁宝）的收支流水。

---

## 3. 后端接口契约与极简查询实现

### 3.1 核心 API 清单

| 方法 | 路径 | 说明 | 参数 |
| --- | --- | --- | --- |
| `GET` | `/finance-budget/status` | 查询指定月份的零花钱预算与消费进度 | `budgetMonth` (YYYY-MM), `orgId` (可空), `belongTo` (可空) |
| `POST` | `/finance-budget/save` | 保存/调整指定月份零花钱预算 | `FinanceBudgetSaveReq` (`budgetMonth`, `orgId`, `budgetAmount`, ...) |
| `GET` | `/finance-budget/categories` | 动态提取近两月已有记账分类池 | `budgetMonth` (YYYY-MM), `belongTo` (可空) |

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

### 3.3 预算核算与使用率契约 (2026-10 负使用率与净结余演进)
1. **实际开销 (`actualExpense`)**：
   - 纯支出模式 (`expense`)：`actualExpense = totalExpense`
   - 纯收入模式 (`income`)：`actualExpense = totalIncome`
   - 双选模式 (`expense,income`)：`actualExpense = totalExpense - totalIncome`（净支出，当收入大于支出时为负数，代表净结余冲抵）
2. **剩余预算 (`remainingAmount`)**：
   - `remainingAmount = budgetAmount - actualExpense`
3. **预算使用率 (`usagePercent`)**：
   - 公式：`actualExpense / budgetAmount * 100`，保留 1 位小数（`RoundingMode.HALF_UP`）；
   - 上限未设置或非法（`budgetAmount <= 0`）：返回 `0.0%`；
   - **允许真实负值**：当 `actualExpense < 0`（如双选统计下收入大于支出），如实返回负使用率（例如 `-46.0%`、`-191.3%`），不强行截断为 `0.0%`；
   - **前端视觉守卫**：
     - PC 端 `<a-progress>` 进度条轨道宽度进行 `[0, 100]` 夹紧（`Math.max(0, Math.min(usagePercent, 100))`），文字 `:format` 如实展示负百分比；
     - 移动端 `<van-progress>` 同理使用 `Math.max(0, Math.min(100, usagePercent))` 夹紧视觉进度条，`:pivot-text` 如实展示负百分比。

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
2. **移动端极速记账全链路重构 (`financeManagerDetail/index.vue`, I-06 落地)**：
   - **收支分段选择 (Segmented Tabs)**：顶部突出「支出 / 收入」快速切换，带红绿状态指示圆点；
   - **大字号即时金额看板 (Hero Amount)**：醒目展示大额数字与光标指示，所选分类/时间/支付方式/备注以小胶囊置于副栏；
   - **九宫格分类网格 (Category Icon Grid)**：预设核心消费语义大图标（餐饮、交通、购物、日用、娱乐等），动态融合后端近两月已有分类（`getBudgetCategories`），轻触具有 `navigator.vibrate(10)` 触感震动与微动效，点选分类自动填充名称；
   - **内置定制记账数字键盘 (Built-in Keypad)**：屏幕底部常驻定制网格键盘（数字键、退格、日期快捷、再记一笔、完成），彻底避免系统软键盘反复弹起顶遮界面的卡顿抖动；
   - **连续记账连记支持**：提供「再记一笔」按钮，保存成功后震动并清空金额/名称，保留当前日期与支付环境，支持多笔支出无缝秒级录入；
   - **高级设置收折面板**：非高频字段（归属人、状态）折叠收纳，保证主流程极简。
3. **契约保障**：
   - 金额与 ID 保持严格类型安全，日期调用 `@/utils/dayjs` 格式化，接口严格采用对象解构。

### 4.3 PC 端财务信息页面交互升级 (2026-10 Task-Skill & Impact-Table 落地)
1. **极速记账优化 (`finance-manager-detail/index.vue`)**：
   - **弹窗视界优化**：宽度从臃肿的 1000px 收敛至紧凑的 680px，降低认知负荷与眼球跳动幅度；
   - **常用类别 Pill 胶囊**：动态获取近两月记账类别并渲染为快捷药丸，点击一键填入 `typeCode`，保留手工输入与下拉微调兜底；
   - **智能上下文预填**：自动装配默认支付方式（微信 `wx`）、当前时间（`dayjs()`）、收支类型（`expense`）、有效状态（`1`）与当前登录用户 ID，极简记账仅需填写「名称」与「金额」；
   - **高效连续记账**：底部新增「保存并再记一笔」按钮，成功后清空金额/名称/类别并保留环境上下文，无需反复开关弹窗。
2. **预算 ➔ 账单明细一键联动穿透 (`index.vue`)**：
   - 点击预算卡片中的计入类别胶囊（`cat-pill`）或当月已用金额（`clickable-sub-stat`），直接联动下方账单表格过滤出当月对应分类明细；
   - 列表上方自动展示「已联动过滤：分类/预算 (当前月份)」高亮横幅，并提供一键「恢复全量明细」撤销操作。
3. **周期快捷 Pill 胶囊置顶 (`finance-manager-filter/index.vue`)**：
   - 筛选栏置顶快捷选择胶囊：`[本月]` `[上月]` `[近30天]` `[全部]`，支持 1 击即时刷新日期范围并防抖/立即触发服务端多维统计与列表；
   - 外部联动（如穿透筛选）自动同步回激活态，非预设区间平滑降级为自定义态。
4. **表格明细视觉升级 (`index.vue`)**：
   - 账目类别列由纯文本升级为确定性哈希柔和彩胶囊（`category-pill`），直观清晰且支持点击快速筛选该分类；
   - 金额列维持等宽与红绿收支语义，确保高频对账视觉舒适度。
5. **概览卡片微洞察与日均/Top消费微标签 (`index.vue`)**：
   - **日均支出动态计算**：自适应筛选时段（当月已过天数/自选区间实际跨度/参考天数），计算展示 `¥xx.xx/天` 日均支出及依据说明；
   - **主要支出分类 Top 3 微标签**：自当前明细聚合提取支出排名前 3 的分类及其发生额占比（如 `餐饮 42%`、`交通 28%`）；
   - **微标签穿透联动**：Top 分类微标签轻触即可直接联动触发账目类别筛选（`filterByCategory`），并在表格上方呈现高亮联动横幅与一键清除，与预算穿透体验高度统一。
6. **零花钱预算弹窗深度体验与 Taste-Skill 美学升级 (`index.vue`)**：
   - **一体化快捷金额胶囊 (`QUICK_BUDGET_PRESETS = [1000, 2000, 3000, 5000]`)**：移除生硬虚线分割，快捷档位作为「无边框微胶囊」紧密融入金额输入区下方，支持一键快速填充；
   - **原生 `<a-segmented>` 统计范畴切换**：替代粗糙手写按键，集成原生平滑滑动微动效，消灭贴边与纵向挤压遮挡感；
   - **信息极简降噪 (Signal-to-Noise Ratio)**：移除冗余重复的浅蓝横幅，释放 50px 充裕呼吸空间；
   - **收支方向微质感 Tag**：收支类型采用标准微质感 Tag，带 `CheckOutlined` 勾选状态反馈；
   - **双轨分类池兜底 (`availableCategories`)**：自动合并系统 10 项核心消费分类与近两月账本流水，彻底解决新用户分类池为空的冷启动问题；
   - **玻璃态实时试算与健康度模拟 (`previewStats`)**：弹窗内输入金额即刻模拟当月已计、试算结余与预算使用率进度条，超出预算时即时醒目标红预警；
   - **心智因果归位与无割裂线容器化升级**：收支方向作为顶层全局前置，统计范畴紧随其后；自选分类紧贴切换器在白底独立轻质感容器（`bg-white rounded-xl shadow-2xs`）中展开，彻底消除卡片内多重横向割裂线；分类选项全面升级为现代全圆角微胶囊 Pills（未选中轻灰柔和、选中科技蓝高亮带触控微动效）。
7. **概览卡片首次加载骨架屏占位与平滑刷新 (2026-10 Ponytail 骨架规范)**：
   - **冷启动虚假零值根除**：账单收支统计与月度预算两栏分别包裹 Ant Design Vue 原生 `<a-skeleton :loading="!isFirstLoaded" active>`，彻底消灭首屏数据网络传输期间呈现刺眼 `¥0.00`、`0笔`、`0%` 的数据心智误导；
   - **零跳动防抖 (No-CLS)**：概览卡片容器设定 `min-height: 72px` 配合行高与微圆角，骨架屏条带与加载后内容高度严格对齐，杜绝高度弹跳；
   - **Stale-while-revalidate 平滑二次筛选**：骨架屏仅在初次进入时生效；后续用户点击快捷周期或切换查询条件时，旧数据平滑停留并由表格 loading / 进度条驱动，拒绝频繁灰白闪烁。
8. **预算卡片 Taste-Skill 结构对称美学与消灭遮挡 (2026-10 落地)**：
   - **双卡片同构化对称设计**：预算卡片升级为与左侧账单统计 1:1 对齐的 3 层结构（Header 标题栏、主金额行、副洞察行），消灭左高右低的落差，使中间垂直分割线两端视觉重量完美平衡；
   - **主金额与副指标层级解耦**：主金额行仅保留「剩余可用」与「上限/已计」，专注核算；将进度条与「计入分类」下沉至独立的 `budget-insights-row`（带浅灰微虚线与左侧日均支出行水平对齐）；
   - **负百分比零碰撞与 AntD 样式逃逸防御**：针对 Ant Design Vue `<a-progress>` 底层将 `.ant-progress-text` 硬编码为 `width: 2em` 且对 `.ant-progress-outer` 施加负 margin 导致长字符串（如 7 位 `-191.3%`）溢出逃逸碰撞右侧元素的缺陷，通过 scoped `:deep` 深度重置为 `display: flex; align-items: center;`、`outer { margin-inline-end: 0 !important; }` 与 `text { width: auto !important; }`，彻底从容器盒模型根源杜绝文本外溢碰撞；进度条容器固定 160px，右侧分类胶囊药丸自适应舒展，展示 3 个 + 溢出计数。
9. **预算进度条 Taste-Skill 财务风控六阶语义色板 (2026-10 落地)**：
   - **多端一致性色阶体系**：
     - `< 0%`：净结余充裕 / 收入大于支出 ➔ **翡翠绿**（PC `#52c41a` / 移动 `#07c160`）；
     - `0% ~ 75%`：常规安全区 ➔ **品牌蓝**（PC `#1677ff` / 移动 `#1989fa`）；
     - `75% ~ 90%`：适度关注区 ➔ **琥珀金**（PC `#faad14` / 移动 `#ff976a`）；
     - `90% ~ 100%`：临界高压区 ➔ **火山橙**（`#fa541c`）；
     - `100% ~ 150%`：超支破线区 ➔ **警示红**（PC `#ff4d4f` / 移动 `#ee0a24`）；
     - `≥ 150%`：严重爆表失控区 ➔ **深绛红**（`#cf1322`）。
   - **全链路感知协同**：概览卡片进度条、弹窗实时动态试算进度条与移动端卡片进度条完全统一该语义色板。

- **弹窗实时动态试算与健康度模拟**：设置/调整预算弹窗内，监听收支方向（支出/收入/双选）、统计范围模式（全量/自选）及具体选中分类（categoryCodes）的变动，采用 150ms 防抖动态联动调用服务端 `/finance-info/summary` 接口模拟计算当前月份符合该配置的「本月已计」与「试算结余」，支持负数净结余规范排版（-¥xxx）与加载状态提示。