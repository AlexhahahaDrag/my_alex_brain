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
- **移动端零花钱卡片体验升级 (Taste-Skill & Impact-Cable)**：
  - 排版与金额：负金额规范排版（-¥6,120.40），全面接入逗号千分位分割，净结余状态下采用健康墨绿色阶（#07c160）。
  - 空间减负与折叠收敛：分类标签默认收起只展示前4项加灵动展开胶囊（+N项 ∨），杜绝几十个标签撑爆首屏卡片高度，并自动过滤异常字符。
  - 决策抓手与情绪反馈：卡片头部新增动态情绪徽标（资金充裕 / 控制良好 / 接近限额 / 超支透支），进度条增加圆角衬底槽轨。
  - 链路闭环：点击已计金额（drillDownBudgetMonth）自动联动筛选当月账单并提供触觉震动反馈，打通追溯明细链路。

10. **移动端弹窗与 PC 端视觉及试算深度对齐 (2026-10-11)**：
   - **收支类型语义色阶**：移动端预算调整弹窗对齐 PC 端颜色语义，支出(消费)采用浅粉红底 #fff1f2、玫瑰红字与边框 #fda4af/#e11d48；收入采用浅绿底 #ecfdf5、翡翠绿字与边框 #6ee7b7/#059669；勾选图标同步跟随高亮色。
   - **实时试算与健康度模拟卡片**：移动端同步落地 3 列试算卡片（设定预算、本月已计、试算结余）与健康度进度条（<van-progress>），结合 150ms 防抖调用 /finance-info/summary 实时反馈收支方向与分类选择带来的月度预算变动。

- **移动端零花钱卡片渐进式折叠架构 (Progressive Disclosure)**：
  - 默认收起态：常驻高度仅约 40px，左侧展示钱袋图标与本月零花钱 + 状态徽标，右侧展示「剩余 ¥9,320.40」与「0%」微胶囊，为核心账单流水释放 140px+ 黄金视口。
  - 点击展开态：支持平滑内联展开，呈现月度预算、已计金额（支持一键下钻）、完整进度条与统计分类及调整预算入口，并提供微震动触觉反馈。
  - 不吸顶自然滚动：零花钱卡片内嵌于 CommonPullRefresh 滚动视图内作为内容首节点，当用户上滑浏览明细时随列表自然滑出视口，完全不占用吸顶空间，将全屏 100% 留给账单明细流水。

