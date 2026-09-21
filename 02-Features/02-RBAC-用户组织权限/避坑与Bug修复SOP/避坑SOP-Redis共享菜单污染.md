---
title: 避坑 SOP - Redis 共享菜单污染
tags: [backend, frontend, troubleshooting, redis, memory-pollution, rbac]
aliases: [菜单污染SOP, Redis共享菜单污染SOP]
created: 2026-08-30
updated: 2026-09-16
status: active
---

# 🚨 避坑 SOP - Redis 共享菜单污染

所属功能模块：[[02-Features/02-RBAC-用户组织权限/PRD-用户体系与组织权限功能说明|PRD-用户体系与组织权限功能说明]]

---

## 1. 现象与危害
超级管理员登录退出后，普通用户登录时，页面左侧菜单栏意外出现无权的系统管理菜单。

---

## 2. 根本原因
Redis 缓存了公共全量菜单树（`LoginKey:login:in:menu_all_tree`）。在后端过滤或前端 Pinia 生成路由时，代码直接就地修改了 `node.children` 属性，污染了 Java/JS 运行时的内存对象引用，并可能被反写回 Redis。

---

## 3. 防护规约
1. **前端**：必须使用 `structuredClone(menuTree)` 进行深拷贝后再执行角色过滤；
2. **后端**：从 Redis 获取共享菜单树时，必须实例化新集合返回，严禁在原对象引用上修改数据结构。
