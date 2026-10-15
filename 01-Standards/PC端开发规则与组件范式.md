---
title: PC 端开发规则与组件范式
aliases: [PC端规则, PC组件范式, PC开发规范]
created: 2026-09-16
updated: 2026-10-15
status: active
tags: [techspec, system, frontend]
---
# 💻 PC 端开发规则与组件范式 (`alex_miaosha_front`)

返回导航：[[Home]] | [[00-System/全栈系统架构总览与交互流|系统全景]]

---

## 1. 技术栈与架构基准

- **核心框架**：Vue 3 (Composition API) + TypeScript + Vite；
- **包管理工具**：`pnpm`（CAS 内容寻址硬链接存储，全局 `pnpm-lock.yaml` 约束）；
- **UI 组件库**：Ant Design Vue v4 (`ant-design-vue`)；
- **全局状态**：Pinia + `pinia-plugin-persistedstate`；
- **自动化插件**：`unplugin-auto-import` 与 `unplugin-vue-components`（常用 API 如 `ref`, `computed`, `watch`、`src/composables` 下所有业务 hooks 如 `usePagination`, `useRowSelection`, `useDictInfo` 及 AntD 组件自动解析导入）。

---

## 2. 核心红线与强制工程规约 (Ponytail Red Lines)

> [!CAUTION]
> ### ⛔ 严禁违背的三大开发红线
> 1. **ID 安全契约（字符串化）**：
>    - 前端处理主键/外键 ID 时，**一律保持为 `string` 字符串类型**，严禁将其作为 JavaScript `number` 解析（防止低位数字截断变为 `00` 导致资产/权限错乱）。
> 2. **API 响应解构范式**：
>    - 发起接口请求并处理响应时，**统一采用对象解构形式**：
>      ```typescript
>      const { code, data, message } = await api();
>      ```
>    - 严禁直接通过 `res.code` 链式点读取。
> 3. **Pinia 共享菜单树防污染**：
>    - 全局菜单树从 Redis 缓存获取后，在前端做角色过滤或路由重组时，**必须使用 `structuredClone(menuTree)` 进行深拷贝**，严禁就地修改 `node.children`，防止污染共享内存与缓存。
> 4. **路由守卫双保险与动态路由内存生命周期**：
>    - 动态路由生成与注册（`addRouter()`）**必须由 `try...finally` 包裹**，确保无论路由树为空、权限受限提前 return 还是接口异常，`userStore.changeRouteStatus(true)` 必定执行，保证有限状态机收敛进入终态；
>    - 路由注册状态标志位（`hasMenu` / `getRouteStatus`）**必须严格维持在前端内存生命周期中，严禁持久化到 localStorage**。同时路由全局守卫**必须采用物理双保险**：`if (!userStore.getRouteStatus || routes.length <= BASE_ROUTE_COUNT)`，确保页面刷新（F5）导致 Vue Router 内存路由树重置后，即使本地存储有残留标记，也能必定触发重新挂载，彻底根治刷新后仅剩首页的顽疾；
>    - 全局路由表 `routes` 声明为 Vue `reactive([...])` 响应式数组时，所有挂载的 Vue 组件对象（`Layout`, `ParentLayout` 以及视图组件模块）**必须统一使用 `markRaw()` 包装**，严禁让 Vue 深度代理组件实例本身，消除运行时性能开销与控制台警告；
>    - 后端接口 `GET /user/menus` 已在服务端完成严格的 RBAC 菜单可见性过滤，前端 `addRouter()` 对无 `permissionCode` 或非超管用户返回的有效菜单节点**必须默认放行注册**，严禁在前端再次以空权限码将合法菜单丢弃。
> 5. **Token 过期与僵尸态自动清理 (Token Reset SOP)**：
>    - 当接口响应 401/403 或冷启动因 Token 过期导致路由获取失败时，**必须统一调用 `userStore.resetAuth()`** 彻底销毁本地存储与内存中的 Token、用户上下文及路由状态，并安全跳转至登录页（携带原 `redirect` 地址），防止用户关闭浏览器后隔天冷启动引发无限死循环重定向。
> 6. **日期/时间统一收敛于 `@/utils/dayjs`（仅导出类型，禁导实例）**：
>    - 严禁在业务页面或组件中直接 `import dayjs from 'dayjs'`，底座 `src/utils/dayjs/index.ts` 仅导出 `type Dayjs`，不导出 `dayjs` 实例；
>    - 格式化与时间运算统一使用 `@/utils/dayjs` 提供的纯工具函数（`formatDate`、`formatTime`、`formatMonth`、`formatDayjs`、`nowDayjs`、`getMonthRange`、`getMonthRangeDayjs`）；
>    - 常用日期/时间格式常量统一收敛于底座（`defaultDateFormat = 'YYYY-MM-DD'`、`defaultTimeFormat = 'YYYY-MM-DD HH:mm:ss'`、`dataTimeFormat = 'YYYY-MM-DD HH:mm'`、`dateFormatYYYYMM = 'YYYY-MM'`），**业务组件中直接引入并在模板或调用处直接使用（如 `:format="dataTimeFormat"`），严禁在组件内声明冗余中间变量进行二次赋值（如 `const dateFormatter = dataTimeFormat`）或散落硬编码魔法字符串**；
>    - 范围筛选与默认区间严禁在组件中手写 `dayjs().startOf('month')`，统一采用 `getMonthRange()`（字符串元组）或 `getMonthRangeDayjs()`（Dayjs 元组，专供 AntD RangePicker presets）。
> 7. **Pinia 持久化配置适配规约 (`pinia-plugin-persistedstate` v4)**：
>    - 基础设施层 `piniaPersistConfig(key, paths)` 中，持久化字段键必须遵循 v4 标准规范配置为 `pick: paths`（而非旧版本 `paths`）；
>    - 配置参数类型统一绑定 `PersistenceOptions`，确保类型推断与运行时存储白名单完全对齐。

