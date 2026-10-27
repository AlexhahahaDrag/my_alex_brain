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
> 8. **Vue 3 响应式安全与 ShallowRef / MarkRaw 性能防御红线 (`vuejs-skills`)**：
>    - **严禁直接解构响应式对象**：严禁直接对 `props` 或 `reactive` 进行普通解构（`const { propA } = props` 会导致响应式彻底丢失），必须使用 `toRefs` / `toRef` 或 Vue 3.3+ 响应式解构编译宏；
>    - **大型对象与第三方实例浅层代理**：对于复杂的 ECharts 实例、只读大型字典、地图组件及路由动态组件，统一使用 `shallowRef` 或 `markRaw()`，彻底消除 Vue 深度依赖追踪带来的性能开销与警告；
>    - **派生状态 Computed 优先**：计算衍生值必须使用 `computed`，严禁使用 `watch` 监听 A 手动赋值给 B，消除异步延迟与竞态死循环。
> 9. **高质量业务 Composable 封装与生命周期清理红线 (`create-adaptable-composable`)**：
>    - **多态参数与 toValue 解包**：`src/composables` 下的业务 hooks 入参类型推荐使用 Vue 3.3+ 原生 `MaybeRefOrGetter<T>`，内部统一使用 `toValue(param)` 进行解包，兼容普通值、`ref` 与 getter 函数；
>    - **扁平 Ref 展开**：Composable 返回值必须为扁平 JavaScript 对象，内嵌 `ref` 或 `computed` 属性，调用方可直接安全解构；
>    - **资源生命周期闭环**：凡在 Composable 内部建立的定时器（`setInterval`）、DOM 事件监听、SSE 流式通道或 Observer，必须在 `onScopeDispose` 钩子中安全注销释放，杜绝组件卸载后发生内存泄漏；
>    - **异步竞态防御**：对高频触发的异步查询 hooks，必须内置请求序列号校验（`reqId === currentRequestId`），避免后发先至的数据被旧响应覆盖。
> 10. **Pinia 状态安全解构与 Vitest 行为驱动测试红线 (`vue-pinia` & `vue-testing`)**：
>     - **Pinia 解构必须走 storeToRefs**：从 Pinia Store 中解构 State 与 Getters 时，**必须且仅能通过 `storeToRefs(store)`**，严禁直接解构；Actions 方法可直接从 Store 实例解构；
>     - **单元测试推进微任务规范**：Vitest 单元测试中推进异步微任务队列**统一使用 `flushPromises()` 与 `nextTick()`**，坚决杜绝在测试代码中硬编码 `setTimeout` 固定延时；
>     - **雪花 ID 字符串类型防截断测试覆盖**：前端核心单元测试必须包含对大于 `Number.MAX_SAFE_INTEGER` 的超大雪花 ID 的断言，确保全链路状态机保持 `string` 类型不变。

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
- **财务衍生支付方式图标精确映射规约 (2026-10-26)**：
  - 业务中衍生出的细分子渠道（如话费衍生 `hfw`、`hfm`、`hfmj`，电费衍生 `dfw`、`dfm`）在获取渲染图标时，**必须且仅在获取图标逻辑中**通过精确字典映射转换为基础图标编码（如 `hf`、`df`）；
  - **严禁使用 `includes` 模糊字符串匹配**（避免因子串巧合引发误判）；
  - 图标精确映射与业务名称展示（如字典名称或原始编码）严格隔离，不得污染展示文本。

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
- **操作列微质感动效规范 (Hover背景 + 文字与SVG同步平滑放大)**：
  - 操作列按钮（查看/编辑/删除/成员/授权等）统一挂载 `.table-action-btn` 与语义类型类（`view-action-btn`, `edit-action-btn`, `delete-action-btn`, `member-action-btn`, `auth-action-btn`）；
  - **默认态**：纯净无边框透明底色，维持表格行视觉通透；
  - **鼠标滑过态 (Hover)**：
    - 呈现对应语义类型的微底轻色胶囊背景（查看灰底 `#f1f5f9` / 编辑蓝底 `#eff6ff` / 删除红底 `#fff1f2` / 成员绿底 `#f0fdf4` / 授权紫底 `#f5f3ff`，暗黑模式自动适配半透明高质感深色底）；
    - 按钮整体触发平滑 GPU 微缩放 `transform: scale(1.08)`（配合 `transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1)`），文字与内部 SVG 图标同步变大，且图标额外追加微放大 `transform: scale(1.15)`，形成极具触觉品质的“磁吸灵动变大”体验；
    - 彻底根除因字号突变导致的表格列宽抖动（Zero-CLS）；
  - **点击态 (Active)**：触发轻微下压回弹 `transform: scale(0.96)`，提供即时触觉反馈。

