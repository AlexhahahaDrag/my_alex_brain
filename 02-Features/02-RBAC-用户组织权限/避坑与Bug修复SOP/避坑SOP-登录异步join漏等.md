---
title: 避坑 SOP - 登录异步 join 漏等
tags: [backend, troubleshooting, login, async, completablefuture, rbac]
aliases: [登录异步漏等SOP, 异步登录排错SOP]
created: 2026-08-30
updated: 2026-09-16
status: active
---

# 🚨 避坑 SOP - 登录异步 join 漏等

所属功能模块：[[02-Features/02-RBAC-用户组织权限/PRD-用户体系与组织权限功能说明|PRD-用户体系与组织权限功能说明]]

---

## 1. 现象与危害
前端登录成功跳转后，后续接口偶发性报错 `401 Unauthorized` 或 `Permission context is null`，用户头像为空。

---

## 2. 根本原因
登录逻辑中，头像 OSS 获取与权限树计算通过 `CompletableFuture.supplyAsync` 异步执行。若主线程漏写 `allFutures.join()`，主线程会直接向 Redis 写入未完成组装的登录态并返回给前端，前端立即发起的请求在 Redis 中命中空权限数据。

---

## 3. 正确修复代码

```java
CompletableFuture<String> avatarFuture = CompletableFuture.supplyAsync(() -> ossService.getAvatarUrl(userId));
CompletableFuture<UserContext> permFuture = CompletableFuture.supplyAsync(() -> permissionService.buildContext(userId));

// ✅ 必须显式 join 阻塞等待两项异步任务全部执行完毕
CompletableFuture.allOf(avatarFuture, permFuture).join();

// 写入 Redis 并返回
redisTemplate.opsForValue().set(loginTokenKey, token, 7, TimeUnit.DAYS);
return buildLoginResult(token, avatarFuture.getNow(null), permFuture.getNow(null));
```