---

## 3. 组件开发与代码组织范式

### 3.1 类型与配置统一管理
- 导入 Pinia 状态或 Vue 组件中的用户、角色、机构相关数据类型时，**必须统一在对应页面配置文件夹中导入**（例如 `@/views/user/roleInfo/config`、`@/views/user/menuInfo/config`），严禁在单页面内重复冗余定义相同的 interface。

### 3.2 严禁重复显式 import
- 对于已由 Vite 插件自动注册的 API（`ref`, `computed`, `watch`, `useRoute`, `useRouter`）与 Ant Design Vue 组件，**严禁在 `.vue` 文件中手动重复 `import`**。

### 3.3 路径别名强制使用规范 (Path Alias Standard)
- 所有模块、配置文件、API 接口、数据类型及组件的导入，**一律强制统一使用 `@/` 路径别名**（例如 `@/views/product/pmsAttr/pmsAttrListTs`、`@/views/finance/financeManager/api`）；
- **严禁使用任何相对路径（如 `./`、`../`、`../../`）**，杜绝层级相对寻址带来的歧义与移动文件时的断链隐患，保持全栈工程导入范式的高度一致性。

### 3.4 SVG 图标引用范式
- 严禁在模板中使用已废弃的旧 `<MySvgIcon>` 标签；
- 本地 SVG 图标统一通过 `unplugin-icons` 机制（如 `~icons/my-menu-svg/*`、`~icons/my-finance-svg/*`、`~icons/my-soft-svg/*`）按需引入，或使用 `@/views/common/config` 下的 `iconComponentMap` 结合 `<component :is="iconComponentMap[key]" />` 动态组件机制渲染。

### 3.5 页签生命周期与局部无感微刷新范式 (`my-tabs`)
- **双轨交互触达模型**：
  - 页签操作统一支持**就近右键菜单（Tab ContextMenu）**与**右侧快捷工具组（Toolbar）**双轨触达，禁止仅提供单一隐晦图标；
  - 提供 7 大标准化原子操作：刷新当前页、关闭当前、关闭左侧、关闭右侧、关闭其他、关闭所有、内容区全屏；
  - 操作项严格结合当前页签位置实现智能禁用（如无左侧业务页签时禁用关闭左侧、仅首页时禁用关闭其他/关闭所有）。
- **禁止破坏性整页重载 (`window.location.reload`)**：
  - 局部无感微刷新统一采用 Layout 级 `provide('reloadRouteView', reloadRouteView)` 配合 `<router-view v-if="isRouterAlive" />` 的 `nextTick` 切换机制；
  - 严禁直接调用 `window.location.reload()`，确保 Sider/Header 布局、Pinia 用户登录态不发生白屏闪烁或重绘。
- **首页不变性 (Home Invariant)**：
  - 首页页签始终固定于首位（index 0）且 `closable = false`，禁止被关闭或移位。
- **快捷工具条 Tailwind 现代美学规范 (Tailwind Modern Design System)**：
  - **彻底摒弃灰色背景**：常态与悬浮均严禁使用任何灰底、灰色边框（彻底杜绝脏色与方块割裂感）；
  - **常态纯净呼吸**：`bg-transparent text-slate-400`（`#94a3b8`），图标与背景天然融合；
  - **悬浮品牌微光色晕 (Tailwind Sky-Blue Wash)**：悬浮时浮现轻透天空蓝微底（`bg-blue-50` `#eff6ff` + `border-blue-100` `#dbeafe`），图标跃升为科技蓝（`text-blue-600` `#2563eb`），外加柔和天空蓝微投影（`shadow-sm`），结合 `transform: scale(1.12)` 平滑放大；
  - **点击弹性反馈**：点击时 `active:scale-95` 微收缩弹性复位，圆角统一定义为 `rounded-lg`（`8px`）。

### 3.6 自定义权限指令 (`v-permission`) 挂载规约
- **真实 DOM 实体节点契约**：Vue 3 自定义指令（如 `v-permission`，内部依赖 `el.parentNode?.removeChild(el)`）必须挂载在具备实体 DOM 根节点的原生元素或组件上；
- **禁止直接挂载至高阶抽象组件**：Ant Design Vue 的 `<a-popconfirm>`、`<a-tooltip>`、`<a-dropdown>` 等属于由 Trigger/Popover 拼接的非单一真实 DOM 根节点组件。直接在其上声明 `v-permission` 必定触发 Vue 运行时黄色警告：`[Vue warn]: Runtime directive used on component with non-element root node. The directives will not function as intended`，并可能导致指令失效或 DOM 销毁异常；
- **标准包裹范式**：遇此类组件需实施按钮级权限控制时，**必须统一在外层包裹实体 `<span>` 并挂载 `v-permission`**：
  ```html
  <span v-permission="'user:delete'">
    <a-popconfirm title="确认删除?" @confirm="handleDelete(record.id)">
      <a-button type="primary" size="small" danger>删除</a-button>
    </a-popconfirm>
  </span>
  ```