### 3.9 菜单与多级路由权限管理 SaaS 范式（左侧树+右侧全下级纯平铺列表模版）
- **左右双栏架构解耦 (Left-Tree Right-List Architecture)**：
  - **左侧菜单架构树 (`w-full lg:w-64 flex-shrink-0`)**：展示系统顶级模块与多级目录树（采用 AntD `<FolderFilled />` / `<FileTextOutlined />` 矢量图标），支持节点名称实时过滤搜索、全树一键「展开/收起」，默认展开一级目录，置顶「全部菜单（全量平铺）」快捷选项；点击任意节点联动聚焦右侧列表；
    - **a-tree 自定义 `#title` 插槽避坑 (2026-10-16)**：Ant Design Vue 4 的 `<a-tree>` `#title` 插槽只提供节点字段与 `dataRef`，**不存在 `record`**；写成 `{ title, record }` 并访问 `record.parentId` 会抛 TypeError，导致整棵树渲染为空。统一使用 `#title="{ title, dataRef }"` 并配合可选链 `dataRef?.xxx`。
    - **三态树图标与内页详情归属规范 (2026-10-16)**：三级详情/内页（`hideInMenu: '1'`）应作为三级子菜单归属到二级「*管理」菜单之下。左侧树节点图标严格遵循三态视觉映射：一级目录渲染 `<FolderFilled class="text-blue-500" />`，二级常规菜单渲染 `<FileTextOutlined class="text-emerald-500" />`，三级隐藏内页渲染 `<FileProtectOutlined class="text-slate-400" />`。
    - **非 Layout 路由自动拍平机制 (Route Flattening SOP, 2026-10-16)**：二级管理页面组件（如 `menuInfo/index.vue`）不是 `Layout`（内部不含 `<router-view />`）。在 PC 端（`src/router/index.ts`）与移动端构建 Vue Router 路由表时，凡是父节点组件不是 `'Layout'` 的子路由（三级详情页），在递归解析时必须将其自动拍平提升到外层 `Layout` 路由的 `children` 中（`nestedRoutes = childRoute.children.splice(0); routeInfo.children.push(...nestedRoutes)`），确保多级详情路由能被一级 Layout 的 `<router-view />` 正常渲染，杜绝白屏与未挂载异常。
  - **右侧全下级纯平铺工作区 (`flex-1 min-w-0`)**：
    - **全下级纯平铺算法 (Pure Flat Descendants Array)**：当在左侧选中任意节点时，右侧展示该节点本身及其全部下级（包括子级、孙级及更深层级所有节点）；
    - **彻底根治 `+` 号展开折叠列 (Anti-Plus Icon SOP)**：Ant Design Vue `<a-table>` 只要在数据行中检测到 `children` 键就会自动判定为树形表格并强行渲染 `+` 折叠展开列。在递归打平算法中**必须使用解构彻底剥离 `children` 属性**（`const { children, ...pureItem } = node`），传入表格的数据源为 100% 纯一维数组；
    - **高密度双行复合列（Taste-Skill `VISUAL_DENSITY=8`）**：「菜单信息」列复合展示标题、类型胶囊（目录/菜单/内页）与副行路由名 `name` 及 `icon`；「路由与组件」列复合展示 `path` 与副行 `component`；配合 `Zero-Page-Scroll` 内部滚动消除外层滚动条；
  - **新增智能继承与行级操作**：若在左侧选中了某模块，点击「新增菜单」自动将该模块锁定为上级；行级提供「查看」（只读）、「+子项」、「编辑」、「删除」；
- **行内状态快速微开关与权限码一键复制**：
  - 状态列采用行内 `<a-switch>` 极速启停（提交时剥离 `parentTitle` 等前端派生字段，并同步写回 `fullMenuTree` 源节点）；
  - 权限标识 (`permissionCode`) 采用等宽字体微底标签，支持悬浮一键写入剪贴板；
- **详情抽屉统一 `<BaseRbacDrawer>` 与防成环守卫 (`menuInfoDetail`)**：
  - 统一使用 `<BaseRbacDrawer>`（取代旧 `<a-modal>` 及废弃的 `subMenuManager`），支持新增、编辑、`readonly` 只读查看三合一；
  - 直接复用父组件传入的 `treeData`（`fullMenuTree`），并在编辑模式下递归将自身及所有子孙节点置为 `disabled: true`，前后端双重阻断父级成环；
  - 排序字段采用 `<a-input-number>` 强类型约束，布尔开关收敛为单层水平控制条（消除多层嵌套边框卡片）。

### 3.10 财务流水与企业级数据表格 SaaS 范式 (Finance Taste-Skill SOP, 2026-10-27)
- **双行高密度复合列架构 (Taste-Skill `VISUAL_DENSITY=8`)**：
  - 将「账目名称」、「状态」、「归属成员」与「业务时间」高度收敛为单一主复合列（`compound-name-cell`）：主行渲染加粗账目名称与 7px 状态呼吸微圆点（有效绿点 `bg-emerald-500` 带双环光晕 / 失效灰点 `bg-slate-300`），副行渲染成员轻量徽章与等宽业务时间；
  - 彻底剔除单独占列的「收支类型」与「状态」文本列，腾挪出宝贵的横向视口空间，确保在大中小各分辨率下一屏通透、无横向滚动条；
