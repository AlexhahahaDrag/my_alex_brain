---
title: 避坑 SOP - 用户列表多角色显示与 RBAC 按钮权限缺失排查
tags: [backend, frontend, rbac, permissions, userManager, troubleshooting]
aliases: [用户列表角色为空SOP, RBAC按钮权限缺失SOP]
created: 2026-09-23
updated: 2026-09-23
status: active
---

# 🚨 避坑 SOP - 用户列表多角色显示与 RBAC 按钮权限缺失排查

所属功能模块：[[02-Features/02-RBAC-用户组织权限/PRD-用户体系与组织权限功能说明|PRD-用户体系与组织权限功能说明]]

---

## 1. 现象与危害
1. **角色名称空白**：用户管理界面（PC 端 `/user/userManager`）表格中，所有用户的「角色名称」列均显示为空白，即使数据库中已为用户绑定了角色。
2. **操作按钮全面消失**：机构管理员（如 `家庭管理员` `family_admin`、`组织用户管理员` `org_user_admin`）登录后，页面顶部「新增/删除」按钮及表格操作列「编辑/删除」按钮完全未渲染，导致非超管管理员无法对辖区用户进行维护。

---

## 2. 根本原因

### (1) 角色名称为空原因（数据契约与装配断层）
- **SQL 聚合但 VO 缺字段**：`TUserMapper.xml` 在 `getPage` 中已查询 `group_concat(ri.role_name) as roleName`，但 `TUserVo` 实体类中缺失 `roleName` 和 `roleCode` 属性，导致 MyBatis 查询结果被静默丢弃。
- **多角色列表未装配**：用户与角色为多对多关系，`getPage` 默认只做了单表联查，未调用 `roleUserInfoService.getRoleInfoList` 为分页 records 装配 `roleInfoVoList` 实体列表。

### (2) 操作按钮消失原因（RBAC 权限元数据缺失 + 指令物理移除）
- **细粒度权限校验**：前端按钮挂有 `v-permission="'user:add'"`、`v-permission="'user:edit'"`、`v-permission="'user:delete'"` 指令。
- **超管与非超管分支**：超管（`super_super`）直接放行；非超管需校验 `permissionSet.has(code)`。
- **权限元数据缺失**：数据库表 `t_permission_info` 初始化时仅录入了菜单级权限码（如 `user:userManager`），未录入 `user:add`, `user:edit`, `user:delete` 等按钮级权限，且 `t_role_permission_info` 未将按钮权限授权给目标管理角色。
- **DOM 物理移除**：校验未通过时，指令调用 `el.parentNode.removeChild(el)` 物理删除了按钮 DOM。

---

## 3. 防护规约与修复方案

### (1) 后端装配与契约规约
1. **VO 字段对齐**：`TUserVo` 必须声明 `private String roleName;`、`private String roleCode;` 以及 `private List<RoleInfoVo> roleInfoVoList;`。
2. **分页数据填充**：在 `TUserServiceImpl.getPage()` 中，必须遍历当前页 `records`，调用 `roleUserInfoService.getRoleInfoList(record.getId(), false)` 注入 `roleInfoVoList`，并聚合 `roleName`/`roleCode`。

### (2) 前端展示规约
- 表格 `roleName` 列使用自定义插槽，优先遍历 `record.roleInfoVoList` 渲染为精致的 `<a-tag color="blue">` 胶囊徽章；多角色并列清晰直观，无角色显示 `-`。

### (3) RBAC 权限标准补齐（严禁前端硬编码兜底）
- 严禁在前端 `utils/permission` 中为特定角色硬编码跳过按钮校验，避免破坏权限体系原子性。
- 通过标准幂等 SQL 脚本（`docs/sql/2026-09-23-user-button-grants.sql`），在 `t_permission_info` 录入按钮权限点，并在 `t_role_permission_info` 授权给 `family_admin`（家庭管理员）、`org_user_admin`（组织用户管理员）等目标管理角色，保障系统权限配置透明可追溯。