### 3.7 Tailwind 现代美学设计系统规约 (Modern SaaS / Tailwind Tokens)
- **工程底座集成规约 (Vite + PostCSS + Tailwind 3.x)**：
  - 前端工程基于 `tailwindcss` + `postcss` + `autoprefixer` 构建 JIT 原子样式层；
  - **核心避坑红线 (AntD 护城河)**：`tailwind.config.js` **必须显式声明 `corePlugins: { preflight: false }`**，严禁引入 Tailwind 默认的 Preflight 全局重置样式，防止将 Ant Design Vue 4.x 的按钮背景、输入框和原生表格边框排版冲掉；
  - **样式注入规范**：在 `src/style/index.less` 顶部仅声明 `@tailwind utilities;`，确保工具类具备最高作用域且不破坏既有组件库。
- **画布背景规范**：
  - 彻底淘汰生硬灰暗背景（`#f0f2f5`、`#f4f6fa`），统一升级为 Tailwind `bg-slate-50/60`（`#f8fafc`），营造呼吸感与透气感；暗黑模式对齐 `dark:bg-slate-950`。
- **容器与卡片质感**：
  - 标准卡片与面板统一采用 `rounded-2xl`（`16px`）柔润圆角，辅以细微边框 `border border-slate-200/80` 与轻投影 `shadow-xs` / `shadow-sm`；
  - 严禁使用生硬的 `7px` 边框与沉重的 `0 7px 18px` 脏暗阴影；
  - 悬浮交互支持自然微浮动：`hover:-translate-y-0.5 hover:shadow-md transition-all duration-200`。
- **KPI 指标卡 (Metric Cards)**：
  - 统一采用极简白底卡片或药丸胶囊，数字使用加粗高可读性字体（`text-slate-900 font-extrabold`）；
  - 图标徽标统一采用双环药丸轻色底（Emerald: `bg-emerald-50 text-emerald-600 ring-1 ring-emerald-100/80`，Rose: `bg-rose-50 text-rose-600 ring-1 ring-rose-100/80`，Blue: `bg-blue-50 text-blue-600 ring-1 ring-blue-100/80`，Amber: `bg-amber-50 text-amber-600 ring-1 ring-amber-100/80`）。
- **过滤栏与胶囊标签**：
  - 快速过滤与统计标签统一采用 `rounded-full` 药丸胶囊风格，激活态采用现代品牌主色搭配浅色底。
- **表格列宽工程红线**：
  - Ant Design Vue 表格列配置必须逐列显式指定 `width` 与对齐方式（居中/居左），防止因内容长度动态伸缩引发列挤压扭曲。

### 3.8 中后台表格满屏自适应与防外层 Y 轴滚动条范式 (Zero-Page-Scroll SOP)
- **视口根基锁定**：
  - 全局布局外层容器 `.main-layout` 必须严格保持 `height: 100%; overflow: hidden;`；中间主体 `.layout-content` 设置 `flex: 1; min-height: 0;`；
  - 路由容器 `.content-container` 设置 `flex: 1; min-height: 0; overflow-y: auto;`。
- **业务表格页一屏自适应核心结构**：
  - 页面根节点（如 `.finance-manager-page`）设置纵向 Flex 布局：`display: flex; flex-direction: column; height: 100%; overflow: hidden; box-sizing: border-box;`；
  - 顶部筛选栏（`.search`）、概览条（`.summary-bar`）、操作按钮区（`.button`）统一声明 `flex-shrink: 0;`；
  - 核心表格卡片（`.content`）声明 `flex: 1; min-height: 0; display: flex; flex-direction: column; overflow: hidden;`；
  - 表格卡片内部利用 `ResizeObserver` 动态计算容器可用高度，并实时传给 `<a-table :scroll="{ x: 1080, y: tableScrollY }">`；
  - **效果保证**：彻底根治页面级出现垂直滚动条导致表头被滚走、分页栏错位的弊端，确保任何分辨率与筛选器展开/折叠状态下，表头和底部分页器永久锁定可视，滚动仅在表格 Body 内部发生。

### 3.8 左右双栏树表联动现代 SaaS 范式（机构管理/部门组织模版）
- **布局架构解耦**：
  - 左侧独立组织架构树卡片（`w-full lg:w-72 flex-shrink-0 bg-[var(--card-bg)] border border-slate-200/90 rounded-2xl shadow-xs p-4`），自带独立过滤徽标与重置链接；
  - 右侧主工作区纵向堆叠：紧凑单行高效搜索栏卡片（`filter-card`） + 一体化表格面板（`table-card`）；
- **树表联动防抖与状态重置**：
  - 点击左侧树节点时，仅将选中节点 ID 绑定至 `searchInfo.parentId`，配合 `watch(searchInfo, debounceQuery, { deep: true })` 触发无感自动刷新；
  - 必须提供显式的 `clearTreeFilter` 机制，支持用户一键清空树选中节点并查看全局数据；