- **金融金额靠右对齐与红绿自解释规范**：
  - 金额列统一声明 `align: 'right'`，数字应用 `font-mono tabular-nums text-[15px] font-bold tracking-tight`，确保垂直正负号、数字与分位符绝对垂直对齐；
  - 收入渲染翡翠绿（`text-emerald-600`，带 `+¥` 前缀），支出渲染玫瑰红（`text-rose-600`，带 `-¥` 前缀），通过色彩和正负号直接传达业务语义；
- **支付渠道编码暗号转译与品牌微胶囊**：
  - 严禁向终端用户直接暴露 `wx`、`mm`、`yhk`、`dfw` 等开发代号；
  - 前端建立「静态词典 + 后端动态字典 + 常见业务前缀降级」三级转译体系（`hfw` 自动转译为「话费 (我)」，`dfm` 自动转译为「电费 (妈)」）；
  - 支付方式统一渲染为带有品牌矢量微图标的圆角微胶囊（`.from-source-pill`）；
- **响应式双子 SaaS 概览面板 (Twin Metric Cards Grid)**：
  - 告别单行死板拥挤的硬分割卡片，升级为响应式 CSS Grid 双子独立指标卡（`.finance-overview-grid`）；
  - 左卡「账目收支核算盘」与右卡「当月零花钱预算盘」各自自闭环，卡片统一挂载 `rounded-xl border border-slate-200/80 shadow-xs`，核心数字放大至 18px 等宽粗体；
- **筛选栏 Segmented 滑块与通透 Pills 胶囊规范**：
  - 收支类型快捷切换弃用传统方形厚重 Radio，升级为 Tailwind 浅灰底 Segmented 分段器（滑块激活态为白底带微投影 `shadow-xs` 与品牌蓝字）；
  - 周期快捷切换采用轻量微圆角 Pills，提升日常记账的灵敏与透气感。

### 3.11 KPI 指标卡微图表渲染范式 (Metric Card ECharts Sparkline)
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
- **图标组件显式导入红线**：
  - `unplugin-vue-components` 仅对 Ant Design Vue 原生基础组件（`<a-*>`）执行自动解析；
  - 所有来自 `@ant-design/icons-vue` 的图标组件（如 `TeamOutlined`、`ReloadOutlined`、`ExclamationCircleOutlined` 等）**严禁遗漏显式导入**，必须在使用的各 SFC / 子组件中通过 `import { ... } from '@ant-design/icons-vue'` 显式引入，彻底杜绝运行时 `[Vue warn]: Failed to resolve component` 警告。

---

## 8. 代码质量与安全规约 (Code Quality & Security Standard)

- **SonarQube S2245 伪随机数发生器规范**：
  - 严禁在有安全要求、敏感操作、或易触发静态质量门禁的业务场景中滥用 `Math.random()`（会触发 SonarQube `typescript:S2245` 安全热点告警）；
  - 统一优先采用现代浏览器原生标准的 Web Crypto API（`window.crypto.getRandomValues`）生成加密安全的随机数；
  - 即使对于登录页角色眨眼等前端动画或延迟场景，如需使用随机数，优先封装 `window.crypto.getRandomValues` 生成 `[0, 1)` 区间随机数，从源头彻底杜绝安全告警与静态扫描阻断。
- **SonarQube S7758 字符码点处理规范 (`codePointAt` vs `charCodeAt`)**：
  - 严禁在字符串哈希计算、字符编码解析中调用旧版 `String.prototype.charCodeAt()`（会触发 SonarQube `typescript:S7758` 告警）；
  - 统一优先使用 ES6 标准的 `String.prototype.codePointAt()` 或 `for...of` 字符迭代器；
  - 彻底保障对 4 字节 Emoji 表情（如账本分类图标 🍔、🚗、💰、🛍️）及 Unicode 扩展平面汉字的完整码点兼容，杜绝代理对截断导致哈希分布失衡或异常。

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

---

## 11. RBAC 与系统底座详情抽屉读写分离与无障碍 (A11y) 亲和性范式 (Drawer Read/Write Separation & A11y)

- **只读模式反模式根治 (Banish Disabled Form Anti-pattern)**：
  - **严禁只读模式下全量展示灰置 Disabled 表单**：在查看模式下，直接展示灰底表单控件会导致视觉杂乱、可读性极差且传递了“可交互”的假象；
  - **全量覆盖五大系统详情抽屉 (`userManagerDetail` / `orgInfoDetail` / `roleInfoDetail` / `permissionInfoDetail` / `menuInfoDetail` / `dict-detail`)**：
    - **Hero Profile 卡片**：顶部聚合专属领域图标/头像、启用/有效状态徽章、大字主名称与等宽编码（Code），建立清晰的第一视觉焦点；
    - **结构化拓扑与归属卡片**：将 `parentId`、`orgIds`、`roleIds` 等关联外键通过树或选项字典解析为可读中文名称徽章（如上级机构、上级菜单、上级权限、角色管辖机构 Tag），**彻底杜绝雪花算法原始大数裸露**；
    - **权限树与键值映射卡片**：角色查看模式提供只读权限树预览，字典查看模式提供类别与分类编码键值网格；
    - **职责与简介卡片**：独立轻质卡片，多行排版呈现职责描述或备注说明，无内容时提示斜体占位文案。