11. **移动端零花钱卡片 Taste-Skill 与 Impact-Cable 深度演进 (2026-10-11)**：
   - **极光微拟物底座 (Icon Squircle)**：28×28px 浅蓝极光圆角微底座（linear-gradient(135deg, #eff6ff 0%, #dbeafe 100%)）包裹 alance-o 资产图标，确立卡片左侧视觉重心。
   - **呼吸感状态微圆点徽标**：状态胶囊内嵌状态微圆点（● 控制良好、● 资金充裕、● 超支透支），搭配透明微边框与呼吸底色。
   - **金融等宽微排版**：拆分货币符号 ¥（10.5px）与金额主体（14.5px tabular-nums），使用率百分比改用中性柔和胶囊，解决绿/蓝刺眼色彩冲突。
   - **常驻底边隐式微环境进度条 (Ambient Hairline Progress)**：折叠态下卡片底部边缘内置 2.5px 隐式平滑水位进度槽，无需展开即可扫视当月预算水位。
   - **交互穿透与触觉闭环**：展开/折叠箭头带 22px 微圆轻触按钮与 180° 平滑旋转，附带 20ms 触觉微震动；展开态三列指标卡片岛屿化（mount-card），「本月已计」挂载「明细›」一键联动穿透当月明细列表；「预算设置」强化操作路径。

12. **接口契约修正与 400 格式错误根因排查 (2026-10-11)**：
   - **LocalDate 日期反序列化规约**：后端 `com.alex.api.finance.vo.finance.FinanceInfoVo` 中 `infoDateStart` 与 `infoDateEnd` 声明为 `LocalDate`。前端动态试算调用 `/api/v1/finance-info/summary` 时，必须使用纯日期格式 `.format('YYYY-MM-DD')`，严禁拼接时分秒（如 `YYYY-MM-DD 00:00:00`），避免 Jackson 反序列化抛出 `DateTimeParseException` 触发全局 400 异常。
   - **多分类过滤字段契约对齐 (`typeCodes: string[]`)**：多分类过滤应透传 `typeCodes: string[]` 字段，后端 XML 原生映射为 `finance_info.type_code in (...)`；修正前端原先向单值 `typeCode` 拼接逗号字符串的错误。
   - **多端一致性覆盖**：PC 端（`alex_miaosha_front`）与移动端（`alex_miaosha_mobile`）的 `FinanceManagerData` 契约定义与 `fetchPreviewSimulation` 均已完成同步修正并通过构建打包验证。

13. **移动端零花钱预算二级独立动态路由与细粒度权限矩阵 (2026-10-12 演进)**：
   - **独立动态路由拆分 (方案 2)**：
     - 将原本 180 行内嵌于 `financeManager/index.vue` 的弹窗彻底解耦为全新二级动态路由页面；
     - 菜单元数据：`menu_code: 'financeBudgetManage'`, `path: '/finance/financeManager/budgetManage'`, `component: '/src/views/finance/financeManager/budgetManage/index.vue'`, `hide_in_menu: '1'` (二级隐藏路由，不污染底部导航 TabBar)，准入权限码挂载为 `finance:financeManager:budget:view`；
     - 路由生命周期：主列表页通过 `onActivated(() => loadBudgetStatus())` 监听返回事件，实现从独立预算页面返回时大盘卡片水位自动同步无缝刷新。
   - **细粒度权限矩阵**：
     - `finance:financeManager:query`：零花钱流水查询；
     - `finance:financeManager:add`：记一笔流水新增（导航栏右上角加号按钮受此权限控制）；
     - `finance:financeManager:edit`：修改流水；
     - `finance:financeManager:delete`：删除流水；
     - `finance:financeManager:budget:view`：预算大盘与动态试算查看；
     - `finance:financeManager:budget:edit`：预算配置调整与保存。
   - **角色与属主权限保护机制 (Owner-Scoped Data Protection)**：
     - **家庭成员 (`user` / `family_member`)**：拥有流水增删改查权限，但受**数据属主严格保护**——仅能删除和修改个人创建/归属的流水（`String(item.belongTo) === String(myId)`）；对他人流水详情呈现只读提示条并隐藏/置灰提交按钮，卡片侧滑删除按钮自动隐藏；拥有预算查看权限（`budget:view`），自动进入只读浏览与试算模式，禁用输入与修改并隐藏保存按钮。
     - **家庭管理员 (`family_admin` / `admin`) 与超管 (`super_super`)**：拥有家庭组内全量管理权限，可删除和修改任意成员的记账流水，并拥有预算编辑调整保存权 (`budget:edit`)。



14. **AI 智能测算推荐与口语化指令一键配置演进 (2026-10-12 全套一体化)**：
   - **大模型与规则引擎双路由降级 (100% 可用性保障)**：
     - 后端服务：`FinanceBudgetAiService` 基于微服务 `alex-ai-dev` (`AiAnalyzeApi.chat()`) 提供深度消费画像洞察，在 AI 服务未就绪或响应超时/解析异常时，无缝平滑回退至本地规则引擎（`isFallback: true`），系统 0 抛错，保证 100% 高可用。
     - **智能推荐测算 (`POST /finance-budget/ai/recommend`)**：
       - 分析近 3 个月历史流水，智能计算月均真实支出；
       - 生成三档个性化预算额度：节约档（85% 强力存钱）、适度档（100% 推荐平衡）、宽松档（115% 宽松余量）；
       - 智能识别核心高频消费分类并推荐纳入重点管控，生成个性化财务健康建议文案与每日均摊消费建议额度。
     - **自然语言一句话口语指令解析 (`POST /finance-budget/ai/parse`)**：
       - 支持口语化输入（如“帮我设2500预算，重点管餐饮和交通”、“设3000预算”、“不设上限”）；
       - 智能抽取 `budgetAmount`、`categoryCodes` 与管控意图，前端秒级自动回填表单并给出解析理由提示。
   - **PC 端与移动端全场景落地**：
     - **PC 端 (`alex_miaosha_front`)**：预算调整弹窗内嵌高质感渐变灵动 AI 面板，支持三档胶囊一键采纳、重点分类一键采用、口语化快捷指令 Chips 与一键解析回填。
     - **移动端 (`alex_miaosha_mobile`)**：二级独立预算路由页面顶部内嵌轻量 AI 智能卡片，支持口语输入、三档推荐一键填单、重点分类一键勾选，并全面集成触觉微震动（`navigator.vibrate`）提升操作手感。
   - **Taste-Skill 与 Impact-Cable 深度体验优化演进 (2026-10-12)**：
     - **渐进式折叠收敛 (Progressive Disclosure)**：PC 弹窗与移动端 AI 卡片默认收起为 ~40px 高度的高质感摘要条，释放 220px+ 黄金视口，彻底杜绝首屏遮挡与长滚动条压迫；支持一键平滑展开完整口语输入与三档卡片。
     - **心智融合 (Unified Presets)**：将 AI 适度推荐额度直接融入下方预算额度输入框的“快捷填充”行内（打上 `AI推荐` 徽标），消除两套金额选择器的认知混乱。
     - **回填微光与触感闭环 (Flash Highlight & Haptics)**：AI 解析或一键采纳时，为受影响的金额输入框和分类容器触发 1.2 秒柔和蓝紫光晕脉冲，移动端伴随双击触觉震动反馈，给予用户清晰笃定的交互结果。
     - **移动端输入与防遮挡**：增加口语指令一键清空图标，优化键盘弹出与视口联动体验。


15. **零花钱预算 AI 辅助与超支状态机端到端自动化测试 (2026-10-15 E2E 闭环)**：
   - **测试架构与遵循规约**：
     - 遵循项目根目录 `TESTING_STANDARD.md`，使用 Playwright + Vite 模拟沙箱运行端到端 Smoke 套件（`pnpm test:ai:smoke`）；
     - 所有交互元素与断言节点均绑定语义化 `data-testid`，禁止中文文案硬编码选择器；
     - 拦截 `/api/am-` 微服务接口，严密隔离 AES-128-CBC 加密报文，确保测试在离线与 CI/CD 场景具备 100% 幂等性。
   - **测试用例清单与状态机覆盖**：
     - `AI-BUDGET-001` (AI 智能测算与三档推荐采纳)：进入财务收支明细大盘，打开预算弹窗，验证 AI 测算成功后自闭环展开三档推荐卡片（节约档 85%、适度档 100%、宽松档 115%），点击适度档推荐（2600.00），断言金额输入框准确回填并高亮。
     - `AI-BUDGET-002` (口语化指令解析与回填)：展开 AI 面板，模拟输入自然语言口语指令（如“帮我设2500预算，重点管餐饮和交通”），点击“AI 解析回填”，断言金额输入框自动回填 2500 并正确应用分类范围。
     - `BUDGET-E2E-STATUS-OVER` (预算超支状态机判定)：模拟当月实际开销（2850）超出设定的预算额度（2000），页面大盘状态机精准判定并触发超支展示：断言 `budget-over-tag`（已超支红色标签）成功呈现，超支透支金额负数格式化展示为 `-¥850.00`。
   - **核心 DOM 测试桩契约 (`data-testid`)**：
     - `btn-open-budget-modal`：大盘卡片“调整预算”按钮
     - `budget-over-tag`：超支状态警示标签
     - `budget-remaining-value`：剩余可用 / 超支透支金额展示节点
     - `btn-fetch-ai-recommend`：AI 助手“智能测算”按钮
     - `btn-toggle-ai-expand`：AI 助手展开/收起切换按钮
     - `ai-command-input`：自然语言指令输入框
     - `btn-ai-parse-command`：AI 指令解析并回填按钮
     - `btn-apply-tier-moderate`：适度档推荐金额一键采纳卡片
     - `budget-amount-input`：预算金额数字输入组件
   - **工程防护增强**：
     - `useUserInfo.ts`：增加 `Array.isArray(data) ? data : []` 边界防线，规避 Ant Design Vue `VcSelect` 在异常返回下触发 `toRaw(...).slice is not a function` 页面崩溃；
     - `BudgetAiAssistantPanel.vue`：测算成功回调中自动触发 `isAiExpanded = true`，大幅优化用户操作流向感知。


16. **移动端零花钱预算 AI 辅助与触觉震动端到端自动化测试 (2026-10-15 移动 E2E 闭环)**：
   - **移动端多用例矩阵**：
     - `GIFT-AI-MOBILE-BUDGET-001` (独立二级路由页面加载与三档采纳)：通过二级动态路由 `#/finance/financeManager/budgetManage` 进入页面，自动触发测算，点击适度档推荐（¥2,600），断言预算金额输入框（`data-testid="input-budget-amount"`）回填为 `2600`。
     - `GIFT-AI-MOBILE-BUDGET-002` (自然语言口语指令回填与触觉振动断言)：展开 AI 面板，模拟输入口语指令“帮我设2500预算，重点管餐饮”，点击“回填”，通过 `installHapticProbe` 拦截并成功断言 `navigator.vibrate(10)` 触觉震动次数递增，断言金额与重点分类正确应用。
     - `BUDGET-MOBILE-STATUS-OVER` (移动端超支状态机判定)：进入财务收支明细页，模拟实际开销 2850 超出预算 2000，断言零花钱大盘卡片渲染“超支透支”健康徽标（`data-testid="budget-health-badge"`）与 `-¥850.00` 负数展示（`data-testid="metric-remaining-amount"`）。
   - **双端测试桩标准化规范**：
     - `data-testid="budget-health-badge"`：健康度徽标
     - `data-testid="metric-remaining-amount"`：大盘常驻条金额节点
     - `data-testid="btn-mobile-ai-recommend"`：移动端测算按钮
     - `data-testid="btn-mobile-toggle-ai"`：移动端折叠展开按钮
     - `data-testid="mobile-ai-command-input"`：移动端口语输入框
     - `data-testid="btn-mobile-ai-parse"`：移动端 AI 解析按钮
     - `data-testid="btn-mobile-apply-moderate"`：移动端适度档推荐卡片
     - `data-testid="input-budget-amount"`：移动端预算金额输入组件
   - **全量测试通过性**：移动端 AI Smoke 测试套件（`pnpm test:ai:smoke`）7/7 100% 通过，66/66 单元测试全过，生产打包验证完成。


17. **移动端个人财务与零花钱预算完整 E2E 烟测闭环 (2026-10-15 移动质量工程)**：
   - **核心用例**：
     - `FINANCE-E2E-MOBILE-POCKET-BUDGET` (零花钱卡片展开与预算设置全流程)：验证 `PocketMoneyCard` 展开、跳转预算设置、动态分类触控 Chip 选择、收支多选、保存预算（断言 POST `/finance-budget/save` 与路由回退）；
     - `FINANCE-E2E-MOBILE-KEYPAD-RECORD` (九宫格触觉反馈与数字键盘极速记账)：验证 `FinanceCategoryGrid` 餐饮点选与触觉探针（`navigator.vibrate`）捕获、定制数字键盘 `FinanceKeypad` 连续输入（`88.5`）、表单提交与保存返回列表（断言 POST `/finance-info` 与路由跳转）。
   - **新增组件契约桩 (`data-testid`)**：
     - `pocket-money-header`：零花钱卡片展开折叠常驻条
     - `chip-direction-expense` / `chip-direction-income`：收支多选方向 Chips
     - `chip-category-{name}`：动态分类 Chips
     - `btn-save-budget`：保存预算设定按钮
     - `category-cell-{name}`：记账九宫格分类项
     - `keypad-btn-1` ~ `keypad-btn-9`、`keypad-btn-dot`、`keypad-btn-submit`：定制数字键盘按键
     - `hero-amount-val`：顶部大字金额展示区
   - **质量矩阵**：移动端 E2E 烟测 6/6 PASS，单元测试 66/66 PASS，生产构建无告警无错误。

18. **多角色权限矩阵（RBAC）与跨租户/跨机构越权防御闭环 (2026-10-15 安全工程)**：
   - **三层全栈权限隔离体系**：
     - **PC 端流水行级管控 (`RBAC-PC-MEMBER-OWNER-PROTECT` & `RBAC-PC-ADMIN-FULL-MANAGE`)**：普通家庭成员（`family_member`, ID=2, OrgId=20）仅可操作本人流水，他人流水复选框置灰禁选且隐藏编辑/删除操作列；家庭管理员（`family_admin`, ID=1, OrgId=20）具备全员流水复选与编辑/删除权限；
     - **PC 端预算只读保护**：`usePermission` 校验 `finance:financeManager:budget:edit`，普通成员打开弹窗展示只读警告横幅（`data-testid="budget-readonly-alert"`）、金额与分类输入全面禁用、隐藏保存按钮；
     - **移动端浏览模式横幅与只读 (`RBAC-MOBILE-MEMBER-READONLY-NOTICE`)**：家庭成员打开零花钱卡片展示“预算详情”按钮（非管理员不展示“预算设置”），进入页面顶部渲染`.readonly-notice-banner`（“家庭成员浏览模式，仅家庭管理员可调整月度预算设定”），金额输入框置为 `readonly`，隐藏保存预算设定按钮，仅展示“返回列表”；
     - **后端服务层越权防御 (`FinanceBudgetOwnershipIT`)**：
       - `resolveTargetOrgId` 强制校验当前登录用户机构，跨机构查询/保存强制抛出 `FinanceException.forbidden("无权访问其他机构数据")`；
       - `saveMonthlyBudget` 执行 `assertCanEditBudget` 防线，普通成员无 `finance:financeManager:budget:edit` 权限尝试调用写接口强制拦截抛出 `FinanceException.forbidden("无权保存预算设定，仅家庭管理员可操作")`。
   - **验证通过性**：
     - PC 端 AI Smoke 测试套件（`pnpm test:ai:smoke`）：12/12 100% PASS；
     - 移动端 AI Smoke 测试套件（`pnpm test:ai:smoke`）：10/10 100% PASS；
     - 后端权限集成测试（`FinanceBudgetOwnershipIT`）：4/4 100% PASS；
     - 全栈回归：PC 单测 53/53 PASS，移动单测 66/66 PASS，双端打包构建零报错。

### 4.4 月度零花钱预算超支与消息中心联动 (2026-10-20)
- **XXL-Job 巡检联动 (SysMessageRemindJob & SysMessageInfoServiceImpl)**：
  - 每日全量巡检当月零花钱预算与实际消费：collectBudgetRemindersForJob；
  - 判定超支（实际支出 > 预算）：生成大类 BUDGET、子类 BUDGET_OVER、优先级 HIGH 的站内信；
  - 判定临近预警（使用率 80%~100%）：生成子类 BUDGET_WARN、优先级 MEDIUM 的站内信；
  - 幂等去重键设计：BUDGET:{orgId}:{userId}:{subType}:{YYYY-MM}，按月严格防重复打扰，并安全联动微信模板消息；
- **PC 端与移动端路由直达**：
  - PC 端消息卡片解析 BUDGET_OVER 标签（红）与 BUDGET_WARN（橙），点击直达 #/finance/financeManager；
  - 移动端消息卡片解析对应视觉徽章与 alance-pay 图标，点击直达 #/finance/financeManager/budgetManage 预算配置面板。

### 4.5 PC 端日常记账心流与零花钱预算穿透联动演进 (2026-10-26 Taste-Skill & Impact-Table 全面落地)
- **极速心流记账抽屉升级 (inance-manager-detail)**：
  - **分类语义彩色药丸 (Category Pill Chips)**：提供精致生动 Emoji（🍱餐饮美食、🚗交通出行、🛒日用百货、💡水电燃气、☕咖啡饮品、🎮数码娱乐、💊医疗健康、💰工资奖金等）；支持点选/反选与智能名称回填；
  - **连续记账与键盘心流 (Ctrl + Enter)**：支持 Ctrl+Enter / Cmd+Enter 快速触发「保存并再记一笔」或「保存」，连续记账无需打断手势心流；
- **预算实时健康感知与消费导航 (FinanceOverviewCard & useFinanceSummary)**：
  - **动态建议日均消费控制线 (udgetDailyGuide)**：在当前月份有预算时，实时计算 emainingDays 与建议日均消费额度（suggestedDaily = remainingAmount / remainingDays），展示如「剩余 14 天 · 建议日均 ≤ ¥86.50」；
  - **超支警报视觉微动效 (pulseGlow)**：超支时触发红色发光呼吸微脉冲动画，提升风险感知度；
- **打破信息孤岛穿透联动 (scrollToTable)**：
  - 点击预算卡片分类标签或当月预算金额时，不仅过滤账目流水，还自动调用 	ableContainerRef.value?.scrollIntoView({ behavior: 'smooth' }) 平滑滚动聚焦到明细表格；
- **自动化测试锁死 (inanceTasteAndDailyGuide.test.ts)**：
  - 编写 3 项专门测试，100% 覆盖未设置预算、正常当月预算日均均摊、超支标记等核心契约。