- **操作列微底悬浮胶囊**：
  - 操作列（编辑/删除）统一采用微底轻色胶囊按钮（如编辑采用 `bg-blue-50 text-blue-600 hover:bg-blue-100 dark:bg-blue-950/60`，删除采用 `bg-rose-50 text-rose-600 hover:bg-rose-100 dark:bg-rose-950/60`），提升点击靶心与视觉呼吸感。

### 3.9 菜单与多级路由权限管理 SaaS 范式（菜单管理/子菜单抽屉模版）
- **根菜单与子菜单层级交互**：
  - 根菜单列表维持 `searchInfo.parentId = '0'` 作为顶层约束；
  - **根菜单查询前后端契约**：前端以 `parentId = '0'` 发起根菜单分页查询，后端 `MenuInfoMapper.xml` 必须兼容将 `parentId == 0 || parentId == '0'` 映射为 `(parent_id IS NULL OR parent_id = 0)`，与 Service 层 `isRootParent` 业务判定对齐，确保数据库中系统预置的一级菜单（`parent_id IS NULL`）能准确查出；
  - 每行根菜单提供独立的「子菜单」胶囊入口（带 `BranchesOutlined`，挂载 `v-permission="'menu:add'"` 与 `data-testid="rbac-menu-row-add-child"`），点击唤起 1000px 现代抽屉（`SubMenuManager`）；
  - 抽屉内部复用 `.modern-menu-table` 样式、药丸标签与悬浮微底操作胶囊，保持层级交互一致性；
- **状态与属性胶囊映射规范**：
  - 权限标识 (`permissionCode`)：采用高辨识度深蓝微底标签（`bg-blue-50 text-blue-700 dark:bg-blue-950/50`）；
  - 隐藏菜单 (`hideInMenu`)：显示状态为淡灰胶囊，隐藏状态为琥珀暖色胶囊；
  - 状态 (`status`)：启用为翡翠绿胶囊，禁用为淡灰胶囊；
### 3.10 KPI 指标卡微图表渲染范式 (Metric Card ECharts Sparkline)
- **按需加载契约**：卡片级微图表必须统一使用 `@/utils/echarts/loadEcharts` 异步加载，严禁全量同步 `import * as echarts`，防止 Vite 预构建打乱类注册顺序；
- **交互与浮层规范**：微图表必须开启 `tooltip: { trigger: 'axis', confine: true }`，悬停时以高对比度浮层展示具体月份与格式化货币金额（如 `¥36,100.00`），杜绝无法感知的静态死图；
- **自适应与销毁保障**：必须监听 `window.resize` 自动调用 `chartInstance.resize()`，并在 `onUnmounted` 生命周期中显式调用 `chartInstance.dispose()` 清理实例，防止 SPA 页面切换产生内存泄露。

### 3.11 AI 赋能交互组件范式 (AI Fast-Entry & Smart Recommendation)
- **财务礼金自然语言快速录入面板 (`.ai-parse-panel`)**：
  - 录入抽屉顶部设立自然语言快捷输入区，单句文本一键调用 `aiParseGiftRecord` 智能提取多要素，并自动回填表单字段（`direction`, `amount`, `payTime`, `externalPersonId`, `eventType`, `remark`）；
  - 严格保持 ID 安全契约，通过 `normalizeGiftResponse` 保证解析返回的实体 ID 为 `string` 类型；
  - 监听事由类型与亲友变动，自动调用智能推荐接口获取推荐金额、礼金考量（`aiReasoning`）以及场景贺词（`aiGreetingTip`），并提供一键复制功能（`copyGreetingTip`）。
- **营销优惠券 AI 方案生成 (`.ai-plan-bar`)**：
  - 优惠券详情页顶部提供 AI 营销策划栏，输入营销目标、预算限额与目标类目，调用 `aiPlanCpnCoupon` 获得最优发券量、门槛与抵扣金额、营销标题与活动说明，支持一键回填表单。
- **电商秒杀商品营销文案生成 (`.ai-copy-modal`)**：
  - 商品列表与详情页中提供独立 AI 营销文案弹窗，调用 `generateProductAiCopy` 输出 30 字爆款标题、Slogan、3-4 条核心卖点与 150 字种草转化详情，提供分项一键复制体验。
- **RBAC 岗位权限智能推荐与一键勾选 (`.ai-role-recommend-bar`)**：
  - 角色权限抽屉内挂载智能推荐栏，基于岗位名称与职责描述调用 `aiRecommendRolePermissions`，后端推导契合的菜单ID列表（纯 `string` 数组并已自动补齐父节点），前端一键自动同步勾选 `selectPermission` 并展开树节点，同时展示 AI 推荐理由。

### 3.12 表格分页与行选择 Composable 极简范式 (`usePagination` & `useRowSelection`)
- **拒绝重型 UniversalTable 容器组件**：
  - 遵循 Ponytail 极简原则与 YAGNI。Ant Design Vue 的 `<a-table>` 本身是标准高质量组件，为其额外二次包装抽象组件会导致插槽深度穿透、泛型推断变脆与 Midscene 测试定位层级破坏；
  - 采用 **Composable 逻辑下沉 + 原生 `<a-table>` 模板直出** 的轻量组合模式。
