---
title: PC 端开发规则与组件范式
tags: [standards, frontend, pc, vue3, antd, pinia, midscene]
aliases: [PC端规则, PC组件范式, PC开发规范]
created: 2026-09-16
updated: 2026-09-16
status: active
---

# 💻 PC 端开发规则与组件范式 (`alex_miaosha_front`)

返回导航：[[Home]] | [[00-System/全栈系统架构总览与交互流|系统全景]]

---

## 1. 技术栈与架构基准

- **核心框架**：Vue 3 (Composition API) + TypeScript + Vite；
- **包管理工具**：`pnpm`（CAS 内容寻址硬链接存储，全局 `pnpm-lock.yaml` 约束）；
- **UI 组件库**：Ant Design Vue v4 (`ant-design-vue`)；
- **全局状态**：Pinia + `pinia-plugin-persistedstate`；
- **自动化插件**：`unplugin-auto-import` 与 `unplugin-vue-components`（常用 API 如 `ref`, `computed`, `watch` 及 AntD 组件自动解析导入）。

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

---

## 3. 组件开发与代码组织范式

### 3.1 类型与配置统一管理
- 导入 Pinia 状态或 Vue 组件中的用户、角色、机构相关数据类型时，**必须统一在对应页面配置文件夹中导入**（例如 `@/views/user/roleInfo/config`、`@/views/user/menuInfo/config`），严禁在单页面内重复冗余定义相同的 interface。

### 3.2 严禁重复显式 import
- 对于已由 Vite 插件自动注册的 API（`ref`, `computed`, `watch`, `useRoute`, `useRouter`）与 Ant Design Vue 组件，**严禁在 `.vue` 文件中手动重复 `import`**。

### 3.3 SVG 图标引用范式
- 严禁在模板中使用已废弃的旧 `<MySvgIcon>` 标签；
- 本地 SVG 图标统一通过 `unplugin-icons` 机制（如 `~icons/my-menu-svg/*`、`~icons/my-finance-svg/*`、`~icons/my-soft-svg/*`）按需引入，或使用 `@/views/common/config` 下的 `iconComponentMap` 结合 `<component :is="iconComponentMap[key]" />` 动态组件机制渲染。

---

## 4. AI 与 E2E 自动化测试体系 (Midscene + Playwright)

- **技术组合**：`@midscene/web` + `@playwright/test`；
- **可交互 DOM 规约**：所有可交互元素必须挂载 `data-testid`，禁止用中文文案做精确匹配；
- **等待策略**：严禁使用 `Thread.sleep` 或 `waitForTimeout`，统一采用 `waitForResponse` / `waitForSelector` / `aiWaitFor`；
- **数据清理护城河**：测试用例产生的临时数据必须在 `try...finally` 中调用清理 API 删除，禁止残留脏数据污染开发数据库。
