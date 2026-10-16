---
title: TechSpec - RBAC 组织与动态数据权限架构
aliases: [RBAC架构, 数据权限实现]
created: 2026-09-16
updated: 2026-09-16
status: active
tags: [techspec, rbac, backend, mybatis, security]
---
# 💻 TechSpec - RBAC 组织与动态数据权限架构

返回导航：[[Home]] | 功能描述：[[02-Features/02-RBAC-用户组织权限/PRD-用户体系与组织权限功能说明|PRD-用户体系与组织权限功能说明]]

---

## 1. 机构关系与角色聚合技术实现

- **机构绑定事务 (`assignSingleOrg`)**：
  ```java
  @Transactional(rollbackFor = Exception.class)
  public boolean assignSingleOrg(Long userId, Long orgId) {
      // 1. 将该用户原有生效记录失效
      update(Wrappers.<TOrgUserInfo>lambdaUpdate()
          .eq(TOrgUserInfo::getUserId, userId)
          .eq(TOrgUserInfo::getStatus, 1)
          .set(TOrgUserInfo::getStatus, 0));
      // 2. 插入新有效归属
      TOrgUserInfo newRel = new TOrgUserInfo();
      newRel.setUserId(userId);
      newRel.setOrgId(orgId);
      newRel.setStatus(1);
      return save(newRel);
  }
  ```
- **多角色权限聚合**：`UserPermissionContextService.buildContext()` 异步构建权限上下文。
- **权限上下文唯一真相源（Single Source of Truth）**：
  登录响应中统一由 `UserPermissionContextVo permissionContext` 承载机构、多角色、权限码与菜单树。`TUserVo` 顶层平铺的历史镜像字段（`orgInfoVo`, `roleInfoVoList`, `menuInfoVoList`, `permissionCodes`, `buttonPermissionCodes`）通过 `@JsonIgnore` 与 `@JSONField(serialize = false)` 停止向网络传输层序列化冗余数据，Java 内部调用通过 Getter 委托无缝回源 `permissionContext`，实现前后端瘦身与契约完全对齐。

---

## 2. JSqlParser 动态 AST 注入与角色判定

基于 MyBatis-Plus 插件，在 SQL 执行前通过 JSqlParser 动态解析语法树注入 WHERE 条件：
1. **防死循环重入保护**：使用 `ThreadLocal<Boolean>` 标志位防止解析过程中触发 `getLoginUser()` 造成无限递归；
2. **反射热点缓存**：使用 `ConcurrentHashMap` 缓存 Mapper 方法上的 `@DataPermission` 注解元数据；
3. **角色分级与管理员判定标准 (`RbacRoleCodes`)**：
   - **超级管理员 (`isSuperRole`)**：精确匹配 `super_super`，全局放行，不受任何数据隔离限制；
   - **机构与业务管理员 (`isAdminRole`)**：匹配 `admin` 或以 `*_admin` 结尾的垂直业务管理员（如 `org_user_admin`、`family_admin`、`gift_admin`、`shop_admin` 等），享有当前机构及子孙机构的数据作用域（`WHERE id IN (SELECT user_id FROM alex_user.t_org_user_info WHERE org_id IN (...))`）；
   - **普通用户与业务角色 (`isUserRole`)**：匹配 `user` 或以 `*_user` 结尾的业务角色，默认回退为个人数据权限（`WHERE id = #{loginUserId}`）。

---

## 3. 机构管理服务与写路径越权防护 (2026-10 极简重构)