- **分页 Composable 极简调用 (`usePagination`)**：
  - 支持传入可选回调 `{ onChange: (p) => query() }`，自动在换页与切换条数时触发数据加载，彻底消除各个页面中机械重复的 `const handleTableChange = (p) => { paginationChange(p); query(); }` 样板包装代码；
  - 模板中直接绑定 `@change="handleTableChange"`。
- **行选择 ID 安全护城河 (`useRowSelection`)**：
  - 必须统一通过 `const { selectedRowKeys, rowSelection } = useRowSelection()` 管控表格多选；
  - 内部自动将 `selectedRowKeys` 严格映射并锁定为 `string[]` 类型，杜绝雪花算法大数 ID 转为 JavaScript `number` 产生精度丢失与越权 Bug。
- **Composables 自动注入与禁止手动显式 import**：
  - `vite.config.mts` 中已将 `src/composables` 配置进 `unplugin-auto-import` 的 `dirs` 扫描范围；
  - `usePagination`、`useRowSelection`、`useDictInfo` 等业务 hooks 均自动全局解析并注入类型声明，**页面中严禁重复手动显式 `import`**，以保持代码精简并消除由于漏写或路径手误带来的 `ReferenceError` 隐患。

### 3.13 导航菜单异步路由防竞态与 `useInjectMenu` 避坑 SOP
- **问题现象**：页面深层刷新或直接按 F5 访问二级/三级路由时，控制台抛出报错：`TypeError: Cannot destructure property 'prefixCls' of 'useInjectMenu(...)' as it is undefined`；
- **底层根因**：
  1. **禁止使用 `:items` 属性**：在 Ant Design Vue 源码中，`<a-menu>` 的 `items` 明确标注为 `// donot use items, now only support inner use`。`:items` 内部通过 `useItems.js` 在响应式 `watch` 中脱机执行 `_createVNode(SubMenu)`，脱离了当前组件模板渲染树上下文。在 Vue 3.5 下遇上深层刷新 + `openKeys` 展开过渡动画（`<Transition>`），内部函数式组件 `SubMenuList` 无法通过 `inject(MenuContextKey)` 向上捕获父级上下文，必然抛出解构异常；
  2. **同级 Key 碰撞放大**：若后端返回的父级菜单数据中 `name` 均为通用字符（如 `"sub"`），导致多个同级 SubMenu 共享相同 key，Vue Virtual DOM 将其识别为同一节点并错误复用，彻底破坏动画挂载链路；
- **治理标准**：
  1. **全量回归官方标准模板语法**：严禁在 `<a-menu>` 上使用 `:items`，必须使用官方原生模板嵌套 `<a-sub-menu>` 与 `<a-menu-item>`，由 Vue 模板编译器直出原生 VNode 保证 `provide/inject` 链路 100% 同步稳定；
  2. **Key 严格全局唯一**：每个菜单节点必须拥有绝对唯一的 key（优先稳定业务标识，其次 `path`，若有重复自动追加唯一索引后缀），彻底杜绝节点复用异常；
  3. **双重跳转与展开回退**：点击事件优先识别路径格式（以 `/` 开头）与路由 `name`，展开项同步统一在 `nextTick` 中延迟执行，保证 DOM 树与上下文完全就绪。

### 3.14 弹窗模态表单统一采用 `<a-form>` 校验规范
- **禁止纯原生 input 或零散校验**：所有编辑/新增业务弹窗，均必须使用标准的 `<a-form ref="formRef" :model="form" :rules="rules">`，并用 `<a-form-item name="xxx">` 包裹输入组件；
- **校验触发与清理时机**：
  1. 弹窗打开时（如 `openModal`），必须在 `nextTick` 中调用 `formRef.value?.clearValidate()`，避免上次错误状态残留；
  2. 点击保存/确定时，必须通过 `await formRef.value?.validate()` 获得校验结果后再调用 API；
  3. 胶囊按钮、快捷预设或标签选择等非直接输入控件修改值后，主动触发 `formRef.value?.validateFields(['fieldName'])`。

### 3.15 通用基础组件归位与引用规约 (Common UI Components SOP)
- **物理目录职责绝对分离**：
  1. **公共组件统一位于 `src/components/`**：通用图表容器（`src/components/chart`）、通用文件/头像上传（`src/components/my-upload`）、机构/菜单树（`src/components/menu-tree`）、RBAC 公共构件（`src/components/rbac`）等跨模块复用组件，严禁放置于 `src/views/` 或 `src/views/common/` 目录下；
  2. **类型与配置归口导出**：通用组件配套接口与类型统一定义在 `src/components/{component-name}/config.ts` 中导出；
  3. **ID 安全契约**：上传文件等对象主键（`FileInfo.id`）必须维持 `string | number`（优先作为 `string` 字符串消费），避免雪花算法 Long 型大整数在浏览器中发生精度截断；
  4. **严禁业务域建立伪代理中转**：通用服务（如数据字典 API）统一直接从 `@/views/dict/api` 引入，严禁在 `views/finance/dict` 等业务模块内建立 `export *` 式的转发别名层，避免引发依赖分流与调用混乱。

### 3.16 Pinia 状态树组织与 Composables 职责范式
- **Pinia Store 模块规范**：
  1. **纯名词目录命名**：`src/store/modules/` 下各状态模块采用领域纯名词命名（如 `login`、`user`、`tabs`），严禁携带 `-store` 冗余后缀；
  2. **统一聚合入口 `index.ts`**：每个模块目录必须提供 `index.ts` 作为统一导出入口，支持 `@/store/modules/{module}` 简洁引入；
