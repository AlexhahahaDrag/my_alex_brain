---
title: TechSpec - RBAC 组织与动态数据权限架构
tags: [techspec, backend, rbac, datapermission, jsqlparser, async, security]
aliases: [RBAC架构, 数据权限实现]
created: 2026-09-16
updated: 2026-09-16
status: active
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

## 3. 关联方案与排错 SOP 导航
- 历史开发实施方案：
  - `02-Features/02-RBAC-用户组织权限/历史开发方案/2026-05-07-rbac-phase1-backend.md`
  - `02-Features/02-RBAC-用户组织权限/历史开发方案/2026-05-07-rbac-system-design.md`
- 关联高危排错 SOP：
  - [[02-Features/02-RBAC-用户组织权限/避坑与Bug修复SOP/避坑SOP-登录异步join漏等|避坑SOP-登录异步join漏等]]
  - [[02-Features/02-RBAC-用户组织权限/避坑与Bug修复SOP/避坑SOP-Redis共享菜单污染|避坑SOP-Redis共享菜单污染]]