- **顶级机构创建约束**：非超级管理员禁止创建或修改根机构（`parentId` 为 null 或 0），防范机构管理员脱离组织体系独立建根。
- **挂靠父级数据范围校验**：新增与修改机构时，若指定了父级机构，必须经由 `assertOrgAccessible(parentId)` 检验该父级是否在当前操作者的机构数据作用域内，杜绝横向越权挂靠。
- **统一超管判断真值**：弃用业务服务内的私有判定逻辑，全局统一使用 `OrgScopeIdsResolver.isSuperAdmin(loginUser)`。
- **接口与前端瘦身**：
  - 后端下线未使用的 `GET /org-info/tree`，统一使用支持多维过滤的 `POST /org-info/tree`；
  - Mapper `getPage` 补齐稳定排序 `ORDER BY t_org_info.id ASC`，状态过滤由 `LIKE` 修正为精确匹配；
  - PC 前端下线冗余左侧树卡片与不准确的当前页 KPI 指标，去除深度 watch 防抖引发的双重请求；
  - **详情与编辑抽屉重构 (`orgInfoDetail`)**：支持新增/编辑/只读三合一，只读模式（`readonly: true`）展示完整审计条（`createTime` / `updateTime`）与自定义关闭；编辑时前端递归将当前节点及其所有子孙节点置为 `disabled: true`，彻底杜绝成环与自选；新建机构默认启用状态。
  - **行级机构成员抽屉 (`OrgUserDrawer`) 与解绑契约**：
    - 废除顶部工具栏突兀脱节的全局穿梭框跳转按钮，改为表格行内「成员」入口就近滑出抽屉；
    - 成员列表基于 `getUserManagerPage({ orgId })` 走纯服务端分页（10条/页），彻底消除前端全量 1000 条数据拉取与内存卡顿；
    - 后端补齐 `DELETE /api/v1/org-user-info/unassign`（`unassignUser`）：单用户锁保护，原子置为失效状态，清理失效历史行（保留最多 5 条）并主动失效 `permission_context` 缓存；
    - `TUserMapper.xml` 补齐 `t_org_user_info.status = 1` 有效态条件过滤，避免历史失效机构关系污染用户组织名称与分页查询。

---

## 4. 用户多机构绑定与列表详情重构 (2026-10 落地)

### 4.1 架构设计：主机构 + 兼任机构模式 (Main / Auxiliary)
- **业务语义**：
  - 用户拥有唯一**所属主机构**（`summary = 'MAIN'`），决定用户主要归属层级与基本组织信息；
  - 用户可兼任 0~N 个**兼任机构**（`summary = 'AUXILIARY'`），用于借调、兼岗或跨部门协作；
  - **零 DDL 数据库兼容**：复用 `t_org_user_info.summary varchar(200)` 列标识主兼类型，历史旧数据中 `summary IS NULL` 缺省视为 `'MAIN'`，无需执行破坏性 DDL 变更。
- **服务层编排与同步事务 (`OrgUserInfoService#syncUserOrgs`)**：
  - 单用户锁防止并发争抢：`synchronized (String.valueOf(userId).intern())`；
  - 全量差异比对（Diff）：比对现有 `status = 1` 的有效记录，对需新增项插入 `status = 1, summary = 'MAIN' / 'AUXILIARY'`，对移除项软删除置 `status = 0`，保留已有历史限制在合理水位；
  - 主动清理与失效：操作成功后立即调用 `permissionContextCacheService.invalidate(userId)`，强制下次查询重建权限上下文。
- **动态数据权限插件天然并集**：
  - `DataPermissionHandlerImpl` 在执行 `@DataPermission` 过滤时，通过 `WHERE org_id IN (SELECT org_id FROM alex_user.t_org_user_info WHERE user_id = ? AND status = 1)` 注入条件；
  - 主机构与兼职机构记录均以 `status = 1` 存放在 `t_org_user_info` 表中，AST 插件自然解析为所有有效机构的并集，兼任人员拥有各机构数据的合法访问权限。
- **聚合查询与前后端契约 (`TUserMapper.xml` & `TUserVo`)**：
  - `TUserVo` 扩展 `Long orgId`, `List<Long> auxiliaryOrgIds`, `String auxiliaryOrgNames`；
  - Mapper 层面通过聚合函数拆分：
    - 主机构：`MAX(CASE WHEN oui.summary = 'MAIN' OR oui.summary IS NULL THEN oi.org_name END) AS orgName`；
    - 兼职机构：`GROUP_CONCAT(DISTINCT CASE WHEN oui.summary = 'AUXILIARY' THEN oi.org_name END SEPARATOR ',') AS auxiliaryOrgNames`。

### 4.2 前端体验与页面交互重构
- **用户列表 (`userManager/index.vue`)**：
  - **紧凑单 KPI 徽标**：剔除当前页 10 条伪算出来的"已启用"虚假指标，只保留精准的"总用户数"；
  - **双重弹窗修复**：移除内层冗余 `Modal.confirm`，由外层 `<a-popconfirm>` 唯一处理确认；
  - **搜索性能优化**：移除对 `searchInfo` 的深度 `watch` 与 300ms 防抖，回归回车/查找按钮驱动；
  - **现代复合列渲染**：
    - 「用户信息」列复合呈现用户头像（无头像自动生成彩色文字徽章）、昵称与 `@username`；
    - 「所属机构」列高亮呈现主机构徽标（`主` 标签）与兼任机构胶囊徽标（`兼: xxx`）；
    - 增加「查看」只读入口（`<EyeOutlined />`），与编辑、删除操作并列。