- **Composables 职责纯正性**：
  1. **`src/composables` 专属于 Vue 组合式函数**：目录内只允许存放 `use*` 开头、具备 Vue 响应式状态或生命周期的 Hook；
  2. **非响应式纯函数与常量严格下沉**：不依赖 Vue 响应式系统的纯映射计算、格式化算法或配置常量，必须归放到 `src/utils/`（如权限相关下沉至 `src/utils/permission/`），严禁放置于 `composables/`，避免被 Vite `AutoImport` 误识别为全局注入项而导致全局命名空间污染。

### 3.17 顶级 Layout 架构与视图纯正性规约 (Layout & Pure Views SOP)
- **顶级 Layout 目录收拢**：
  1. 应用外壳统一位于顶级 `src/layout/index.vue`，侧边栏（`my-navbar`）、顶栏右侧信息（`my-right-info`）、多页签管理（`my-tabs`）统一收纳于 `src/layout/components/`，由 `src/layout/index.vue` 显式引入组装；
  2. 严禁创建 `src/views/layout` 或 `src/views/common`，防止应用骨架部件与路由业务视图混杂；
- **视图目录 `src/views` 纯正性**：
  1. `src/views` 目录内 100% 仅存放路由业务页面，禁止存放全局组件、布局壳或通用配置；
  2. 路由动态构建统一使用纯净模式：`const modules = import.meta.glob('/src/views/**/**.vue')`，禁止在路由中编写反向排除规则（如 `!/src/views/common/**`）；
- **全局类型与 API 归口**：
  1. 跨模块通用弹窗模态框配置接口 `ModelInfo` 统一定义在 `src/types/modal.ts` 中导出；
  2. 图标映射配置统一定义在 `src/config/icons.ts` 中导出；
  3. 全局通用文件/OSS 服务 API 统一维护在 `src/api/file.ts`。

---

## 4. AI 与 E2E 自动化测试体系 (Midscene + Playwright)

- **技术组合**：`@midscene/web` + `@playwright/test`；
- **可交互 DOM 规约**：所有可交互元素必须挂载 `data-testid`（如 `ai-plan-bar`, `btn-generate-ai-plan`, `btn-open-ai-copy`, `ai-copy-input-name`, `btn-apply-ai-copy`, `rbac-ai-recommend-bar`, `btn-ai-recommend-permissions`），严禁用中文文案做精确匹配；
- **等待策略**：严禁使用 `Thread.sleep` 或 `waitForTimeout`，统一采用 `waitForResponse` / `waitForSelector` / `locator.waitFor({ state: 'visible' })` / `aiWaitFor`；
- **数据清理护城河**：测试用例产生的临时数据必须在 `try...finally` 中调用清理 API 删除，禁止残留脏数据污染开发数据库；
- **微服务接口 Mock 与加解密契约护城河**：
  - 在独立运行的 E2E 脚本（如 `run-domain-ai-smoke.mjs`）中进行路由拦截时，**必须且仅拦截以 `/api/am-` 开头的微服务后端接口**（如 `url.pathname.startsWith('/api/am-')`），严禁使用宽泛的 `**/api/**`，以防误拦截 Vite 动态加载的前端源码模块（如 `src/views/.../api/index.ts`）；
  - 网关启用了统一对称加密，前端 Axios 响应拦截器通过 `decrypt(data)` 解密。因此 Mock 响应体**必须统一经过 AES-128-CBC 加密**输出 Base64 字符串（密钥 `20230610HelloDog`，偏移量 `1234567890123456`），防止前端报 `TypeError: base64Str.indexOf is not a function`；
- **跨平台多浏览器通道弹性启动 (Browser Launch Resilience)**：
  - Playwright 启动时配置自动降级链路：优先启动独立 Chromium 实例，若未安装 shell 则自动回退至系统内置通道 `channel: 'msedge'` 或 `channel: 'chrome'`，确保在 CI 及各类开发者本地环境零报错秒级拉起；
- **运行命令**：
  - PC 端 AI 场景自动化烟测：`pnpm test:ai:smoke`（执行 `node scripts/playwright/run-domain-ai-smoke.mjs`）；
  - PC 端 RBAC 权限烟测：`pnpm test:rbac:smoke:local`；
  - 移动端礼金模块烟测：`pnpm test:midscene:gift:local`。

---

## 5. 主题换肤与暗黑模式范式 (Theme & Dark Mode)

- **设计哲学**：遵循 Ponytail 极简老炮模式，零引入任何重型外部取色板或换肤插件。基于 Ant Design Vue 4.x 原生 Design Token 引擎与 HTML5 原生 `<input type="color">` 闭环。
- **状态持久化**：通过 `useThemeStore` (`@/store/modules/theme.ts`) 统一管控 `primaryColor` 与 `isDark`，并使用 `pinia-plugin-persistedstate` 保存在 `localStorage` (`app-theme`)。
- **AntD 根注入**：在 `App.vue` 顶层 `<a-config-provider>` 动态绑定 `:theme="{ algorithm: isDark ? theme.darkAlgorithm : theme.defaultAlgorithm, token: { colorPrimary: primaryColor } }"`。
- **全局非 AntD 容器 CSS 变量桥接**：在 `src/style/index.less` 中声明 `--primary-color`、`--page-bg`、`--card-bg` 等语义化变量，并在 `html.dark` 下自适应切换；`.page-info`、`.search`、`.content` 等自定义布局容器必须绑定上述 CSS 变量，确保明暗切换时背景无黑白撕裂。