- **表单编辑模式的 A11y 亲和性规范 (Label-for Matching)**：
  - 在 `<a-form-item name="xxx" label="yyy">` 下的输入控件（`<a-input>`、`<a-input-number>`、`<a-select>`、`<a-tree-select>`、`<a-date-picker>`、`<a-radio-group>`、`<a-textarea>` 等），必须显式补齐 `id="form_xxx"` 与 `name="xxx"`；
  - 严禁缺失控件 ID 导致浏览器 DevTools Issues 面板提示 `Incorrect use of <label for=FORM_ELEMENT>` 无障碍告警。
- **图标显式导入红线**：
  - 组件内引用的所有 Ant Design 图标（如 `BankOutlined`、`SafetyCertificateOutlined`、`TeamOutlined`、`UserOutlined`、`BookOutlined` 等），必须在 `<script setup>` 中显式通过 `import { ... } from '@ant-design/icons-vue'` 导入，杜绝未导入直接在 template 中使用导致的运行时解析失败。

---

## 12. 系统管理与底座页面现代化 SaaS 美学升级范式 (Modern RBAC & Dict Taste-Skills Standard)

- **容器与背景底色架构**：
  - 最外层容器必须统一采用 `min-h-full p-4 lg:p-6 bg-slate-50/60 dark:bg-slate-950 transition-colors`，取代生硬刺眼的旧纯白全屏背景；
  - 核心卡片容器统一采用语义化变量与圆角阴影：`bg-[var(--card-bg)] border border-slate-200/90 dark:border-slate-800 rounded-2xl shadow-xs transition-colors`，兼顾浅色清新与暗色深邃质感。
- **标准化 Screen-Header 布局**：
  - **左侧品牌指示**：配备 `w-2.5 h-6 rounded-full bg-blue-600 flex-shrink-0 shadow-xs` 品牌亮蓝指示条，配合 `text-xl font-extrabold tracking-tight text-slate-800 dark:text-slate-100` 大标题与 `text-xs text-slate-500 dark:text-slate-400 font-normal` 业务职责副标题；
  - **右侧指标与范围胶囊**：配备轻质统计指示器（如权限总数、角色总数、机构总数、用户总数、字典总数徽标胶囊）与 `scopeHintText` 数据权限范围胶囊（`data-testid="rbac-data-scope-hint"`），强化多租户/机构边界感。
- **紧凑单行高效搜索栏 (`filter-card`)**：
  - 搜索栏采用独立卡片隔离，支持回车触发表单检索（`@press-enter="() => query(true)"`）；
  - 配备主要按钮（`<SearchOutlined />` 查找）与次要操作（`<ReloadOutlined />` 清空重置并自动触发重查），按钮圆角收敛为 `rounded-lg px-4`。
- **一体化现代表格卡片 (`table-card`) 与样式穿透**：
  - 卡片顶部集约化承载 `table-toolbar`（操作动作区 + 提示说明区），下辖表格主体；
  - AntD 表格通过 scoped class（如 `.modern-permission-table`、`.modern-role-table`、`.modern-org-table`、`.modern-user-table`、`.modern-dict-table`）穿透定制表头浅蓝灰底色（`#f1f5f9` / 暗黑模式 `#0f172a`）与圆润内边距，去除生硬分割线；
  - **信息密度与微交互**：
    - 用户复合信息：头像 + 粗体昵称 + 科技蓝等宽账号药丸，配备 `<CopyOutlined />` 点击一键复制剪贴板；
    - 编码字段（角色编码、权限标识、机构编码、字典类别编码 `typeCode`、字典分类编码 `belongTo`）支持一键复制微图标与交互反馈；
    - 机构拓扑与层级：机构名称配备专属组织图标徽章与多行简介，顶级节点自动标识「根机构」胶囊；所属机构展示「主」机构与「兼」机构微胶囊；
    - 状态列：状态开关采用 `<a-switch>` 极速启停与局部防抖，普通状态统一采用带呼吸灯小圆点（`w-1.5 h-1.5 rounded-full bg-emerald-500 animate-pulse`）的轻色药丸胶囊；
    - 操作列统一采用微色轻质胶囊链接按钮（查看/编辑/授权/成员/删除），并配齐标准语义图标，杜绝单调下划线链接。