- **详情抽屉 (`userManagerDetail/index.vue`)**：
  - 支持新增、编辑与只读（`isReadonly`）三合一；只读模式禁用所有表单输入并呈现简洁「关闭」按钮；
  - 所属机构升级为 `<a-tree-select>` 树形选择（单选主机构 + 多选兼任机构）；
  - **双向互斥防重**：计算属性动态将已被选为主机构的节点在兼任树中置为 `disabled: true`；反之兼任机构在主机构树中也被置为 `disabled: true`；切换主机构时自动清除兼任中冲突项；
  - 性别选择升级为药丸式单选按钮组（`a-radio-group button-style="solid"`）。

---

## 5. 菜单管理重构与防成环守卫 (2026-10 落地)

- **后端契约与防成环加固 (`alex_miaosha_user`)**：
  - `MenuInfoVo.parentId` 补充 `@JsonSerialize(using = Long2StringSerializer.class)`，彻底解决 19 位雪花 ID 父菜单传至前端时的 JS `number` 精度截断问题；
  - `MenuInfoServiceImp#updateMenuInfo` 增加 `assertNoParentCycle(id, parentId)` 父级防成环守卫（禁止将上级菜单设为自身或自身下级菜单），并统一复用 `OrgScopeIdsResolver.isSuperAdmin(loginUser)` 与 `assembleMenuTreeWithNullLeaf`（兼容 `parentId == null || parentId == 0L` 为根）；
  - `MenuInfoMapper.xml` 规范化 `order by COALESCE(t_menu_info.order_by, 999999) asc, t_menu_info.id asc` 移出 `<where>` 标签外，并将 `status`、`hideInMenu`、`showInHome` 修正为 `=` 精确匹配。
- **前端 Ponytail × Taste-Skill × Impeccable 重构 (`alex_miaosha_front`)**：
  - 下线零引用的废弃组件 `subMenuManager/index.vue` 及 `config/index.ts` 中未使用的 `labelMap`；
  - 列表表格由 11 列压缩为 8 列高密度复合列（「菜单信息」聚合标题/类型/路由名/图标，「路由与组件」聚合 `path` 与 `component`），启用 `Zero-Page-Scroll` 满屏内部自适应滚动，新增「查看」只读入口并修复批量删除双重弹窗及行内状态切换源树同步；
  - `menuInfoDetail/index.vue` 由 `<a-modal>` 统一升级为 `<BaseRbacDrawer>`，支持 `readonly` 只读查看模式，复用父组件 `fullMenuTree` 消除重复请求，并递归将自身及子孙节点设为 `disabled: true`。

---

## 6. 权限管理重构与防成环守卫 (2026-10 落地)

- **后端契约与防成环加固 (`alex_miaosha_user`)**：
  - `PermissionInfoVo.parentId` 补充 `@JsonSerialize(using = Long2StringSerializer.class)`，防止 19 位雪花 ID 在前端精度丢失；
  - `PermissionInfoServiceImp#updatePermissionInfo` 新增 `assertNoParentCycle(id, parentId)` 防成环校验，禁止将上级权限设为自身或自身子孙节点；`getList` 支持 `isRootParent(item.getParentId())`（兼容 `parentId == null || parentId == 0L` 为根），并删除私有重复方法 `isSuperAdminLogin`，统一复用 `OrgScopeIdsResolver.isSuperAdmin(loginUser)`；
  - `PermissionInfoMapper.xml` 补齐 `order by t_permission_info.id asc` 稳定排序，并将 `status` 过滤条件由 `like` 修正为 `=` 精确匹配。