---

## 6. 构建性能与分包策略规约 (Build & Chunk Optimization SOP)

- **跨平台与 pnpm 隔离的分包正则规约**：
  - `vite.config.mts` 的 `manualChunks(id)` 必须使用规范的 `/[\\/]node_modules[\\/](pkg)[\\/]/` 正则匹配，严禁用 `id.includes('node_modules/pkg')` 写死斜杠，避免 Windows 路径反斜杠导致分包失效；
- **6 大标准持久化 Vendor 分组体系**：
  1. `framework`：`vue`, `@vue/*`, `vue-router`, `pinia`, `pinia-plugin-persistedstate`, `@vueuse/*`（高频缓存，极其稳定）；
  2. `antd-vendor`：`ant-design-vue`, `@ant-design/cssinjs`, `@ctrl/tinycolor`, `async-validator`（UI 核心库，独立长效缓存）；
  3. `antd-icons`：`@ant-design/icons-vue`；
  4. `echarts-vendor`：`echarts`, `zrender`（图表专用异步 chunk）；
  5. `particles-vendor`：`@tsparticles/*`（登录/特效背景按需异步分包）；
  6. `utils-vendor`：`axios`, `lodash-es`, `dayjs`, `bignumber.js`, `crypto-es`, `nprogress`（通用工具库）；
- **依赖瘦身红线**：
  - **严禁引入巨型通用计算引擎 `mathjs`**：金额及数值高精度运算统一使用轻量级 `bignumber.js`（`^11.0.0`），彻底杜绝为简单的加减而卷入数百 KB 的冗余依赖；
  - **严禁引入 `nanoid` 进行 DOM 拼接**：图表、画布等 DOM 挂载统一使用 Vue 3 原生 `shallowRef<HTMLDivElement>()`，严禁通过 `nanoid` 生成动态 ID 并调用 `document.getElementById` 查询；
- **超千行纯 CSS/Less 样式外置规约**：
  - 遇到含大量关键帧动画或背景造型代码的特殊视图（如 `error-404`），超过 200 行的纯样式必须独立抽离至同级 `index.less` 并通过 `<style lang="less" scoped src="./index.less"></style>` 引入，保持 Vue SFC 模板纯净专注。

---

## 7. 次千行重度业务页面精细化解耦范式 (Heavyweight Page Decoupling Standard)

- **单文件规模警戒线**：
  - 业务单文件（`.vue`）规模严控在 **500 行以内**。当单页面代码行数逼近或超过 800~1,000 行时，必须执行系统化解耦重构，全面消除“千行巨型 SFC”；
- **全端千行文件清零标杆实践**：
  1. **机构驾驶舱 (`OrgAdminDashboard.vue`)：1,061 行 ➔ 196 行（暴减 81.5%）**：
     - 指标卡行解耦为 `OrgAdminKpiRow.vue`（承载 4 大 KPI 统计与环比）；
     - 多 Tab 趋势/分类图表解耦为 `OrgAdminChartCard.vue`（承载 ECharts 走势与饼图，保留 `loadEcharts` 异步契约）；
     - 保留 `dash-org-kpi-*`、`dash-org-chart-*` 等自动化烟测契约节点；
  2. **人情事件台账 (`finance/gift/event/index.vue`)：1,070 行 ➔ 436 行（缩减 59.3%）**：
     - 指标卡与高频事件场景解耦为 `GiftEventMetricCards.vue`；
     - 事件分类字典网格、Switch 启停及编辑解耦为 `GiftEventTypeDictionary.vue`；
     - 宿主专注台账表格、高级筛选与 Drawer 表单驱动；
  3. **亲友关系档案 (`finance/gift/person/index.vue`)：1,068 行 ➔ 481 行（缩减 55.0%）**：
     - 亲友关系网络及人情净值（正负差额标签）解耦为 `GiftPersonSummaryCards.vue`；
     - 彻底清除 100+ 行废弃历史弹窗样式，保留 `gift-person-filter-scope` 契约；
  4. **礼尚往来表单抽屉 (`gift-record-form-drawer/index.vue`)：1,046 行 ➔ 493 行（缩减 52.8%）**：
     - 事件联选字段解耦为 `GiftRecordEventField.vue`（保留 `gift-record-form-event-input`）；
     - 往来亲友字段解耦为 `GiftRecordPartiesField.vue`（保留 `gift-record-form-person-input`）；
     - 金额收付字段解耦为 `GiftRecordAmountField.vue`（保留 `gift-record-form-amount-input`）；
  5. **人情往来驾驶舱 (`gift-dashboard/index.vue`)：1,036 行 ➔ 283 行（暴减 72.7%）**：
     - 趋势图表卡片解耦为 `GiftDashboardTrendCard.vue`（保留 `gift-dashboard-trend-chart`）；
     - 亲友排行卡片解耦为 `GiftDashboardRankingCard.vue`（保留 `gift-dashboard-ranking-all`）；
     - 最近流水卡片解耦为 `GiftDashboardRecentTable.vue`（保留 `gift-dashboard-record-all`、`gift-dashboard-float-add`）；
  6. **个人财务账目管理 (`selfFinance/financeManager/index.vue`)：1,169 行 ➔ 684 行（缩减 41.5%）**：
     - 账目表格与批量工具解耦为 `FinanceManagerTable.vue`；
     - 复杂收支统计运算逻辑抽离为 `useFinanceSummary.ts` composable；
  7. **个人财务统计分析 (`selfFinance/financeAnalysis/index.vue`)：998 行 ➔ 607 行（缩减 39.2%）**：
     - KPI 指标卡行解耦为 `FinanceAnalysisKpiRow.vue`；
     - 账户余额与资产网格解耦为 `FinanceAnalysisAccountGrid.vue`；
  8. **企业级登录页 (`login/index.vue`)：958 行 ➔ 473 行（缩减 50.6%）**：
     - 动态交互插画与眨眼角色解耦为 `LoginVisualPanel.vue`，彻底净化宿主表单容器。
