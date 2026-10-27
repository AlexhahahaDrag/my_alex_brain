---
title: 避坑 SOP - 多级缓存泛型擦除反序列化 JSONObject 引发菜单 500
aliases: [多级缓存泛型擦除SOP, 菜单反序列化JSONObject异常SOP]
created: 2026-10-27
updated: 2026-10-27
status: active
tags: [sop, rbac, backend, cache, redis, caffeine, fastjson]
---
# 🚨 避坑 SOP - 多级缓存泛型擦除反序列化 JSONObject 引发菜单 500

所属功能模块：[[02-Features/02-RBAC-用户组织权限/PRD-用户体系与组织权限功能说明|PRD-用户体系与组织权限功能说明]]

---

## 1. 现象与危害
1. 用户在前端登录时，接口 `GET /api/v1/user/menus` 报 HTTP 500（`Internal Server Error`）；
2. 前端控制台输出：`[Vue Router warn]: No match found for location with path "/selfFinance/financeManager"`；
3. 由于动态路由根据菜单生成，菜单加载失败导致动态路由未注册，前端路由守卫拦截并死锁重定向在登录页 `/#/login?redirect=...`，用户完全无法进入系统。

---

## 2. 根本原因与堆栈分析
后端错误日志输出：
```text
Caused by: java.lang.ClassCastException: class com.alibaba.fastjson.JSONObject cannot be cast to class com.alex.api.user.menuInfo.vo.MenuInfoVo
	at com.alex.user.rbac.service.impl.UserPermissionContextServiceImpl.filterMenusByPermissionCodes(UserPermissionContextServiceImpl.java:197)
	at com.alex.user.rbac.service.impl.UserPermissionContextServiceImpl.resolveVisibleMenus(UserPermissionContextServiceImpl.java:145)
	at com.alex.user.rbac.service.impl.UserPermissionContextServiceImpl.buildContext(UserPermissionContextServiceImpl.java:92)
```

### 根因链路：
1. **多级缓存读取方法泛型擦除**：
   在 `MenuInfoServiceImp.java` 中读取全量菜单缓存 `menu_all_tree` 时使用了：
   ```java
   List<MenuInfoVo> cached = multiLevelCacheService.get(cacheRealKey, (Class<List<MenuInfoVo>>) (Class<?>) List.class);
   ```
2. **底层反序列化退化为 JSONObject**：
   `MultiLevelCacheService.get` 仅接受单个 `Class<T> clazz`。传入 `List.class` 时，由于 JVM 泛型擦除，FastJson 无法获取集合内元素的泛型参数，默认反序列化生成了 `List<JSONObject>` 并将其回填到 L1 Caffeine 缓存中。
3. **隐式 CHECKCAST 触发运行时强转崩溃**：
   当 `UserPermissionContextServiceImpl.filterMenusByPermissionCodes` 遍历 `for (MenuInfoVo menuInfoVo : menuList)` 时，JVM 字节码的 `CHECKCAST MenuInfoVo` 指令发现元素是 `JSONObject`，直接抛出 `ClassCastException`。

---

## 3. 终极防护规约

### 1. 缓存层：MultiLevelCacheService 专属 List 泛型方法
在 `MultiLevelCacheService` 中显式扩展 `getList(String key, Class<T> elemClazz)`：
* L1 检查：元素若为目标类型直接返回；若混入 `JSONObject` 脏数据则安全转换为 `List<T>` 并自动修复 L1；
* L2 检查：通过 `JSONObject.parseArray(l2Val, elemClazz)` 正确带类型反序列化为实体列表。

### 2. 业务调用层：禁止强转 `List.class`
读取泛型列表缓存时，严禁使用 `multiLevelCacheService.get(key, (Class<List<X>>) (Class<?>) List.class)`，统一改用：
```java
List<MenuInfoVo> cached = multiLevelCacheService.getList(cacheRealKey, MenuInfoVo.class);
```

### 3. 消费安全防线（纵深防御 Defense in Depth）：
在 `UserPermissionContextServiceImpl.filterMenusByPermissionCodes` 遍历菜单及其子级 `children` 时，以 `Object` 接收并做类型自愈检查（支持 `MenuInfoVo` 直接放行、`JSONObject` 调用 `.toJavaObject(MenuInfoVo.class)` 转换），确保任何异常反序列化都不会导致系统级 500。