- **前端 Ponytail × Taste-Skill × Impeccable 重构 (`alex_miaosha_front`)**：
  - **配置收敛**：合并并删除旧版散落的 `permissionInfoListTs.ts` 与 `permissionInfoDetailTs.ts`，统一收拢至 `src/views/user/permissionInfo/config/index.ts`，剔除冗余 `labelMap/labelCol/wrapperCol` 及 `summary/options` 的伪必填校验；
  - **抽屉与层级补齐 (`permissionInfoDetail/index.vue`)**：由 1000px 宽屏 `<a-modal>` 升级为 `<BaseRbacDrawer :width="560">`，支持新增/编辑/只读（`readonly`）三合一；补齐缺失的「上级权限 (`parentId`)」`<a-tree-select>` 树选择器（递归将自身及子孙节点置为 `disabled: true`）；修复 `savePermissionInfoManager` 未在 `finally` 中重置 `loading.value = false` 导致的保存按钮永久锁死 Bug；
  - **高密度列表 (`permissionInfo/index.vue`)**：落地 `Zero-Page-Scroll` 满屏自适应布局、「权限名称 + 描述」双行复合列、可点击一键复制的 `permissionCode` 科技蓝药丸、`options` 等宽代码标签、翠绿呼吸灯状态胶囊及单次 `<a-popconfirm>` 批量删除，移除 `watch + debounce` 引发的清空双重请求。

---

## 7. 角色管理重构与权限回显修复 (2026-10 落地)

- **后端契约与已选权限过滤加固 (`alex_miaosha_user`)**：
  - **已选权限回显过滤 (`RoleInfoServiceImp#queryRoleInfo`)**：查询 `rolePermissionInfoVoList` 时显式设置 `rolePermissionInfoVo.setStatus(SysConf.VALID_STATUS)`，并将 `RolePermissionInfoMapper.xml` 中 `role_id`、`permission_id`、`status` 由 `like` 修正为 `=` 精确匹配，杜绝失效历史权限绑定混入详情响应；
  - **列表绑定机构聚合 (`RoleInfoMapper.xml` & `RoleInfoVo`)**：`RoleInfoVo` 新增 `orgNames` 字段，`getPage` 增加关联 `t_role_org_info` 与 `t_org_info`（`status = '1' AND is_delete = 0`）的 `GROUP_CONCAT` 聚合子查询，补齐 `order by t_role_info.id asc` 并将 `status` 修正为 `=` 精确匹配；
  - **超管判定去重**：移除 `RoleInfoServiceImp` 内私有重复方法 `isSuperAdminLogin`，统一复用 `OrgScopeIdsResolver.isSuperAdmin(loginUser)`。
- **前端 Ponytail × Taste-Skill × Impeccable 重构 (`alex_miaosha_front`)**：
  - **权限树回显修复与组件收敛**：修复 `roleInfoDetail/index.vue` 与 `authorizationDetail/index.vue` 将已选权限映射为关系表主键 `item.id` 而非 `item.permissionId` 的 Bug（修正为 `String(item.permissionId || item.id || '')`）；下线全局唯一引用且含调试 `console.log` 的冗余组件 `src/components/menu-tree/index.vue`，统一收敛至支持 `disabled` 只读态与暗黑模式的 `<RbacPermissionTreePanel>`；
  - **三大抽屉统一 (`BaseRbacDrawer`)**：`roleInfoDetail`（支持新增/编辑/`readonly` 只读三合一）、`authorizationDetail`（AI 智能权限推荐 + 权限树）、`userAssignmentDetail`（角色成员分配穿梭框）全部升级为 `<BaseRbacDrawer>`；
  - **高密度列表 (`roleInfo/index.vue`)**：落地 `Zero-Page-Scroll` 满屏自适应布局、角色名称+描述复合列、可点击复制的 `roleCode` 药丸、绑定机构胶囊（`orgNames`）、可点击直达授权/成员抽屉的「`N` 权限 / `M` 用户」指标胶囊、呼吸灯状态胶囊及单次确认批量删除；
  - **双栏关系配置页与公共组件收官升级 (`orgUserInfo` / `roleUserInfo` / `RbacDualListSelector` / `RbacDiffPreview`)**：
    - `orgUserInfo/index.vue` 移除成员操作由硬删 `deleteOrgUserInfo` 修正为 `unassignOrgUser(userId, orgId)`（确保置为失效态并主动清理 Redis `permission_context` 缓存与历史修剪）；
    - `roleUserInfo/index.vue` 增加当前角色绑定机构范围徽标（`orgNames`）提示，两页均支持 `?orgId=` / `?roleId=` 路由参数预选与一键返回导航，并升级为 `Zero-Page-Scroll` 布局；
    - `RbacDualListSelector.vue` 与 `RbacDiffPreview.vue` 全面升级为高密度 Tailwind 布局并补齐暗黑模式支持。