- **子组件自动注册契约**：
  - 业务子组件均按规范置于对应视图同级 `components/` 目录下，由 `unplugin-vue-components` 自动扫描注册，**严禁在宿主 SFC 中手动重复 `import`**。

---

## 8. 代码质量与安全规约 (Code Quality & Security Standard)

- **SonarQube S2245 伪随机数发生器规范**：
  - 严禁在有安全要求、敏感操作、或易触发静态质量门禁的业务场景中滥用 `Math.random()`（会触发 SonarQube `typescript:S2245` 安全热点告警）；
  - 统一优先采用现代浏览器原生标准的 Web Crypto API（`window.crypto.getRandomValues`）生成加密安全的随机数；
  - 即使对于登录页角色眨眼等前端动画或延迟场景，如需使用随机数，优先封装 `window.crypto.getRandomValues` 生成 `[0, 1)` 区间随机数，从源头彻底杜绝安全告警与静态扫描阻断。

---

## 9. 表格高质感常驻底栏与高度自适应分页范式 (Table Sticky Toolbar & Adaptive PageSize)

- **核心设计哲学 (Taste-Driven Layout)**：
  - 在固定视口、内部滚动的中后台表格卡片中，**严禁将分页器孤立浮于白色死区最右下角**，必须将其构建为具有清晰边界与轻量背景色（`#fafafa`）的稳固底栏（`.table-footer-toolbar`）；
  - 表格声明 `:pagination="false"`，解耦内置单一右浮动分页，由自定义底栏容器承载，实现左右视觉重心平衡；
- **左右平衡与就近操作布局 (Left-Right Balance)**：
  - **左侧工作状态区**：未勾选时展示千分位账目总量（`共 6,940 笔账目`）；在有 Checkbox 勾选时平滑过渡为批量操作工具条（`已选 3 项` · `批量删除` · `清空选择`），极大缩短操作视线折返路径；
  - **右侧控制器区**：紧凑对齐 `<a-pagination size="small" />`，收敛冗余跳页输入框，保持翻页链路专注敏捷；
- **视口高度感知自适应阶梯 (Adaptive PageSize Tiers)**：
  - 统一封装 `calculateAdaptivePageSize(availableHeight)`：
    - 可用高度 `< 460px`（笔记本/分屏小视口）：默认 **`10 条/页`**；
    - 可用高度 `460px ~ 660px`（标准 1080P 显示器）：默认 **`15 条/页`**（恰好充盈首屏）；
    - 可用高度 `> 660px`（2K/大屏全屏模式）：默认 **`20 条/页`**；
  - **用户手动偏好锁定权**：系统首屏基于窗口高度智能推导初始条数；一旦用户在下拉框中主动变更（如切为 `50 条/页`），系统立即锁定偏好标识，不再被窗口缩放强制重置。

---

## 10. Vue 组件入参契约与 TypeScript 严密类型防御 (Vue Props Contract & Type Safety)

- **数组型 Props 默认值与空安全保护**：
  - 子组件定义数组型 Props 时，必须使用 `withDefaults` 赋予空数组默认值：
    ```typescript
    const props = withDefaults(
    	defineProps<{
    		recentContacts?: Array<{ id: string; name: string }>;
    		familyMembersList?: Array<{ id: string; name: string }>;
    	}>(),
    	{
    		recentContacts: () => [],
    		familyMembersList: () => [],
    	},
    );
    ```
  - 模板中对数组长度判断必须采用空值合并保护：`v-if="(recentContacts?.length ?? 0) > 0"`，严防父组件异步数据就绪前求值抛出 `Cannot read properties of undefined (reading 'length')`；
- **字典与选项遍历强制防御 `Array.isArray()`**：
  - 在选择器组件（如 `gift-org-member-picker`、`gift-person-picker`）及缓存加载层中，对入参列表一律执行 `(Array.isArray(list) ? list : []).forEach(...)`，彻底防御非数组空对象或网络桩兜底响应破坏渲染；
- **全量构建质量门禁 (`pnpm type-check`)**：
  - 项目全局开启 `vue-tsc --noEmit`，严禁在代码中遗留未使用的本地变量（`TS6133`）或隐式未判空属性调用（`TS18048`），保证生产构建 `pnpm build` 与静态类型检查持续 100% 零报错。