- **关系穿梭调配页面范式 (Relation Transfer Screens - `orgUserInfo` / `roleUserInfo`)**：
  - 外层容器统一采用 `min-h-full p-4 lg:p-6 bg-slate-50/60 dark:bg-slate-950 transition-colors flex flex-col gap-5`；
  - 顶部操作卡片配备品牌指示条、模块大标题、业务副标题/当前适用范围胶囊与目标选择器（支持搜索与快速返回）；
  - 主体穿梭卡片承载 `rbac-dual-list-selector` 与 `rbac-diff-preview` 变更差异面板，底部提供清爽的操作确认栏。

---

## 13. 个人财务模块现代化工作台与凭证抽屉范式 (Self-Finance Modern SaaS Standard)

- **日常记账工作台内聚布局 (`financeManager`)**：
  - 淘汰外层游离的 `.button` 工具条，将「记一笔 (`data-testid="btn-add-finance"`)」与「批量删除」内聚至 [`FinanceManagerTable.vue`](file:///f:/workplace/project/myself/frontend/alex_miaosha_front/src/views/selfFinance/financeManager/components/FinanceManagerTable.vue) 的 `.table-card-header` 右侧插槽（`#headerActions`），释放纵向视口空间；
  - 操作列为所有账单提供「查看 (`data-testid="btn-view-finance"`)」入口，结合属主权限控制「编辑 (`data-testid="btn-edit-finance"`)」与「删除 (`data-testid="btn-delete-finance"`)」；
  - **记账明细抽屉 (`finance-manager-detail`)**：采用 `BaseRbacDrawer (560px)`，只读模式下渲染「账单电子凭证卡片」（收支金额 Hero 卡片 + 分类/支付渠道图标/归属成员/业务时间 4 宫格），新增模式下通过 `#footer` 插槽保留「保存并再记一笔」高频记账能力。
- **个人账户与卡券记录 (`accountRecordInfo`)**：
  - 统一 `AccountRecordInfo.id` 为 `string` 类型，采用 `screen-header`（含本页有效金额合计与待过期/已过期预警胶囊）+ `filter-card` + `table-card` 三段式布局；
  - 详情抽屉 (`accountRecordInfoDetail`) 采用 `BaseRbacDrawer (560px)`，实现账户资产凭证只读卡片与垂直 A11y 表单读写分离，并确保 `<a-form-item name="account">` 校验完整闭环。
- **消费储值卡工作台 (`prepaidCardInfoT` & `prepaidConsumeRecordT`)**：
  - 储值卡资产采用响应式网格卡片墙（`.wallet-cards-grid`），支持点击卡片联动穿透下方交易流水表（`PrepaidConsumeRecordTVo` 列契约对齐：`cardName`、`type`、`amount`、`balanceAfter`、`orderNo`、`description`、`consumeTime`）；
  - 充值/消费抽屉 (`recharge-info-modal`) 根据 `type` (`'recharge' | 'consume'`) 动态切换标题、金额标签与成功提示，严禁硬编码单一「充值」文案；
  - 全量消除 `.then(res => res.code)` 链式点读取，统一使用 `const { code, data, message } = await api()` 解构。

---

## 14. 消费券中心现代 SaaS 架构与双抽屉范式 (Coupon Center Modern SaaS Standard)

- **三段式消费券资产台账 (`cpn-coupon-info` / `cpn-user-coupon-info` / `cpn-redemption-record-info`)**：
  - 全量淘汰旧版 `.page-info`，统一升级为 `screen-header`（含券种数、可用余量合计、核销统计胶囊）+ 单行紧凑 `filter-card` + `table-card`；
  - 所有主键与外键（`id`、`userId`、`couponId`、`userCouponId`、`orderId`、`merchantId`）严格定义为 `string` 类型，杜绝雪花 ID 精度丢失；
  - **防抖监听与挂载请求去重规范**：列表页在 `onMounted` 中显式触发首次查询时，`watch(() => searchInfo.value, ...)` 严禁开启 `immediate: true`，并在 `onUnmounted` / `onBeforeUnmount` 中调用 `debouncedFn.cancel()` 释放防抖定时器。
- **消费券详情抽屉与核销抽屉 (`cpn-coupon-info-detail` & `cpn-coupon-redeem-quantity-detail`)**：
  - `cpn-coupon-info-detail` 采用 `BaseRbacDrawer (560px)`，支持 `v-if="isReadonly"` 电子票面只读卡片（面值、门槛、发行/剩余库存、起止有效期、支付状态），并在新增模式下完整保留 **✨ AI 营销策划助手**（`data-testid="ai-plan-bar"` 等全部测试锚点）；
  - `cpn-coupon-redeem-quantity-detail` 采用 `BaseRbacDrawer (480px)`，通过 `defineModel<ModelInfo>('modelInfo')` 兼容父组件 `v-model:modelInfo` 绑定，实时展示核销前后余量联动预览与步进器数量控制。

---

## 15. 商品中心现代 SaaS 架构与价格监测抽屉范式 (Product Center Modern SaaS Standard)

- **商城商品与秒杀池管理 (`pmsShopProduct`)**：
  - 统一采用 `shop-product-screen` 三段式架构（`screen-header` + `filter-card` + `table-card`），顶栏保留 **✨ AI 秒杀营销文案生成**（`data-testid="btn-open-ai-copy"`），弹窗 `ProductAiCopyModal` 完整保留 `ai-copy-input-name`、`btn-generate-ai-copy`、`btn-apply-ai-copy` 锚点并补齐 A11y `id/name`；
  - 价格列实时比对 `price` 与 `comparePrice`：低于对比参考价时自动高亮红色降价胶囊（`.price-drop-chip`）并展示原价删除线；
  - **商品详情与价格监测抽屉 (`pmsShopProductDetail`)**：淘汰原 `width="3000px"` 越界 Modal 与 `<a-form disabled>` 纯静态只读缺陷，升级为 `BaseRbacDrawer (680px)` 读写分离架构：
    - **只读模式 (`v-if="isReadonly"`)**：渲染「商品核心档案卡」（缩略图、来源平台胶囊、店铺名、SKU 编号、商城外链）+「4 宫格价格监测指标卡」（当前售价、对比价、历史最低价、历史最高价）+「全宽 100% 近三十日价格波动走势图」；
    - **新增/编辑模式 (`v-else`)**：提供完整可录入的双列垂直 A11y 表单，编辑已有 SKU 时在表单下方同步渲染近三十日价格曲线。
- **商品属性与规格字典管理 (`pmsAttr`)**：
  - 统一收敛 `pmsAttr/config/index.ts`、`pmsAttrListTs.ts` 与 `pmsAttrDetailTs.ts` 类型定义，将 `id` 与 `catelogId` 严格修正为雪花 ID 安全的 `string` 类型；
  - 淘汰数据库注释式的超长表头与手填 `0/1/2` 数字文本框，抽取 `ATTR_TYPE_OPTIONS`、`ENABLE_STATUS_OPTIONS`、`SEARCH_TYPE_OPTIONS`、`SHOW_DESC_OPTIONS` 语义化字典常量，在筛选栏、表格胶囊与 `BaseRbacDrawer (560px)` 读写分离抽屉中统一使用下拉选择与状态徽标。

---

## 16. 店铺进销存与电商财务全模块现代 SaaS 范式 (Shop ERP & Finance Modern SaaS Standard)

- **全仓 `.page-info` 历史包袱 100% 清零**：
  - `shopFinance`、`shopFinanceAnalysis`、`shopOrder`、`shopCart`、`shopStock`、`shopStockBatch`、`shopStockAttrs` 七大子模块全面升级为 `screen-header` + `filter-card` + `table-card` 现代 SaaS 三段式架构；
  - 全部 6 个详情弹窗统一升级为 [`BaseRbacDrawer`](file:///f:/workplace/project/myself/frontend/alex_miaosha_front/src/components/BaseRbacDrawer/index.vue)，实现 `v-if="isReadonly"` 结构化电子凭证卡片与双列 A11y 表单读写分离。
- **关键契约与数据安全护栏**：
  - **表单模型双向绑定隔离**：严禁不同 `<a-form-item>` 共享同一 `formState` 属性（已根治 `shopFinanceDetail` 中 `payWay` 误绑为 `formState.isValid` 的缺陷）；
  - **多字典联合预加载**：当页面同时依赖有效状态、商品分类与进货地点字典时，必须在 `useDictInfo('is_valid', 'shop_category', 'stock_place')` 中一次性声明预加载，并保持列表页与详情页字典编码 100% 一致；
  - **雪花 ID 全量 `string` 化**：`ShopCartData`（`id`、`shopId`、`userId`、`customerId`）、`ShopStockData`（`id`）、`ShopStockAttrsData`（`id`、`stockId`）、`ShopOrderData`（`id`）、`shopAnalysis.resultType`（`userId`）一律定义为 `string` 类型。
- **商店财务经营分析大盘 (`shopFinanceAnalysis`)**：
  - 顶栏右侧内聚月份选择器（`picker="month"`）与刷新控制；
  - 顶部配备 **4 宫格动态经营 KPI 卡片**（当月商品累计营收、当月累计销售件数、热销榜首商品、主力收款渠道），下方采用 `2×2` 响应式图表网格面板（`.analysis-charts-grid`）承载商品营收占比、支付渠道分布及日/月度销售走势。

---

## 17. 垂直业务领域测试金字塔与契约锁规范 (Testing Pyramid & Contract Lock Standard)

- **测试清单分层落地 (`tests/checklists/`)**：
  - 所有新增或完成现代 SaaS 重构的垂直业务模块（商品中心、消费券、个人财务、店铺进销存），必须在 `tests/checklists/` 沉淀标准化 Checklist（如 `shop-erp-product.md`、`gift.md`、`message.md`）；
  - 清单必须严格覆盖 **0. 元信息**、**1. 核心高价值测试目标与真实代码漏洞防线**、**2. 字段边界（七点法）**、**3. 状态机流转矩阵**、**4. 数据权限矩阵** 与 **5. 不测理由清单**。
- **Vitest 纯 TS 单元测试护城河 (`tests/unit/`)**：
  - **Node/Vitest 运行环境约束**：单元测试严禁级联引入带有未编译模板的 `.vue` 文件或 `@/utils/request`（防 Axios/路由级联加载报错），业务配置与核心算法一律抽取为纯 TypeScript 模块；
  - **大数雪花 ID（string）防御锁**：各垂直模块必须通过测试用例（如 `productCenter.test.ts`、`couponCenter.test.ts`、`shopErp.test.ts`）锁住 `normalize*Ids` 的递归转换行为，保证 `1847192837192837192n` 等大数雪花 ID 在任何数据流向中均稳定表现为 `string` 类型；
  - **业务核心算力锁**：
    - 店铺收支净额：`calcFinanceNetBalance(income, expense) = round((income - expense) * 100) / 100`；
    - 消费券余量与临期判定：`calcCouponRemainingRatio(remainingQuantity, totalQuantity)` 与 `isCouponExpiringSoon(expireRangeStatus)`。
- **L3 端到端 AI 自动化烟测标准 (`scripts/playwright/`)**：
  - **DOM 挂载原则**：所有可交互、有断言诉求的 DOM 必须挂载语义化 `data-testid`（如 `btn-view-product`、`btn-redeem-coupon`、`shop-finance-voucher`、`shop-analysis-kpi-grid`），严禁使用中文文本进行模糊选择；
  - **Persona 角色切换与状态重载**：执行权限矩阵切换时（`switchPersona`），必须显式调用 `await page.reload()` 确保 Pinia `userStore` 完全重置，并等待 `networkidle` 彻底消除路由残留影响；
---

## 18. TypeScript 严格类型安全与通用弹窗契约 (TypeScript Strict Type Safety & ModelInfo Standard)

- **`ModelInfo` 扩展契约**：
  - 通用模态框与抽屉配置接口 [`ModelInfo`](file:///f:/workplace/project/myself/frontend/alex_miaosha_front/src/types/modal.ts) 必须内置 `extraData?: any` 通用业务扩展载荷属性；
  - 允许在列表页与详情抽屉之间安全传递富结构体（如 AI 解析账单结果、复制模板数据、关联父级对象等），杜绝组件间传参导致的 `Property 'extraData' does not exist on type 'ModelInfo'` (TS2339) 错误。
- **`vue-tsc --noEmit` 0 报错工程防线**：
  - **网络请求泛型与配置一致性**：`putData` 与 `postData` 接口统一使用 `config?: Record<string, unknown>` 签名，参数透传时避免强转为已废弃未导出的 `AxiosRequestConfig`；
  - **选项集合类型兼容**：提供给 `a-select` 的常量数组统一绑定 `DefaultOptionType[]` 兼容类型（如 `ATTR_TYPE_OPTIONS: (DefaultOptionType & { color?: string })[]`），避免 `as const` 产生的只读深度不可变修饰符与 UI 组件入参类型冲突 (TS4104)；
  - **抽屉状态非空与可选兼容**：[`BaseRbacDrawer`](file:///f:/workplace/project/myself/frontend/alex_miaosha_front/src/components/rbac/BaseRbacDrawer.vue) 的 `open` 属性统一定义为 `open?: boolean` 并赋予默认值 `false`，完美兼容 `modelInfo.open` 为 `undefined` 时的双向数据绑定 (TS2322)；
  - **数据解构与空值防御**：API 返回数组前必须使用 `Array.isArray(data) ? data : []` 进行类型收敛，避免非法数据引发 `records.map` 崩溃。




---

## 19. 全局 AI 智能体抽屉 (Copilot Drawer) 与流式打字机交互规范 (Global AI Copilot Drawer Standard)

- **全局入口与轻量级解耦 (GlobalAiAssistantDrawer.vue)**：
  - 全局 AI 浮动触发器（data-testid="btn-open-global-ai-drawer"）固定停靠在右下角（z-index: 998），点击滑出 520px 宽度的独立全局抽屉（data-testid="drawer-global-ai"）；
  - **请求与依赖彻底隔离**：流式请求方法 chatAiStream 与普通请求 chatAi 统一定义在 @/api/ai，禁止在此处级联导入 Pinia 用户状态（@/store/modules/user/user）与路由（@/router），以防 Node/Vitest 环境下解析 Vue 语法崩溃。Token 解析统一采用本地缓存快照直接读取（resolveToken()）；
  - **SSE 长连接流式打字机**：
    - 前端基于原生 fetch 与 ReadableStreamDefaultReader 逐帧解析 'text/event-stream'；
    - 支持后端双阶段事件解析：首帧元数据解析（event: meta / model）、内容逐字流式打字（event: chunk / data:）与完成标记（[DONE]）；
    - 支持 Markdown 渲染与呼吸光标动效（.typing-cursor），输出过程中禁用重复提交，遇到网络中断或用户中止支持友好降级展示。
---

## 20. 传统纸质台账高保真打印预览与 @media print 打印穿透规约 (Gift Registry Print Standard)

- **业务背景与场景需求**：
  - 办宴事件（婚礼、满月、寿宴、乔迁）具有极强的“实物礼簿”留存习惯，需要一份排版古朴端正、清晰易核对的纸质礼簿供家庭审阅与对账留存；
- **组件实现规范 (GiftRegistryPrintModal.vue)**：
  - **双重排版风格无缝切换**：
    1. **传统中式礼簿 (traditional)**：采用宣纸淡黄底纹（#fffdf8）、朱红回纹边框（#b91c1c）、典雅书法字体（KaiTi）与仿古印章（“吉祥如意”），专为长辈传阅与传统喜寿办宴设计；
    2. **现代清晰对账 (modern)**：采用白底黑字、紧凑网格与高对比度线条，极度省墨且适合严谨对账；
  - **金额双重展示与中文大写算法 (registryPrint.ts)**：
    - 提供精准的中文大写转换函数 toChineseCurrency(num)，符合银行票据填写规约；
    - 支持智能大额礼金标红高亮（默认 $\ge ¥1000$ 醒目显示，large-amount-row）；
- **原生打印穿透与分页优化 (@media print)**：
  - **无死角遮罩隐藏**：通过 @media print 强制声明 ody * { visibility: hidden; }，仅将 .registry-paper-container 及其子节点设为 isibility: visible，并用 position: absolute; left: 0; top: 0; width: 100% 铺满打印页面；
  - **防跨页腰斩裁切**：表格行强制配置 	r { page-break-inside: avoid; page-break-after: auto; }，避免长名单在翻页处被半行截断；
  - **打印尺寸精确设定**：通过 @page { size: A4 portrait; margin: 12mm 15mm; } 约束 A4 竖版打印边距，自动隐藏弹窗头尾与操作按钮（.no-print）。


---

## 9. 品牌标识与全端 Logo 规范 (`alex-management` AM Twin Peaks - Premium Neo-Glow Edition) (2026-10-26)

- **视觉图形与寓意 (Premium Neo-Glow Edition)**：
  - 采用一笔连成的字母 **A + M**（`Alex Management`），双峰并肩寓意礼尚往来与上扬的收支曲线；独立悬浮在右峰上方的黄金币（`#FDE047` ➔ `#D97706` 双色渐变）代表财务记账与资产沉淀，右上角增强型四芒星代表 AI 智能记账与分析；底色采用三色深邃流光底板（`#0D9488` ➔ `#2563EB` ➔ `#4F46E5`）+ 顶部 1px 晶体高光描边与柔和发光投影。
- **品牌标准色板**：
  - 核心科技流光：`#0D9488` (Teal-600) / `#2563EB` (Blue-600) / `#4F46E5` (Indigo-600)
  - 点缀资产暖金：`#FDE047` ➔ `#D97706`
  - 辅助人情温度：`#FF7A6B`
  - 标题深石板：`#1E293B`
- **全端工程挂载规范（严禁裸露文本 div）**：
  - 矢量图标统一存放于 `public/logo.svg`（`viewBox="0 0 100 100"`），主屏幕高清图标存放于 `public/apple-touch-icon.png`（`192×192`）；
  - `index.html` 统一声明 `<link rel="icon" type="image/svg+xml" href="/logo.svg" />` 优先加载矢量图标；
  - **PC 端侧栏与登录页红线**：侧边栏（`src/layout/index.vue`）必须使用 `<img src="/logo.svg" alt="Alex Platform" class="app-logo-img" />`（`34×34px`），登录页（`src/views/login/index.vue`）必须使用 `<img src="/logo.svg" alt="Alex Platform" class="login-logo-img" />`（`44×44px`），严禁使用 `<div class="login-logo">A</div>` 伪图标文字占位。

---

## 10. 核心依赖升级基准版本号 (Vue + TS + Vite + AntD + Pinia) (2026-10-26)

- **基准依赖版本矩阵**：
  - `vue`: `^3.5.44`（最新响应式核心与 Bug 修复）；
  - `vite`: `^8.3.5` + `@vitejs/plugin-vue`: `^6.0.10`（构建与预热优化）；
  - `pinia`: `^4.1.0`（与 `pinia-plugin-persistedstate` 4.x 深度协同）；
  - `typescript`: `^6.0.3`（与 `vue-tsc` 保持最高稳定兼容）；
  - `ant-design-vue`: `^4.2.6`（4.x 正式最新稳定版）。
- **质量检验全绿战报**：
  - PC 端 `npm run type-check`（`vue-tsc` 类型检查）0 错误通过；
  - PC 端 23 个套件 112 项 Vitest + 7 个套件 21 项 Node 原生测试，**合计 133 项单测 100% 绿灯**；
  - 生产构建 `npm run build` 耗时 ~17.6s 顺利打出 gzip 与 brotli 产物。