---

## 8. 关联方案与排错 SOP 导航
- 实施计划：`user_multi_org_plan.md`、`menu_info_optimization_plan.md`、`permission_info_optimization_plan.md`、`role_info_optimization_plan.md`
- 历史开发实施方案：
  - `02-Features/02-RBAC-用户组织权限/历史开发方案/2026-05-07-rbac-phase1-backend.md`
  - `02-Features/02-RBAC-用户组织权限/历史开发方案/2026-05-07-rbac-system-design.md`
- 关联高危排错 SOP：
  - [[02-Features/02-RBAC-用户组织权限/避坑与Bug修复SOP/避坑SOP-登录异步join漏等|避坑SOP-登录异步join漏等]]
  - [[02-Features/02-RBAC-用户组织权限/避坑与Bug修复SOP/避坑SOP-Redis共享菜单污染|避坑SOP-Redis共享菜单污染]]

---

## 9. 系统与权限菜单体系规范化收敛 (2026-10 落地)

- **痛点与命名缺陷修复**：
  - 原父级菜单命名为「用户管理」，但展开项包含「机构管理」、「菜单管理」、「权限信息」、「角色管理」，产生严重的范畴以偏概全与认知混淆；
  - 同一层级下「信息」与「管理」后缀混用（`用户信息` vs `机构管理` vs `权限信息`），风格割裂且与用户端产生语义套娃。
- **Ponytail × Impact-Cable 极简规范落地**：
  - **一级菜单**：`用户管理` $\rightarrow$ **`系统管理`**（`id = 3`, `name = 'user'`, `path = '/user'`，保持路由 path 与组件完全稳定）；
  - **二级菜单统一**：
    - `用户信息` $\rightarrow$ **`用户管理`**（`id = 10`, `name = 'userManager'`）；
    - `权限信息` $\rightarrow$ **`权限管理`**（`id = 1747162769425887234`, `name = 'permissionInfo'`）；
    - `权限信息详情` $\rightarrow$ **`权限管理详情`**（`id = 1747162770101170177`）。
  - **RBAC 业务依赖自然流向排序 (`order_by`)**：
    1. **机构管理** (`orgManager`, `order_by = 10`)：顶层组织拓扑基座；
    2. **用户管理** (`userManager`, `order_by = 20`)：归属于机构的具体账号主体；
    3. **角色管理** (`roleInfo`, `order_by = 30`)：承接用户与权限的 RBAC 角色载体；
    4. **菜单管理** (`menuInfo`, `order_by = 40`)：系统可分配的前端页面与路由资源；
    5. **权限管理** (`permissionInfo`, `order_by = 50`)：原子级按钮与后端 API 鉴权码。
- **迁移与防线守护**：
  - 全量底库与补丁同步：统一合并追加至 `backend/alex_miaosha/doc/sql/alex_user.sql` 尾部，不新增冗余文件；
  - E2E 自动化测试别名对齐：`alex_miaosha_front/scripts/playwright/run-rbac-smoke.mjs`。

---

## 10. `user_api` 底座工具包结构治理 (2026-10-16 落地)

- **痛点与坏味道治理**：
  - 原工程在 `com.alex.api.user` 下存在单类孤儿包 `user`（仅含 `UserUtils.java`），导致完整全限定名出现 `user.user.UserUtils` 双重叠字与命名冗余；
  - 与同级核心领域包 `userInfo`（承载 `TUserVo`、`UserApi`、`UserPermissionContextVo`）概念同质竞争，职责边界割裂。
- **治理与规范落地**：
  - **包路径收敛**：彻底删除 `com.alex.api.user.user` 孤儿目录，将 `UserUtils` 统一归入 `com.alex.api.user.userInfo.UserUtils`；
  - **全工程无缝对齐**：全量批量替换 4 大微服务（`alex_miaosha_user`、`alex_miaosha_finance`、`alex_miaosha_product`、`alex_miaosha_oss`）共计 55 处 import 引用；
  - **多服务编译与单测防线**：所有微服务编译（`mvn test-compile`）及核心权限/礼尚往来单测集（`RoleOrgAssignApiGuardTest`、`GiftDataScopeOrgSharedTest` 等 80+ 项用例）100% 保持绿灯。





