---
title: 告别写死 where 条件！基于 MyBatis-Plus + JSqlParser 实现企业级动态数据权限拦截器
tags: [MyBatisPlus, JSqlParser, 数据权限, 多租户, 架构设计, SpringBoot]
categories: [架构设计, 生产实战, 权限系统]
date: 2026-09-18 21:50:00
---

# 🛡️ 告别写死 where 条件！基于 MyBatis-Plus + JSqlParser 实现企业级动态数据权限拦截器

> **作者**：Alex
> **专栏**：微服务架构实战与企业级 RBAC 权限中台
> **关键词**：数据权限、JSqlParser、MyBatis-Plus、AST 抽象语法树、ThreadLocal 重入保护
> **阅读时长**：约 16 分钟

---

@[TOC](目录)

---

## 😫 痛点：被 `WHERE` 条件支配的恐惧

在做企业级中后台或 SaaS 系统时，几乎所有开发者都会经历这样的阵痛：

刚开始系统只有几百个用户，大家各管各的，业务代码写得行云流水：
```sql
SELECT * FROM gift_record_info_t WHERE status = 1;
```

很快，业务体量扩大，引入了复杂的**多级组织架构与数据权限矩阵**：
1. **超级管理员（Super Admin）**：俯瞰全盘，能看全公司所有分公司、所有部门的数据；
2. **机构/分公司管理员（Org Admin）**：只能查看属于自己管辖机构及其下属子机构的数据；
3. **普通员工（User）**：哪怕在同一个机构，也只能看自己经手或创建的数据。

于是，恶梦开始了。研发组为了实现隔离，在每个 Service、每个 Mapper XML 乃至每个 LambdaQueryWrapper 里，到处人工手写条件：
```java
// ❌ 噩梦般的侵入式硬编码
if (!isSuperAdmin) {
    if (isOrgAdmin) {
        wrapper.in("org_id", getSubOrgIds(currentOrgId));
    } else {
        wrapper.eq("user_id", currentUserId);
    }
}
```

这种做法有三大致命硬伤：
- **侵入性极强**：业务代码被非功能的权限逻辑严重污染，可读性极差；
- **极易遗漏引发严重泄密**：几十个微服务、数百个接口，新人一旦在某个分页查询里漏写了一行判断，**全公司敏感业务数据瞬间全员裸奔！**
- **无法应对复杂的联表查询**：当业务涉及 `JOIN` 多个业务表时，多表别名和字段映射极其混乱。

我们需要的是一套：**业务代码零侵入、声明式注解挂载、底层由 SQL 解析引擎自动改写 AST 的企业级动态数据权限插件！**

---

## 🏛️ 架构设计：基于 JSqlParser 的 AST 注入拓扑

```mermaid
graph TD
    A[业务发起查询: selectGiftRecordPage] --> B[MyBatis 执行拦截器]
    B --> C{方法是否标注了 @DataPermission?}
    C -->|否| D[原样放行 SQL]
    C -->|是| E[解析 JSqlParser 抽象语法树 AST]
    E --> F[从上下文获取当前登录用户及角色]
    F --> G{当前用户角色类型}
    G -->|超级管理员| H[放行，不拼接任何条件]
    G -->|机构管理员| I[动态注入子查询:<br/>t.org_id IN (SELECT org_id FROM t_org_user_info WHERE user_id = ?)]
    G -->|普通员工| J[动态注入精确限制:<br/>t.user_id = ?]
    I --> K[拼接并生成最终执行 SQL]
    J --> K
    K --> L[底层数据库执行]
```

### 为什么选择 JSqlParser？
很多人尝试用正则表达式或简单的字符串 `replace` 去修改 SQL，这种做法在复杂的 SQL（包含子查询、GROUP BY、ORDER BY、LEFT JOIN）面前脆如薄纸，极易导致语法错误。
**JSqlParser 是一个纯 Java 的 SQL 解析引擎**，它能将一串 SQL 文本解析为**抽象语法树（AST，Abstract Syntax Tree）**。在 AST 上添加一个 `EqualsTo` 或 `InExpression` 条件节点，就像操作 DOM 树一样精准优雅，永远不会产生语法错乱！

---

## 💻 核心落地实战

### 1. 声明式注解定义 (`@DataPermission`)

```java
package com.alex.common.datascope.annotation;

import java.lang.annotation.*;

@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DataPermission {

    /**
     * 主业务表的别名（如 "t"）
     */
    String tableAlias() default "";

    /**
     * 用户所属字段名
     */
    String userColumn() default "user_id";

    /**
     * 机构/租户所属字段名
     */
    String orgColumn() default "org_id";
}
```

---

### 2. 数据权限处理器实现 (`DataPermissionHandlerImpl.java`)

我们继承 MyBatis-Plus 的 `DataPermissionHandler`，实现动态 SQL 条件追加：

```java
@Slf4j
@RequiredArgsConstructor
public class DataPermissionHandlerImpl implements DataPermissionHandler {

    // 性能优化神器：ConcurrentHashMap 缓存方法注解元数据，杜绝每次高频反射
    private static final Map<String, Optional<DataPermission>> ANNOTATION_CACHE = new ConcurrentHashMap<>();

    // 关键防线：ThreadLocal 防重入保护，防止 getLoginUser 内部触发查库引发死循环
    private static final ThreadLocal<Boolean> REENTRANT_GUARD = ThreadLocal.withInitial(() -> false);

    @Override
    public Expression getSqlSegment(Expression where, String mappedStatementId) {
        // 1. 防重入校验
        if (Boolean.TRUE.equals(REENTRANT_GUARD.get())) {
            return where;
        }

        // 2. 检索并命中注解缓存
        DataPermission annotation = getAnnotationFromCache(mappedStatementId);
        if (annotation == null) {
            return where;
        }

        try {
            REENTRANT_GUARD.set(true);

            // 3. 获取当前登录上下文
            LoginUser loginUser = SecurityUtils.getLoginUser();
            if (loginUser == null) {
                return where;
            }

            // 超管直接豁免
            if (loginUser.isSuperAdmin()) {
                return where;
            }

            // 4. 根据角色构建 AST 条件表达式
            Expression permissionExpression = buildPermissionExpression(loginUser, annotation);

            // 5. 与原有的 WHERE 条件通过 AND 逻辑缝合
            return where == null ? permissionExpression : new AndExpression(where, permissionExpression);

        } finally {
            REENTRANT_GUARD.remove(); // 严防线程池污染
        }
    }

    private Expression buildPermissionExpression(LoginUser user, DataPermission annotation) {
        String alias = StringUtils.isNotBlank(annotation.tableAlias()) ? annotation.tableAlias() + "." : "";

        // 机构管理员：注入机构子查询
        if (user.isOrgAdmin()) {
            Column column = new Column(alias + annotation.orgColumn());
            SubSelect subSelect = new SubSelect();
            subSelect.setSelectBody(CCJSqlParserUtil.parseSelect(
                "SELECT org_id FROM t_org_user_info WHERE status = 1 AND user_id = " + user.getUserId()
            ).getSelectBody());
            return new InExpression(column, subSelect);
        }

        // 普通员工：精确限定自己
        Column column = new Column(alias + annotation.userColumn());
        return new EqualsTo(column, new LongValue(user.getUserId()));
    }
}
```

---

## 💣 源码级两大致命暗坑与避坑 SOP

在将拦截器推向生产的过程中，我们踩了两个教科书级别的深坑，极具参考价值：

### 避坑 1：`getLoginUser()` 触发的死循环（StackOverflowError）
- **踩坑现场**：在拦截器中调用 `SecurityUtils.getLoginUser()`，该方法底层为了获取用户最新的有效机构，执行了一次 `orgUserService.getValidOrg(userId)` 查询；
- **致命循环**：这次内部查询又被 MyBatis-Plus 拦截器逮住，拦截器再次触发 `getLoginUser()`，陷入无限套娃递归，最终**微服务抛出 `StackOverflowError` 瞬间雪崩崩溃！**
- **解法**：必须引入 `ThreadLocal<Boolean> REENTRANT_GUARD`。在拦截器前置标记 `true`，拦截器重入时发现标志为 `true` 直接放行原生 SQL，`finally` 块必须严格 `remove()`！

### 避坑 2：MyBatis-Plus `IService.page()` 绕过漏洞
- **踩坑现场**：业务代码里写：
  ```java
  // 💥 致命漏洞：数据越权！
  return giftRecordService.page(page, Wrappers.<GiftRecord>lambdaQuery().eq(GiftRecord::getType, 1));
  ```
- **原因剖析**：`service.page()` 底层执行的是 MyBatis-Plus 默认注入的 `BaseMapper.selectPage()`，**默认的 BaseMapper 方法上根本没有挂 `@DataPermission` 注解！** 拦截器发现无注解直接放行，导致普通用户能查出全公司的数据！
- **规约红线**：
  > 🚨 **团队军规**：凡是涉及组织/多租户隔离的核心业务表，**严禁直接调用 `IService.page()` / `list()`！** 必须在自定义 Mapper 上显式挂载 `@DataPermission` 注解，由 Service 显式调用自定义 Mapper 方法！

```java
// ✅ 正确姿势：显式挂载
public interface GiftRecordMapper extends BaseMapper<GiftRecord> {

    @DataPermission(tableAlias = "t", userColumn = "user_id", orgColumn = "org_id")
    Page<GiftRecordVo> selectCustomPage(Page<GiftRecordVo> page, @Param("query") GiftRecordQuery query);
}
```

---

## 🎯 总结与收获

通过这套基于 **MyBatis-Plus + JSqlParser AST + `@DataPermission`** 的架构重构：

1. **研发效率极大提升**：业务开发人员 100% 聚焦纯业务逻辑，不再需要写一行权限过滤 `where` 代码，只需在 Mapper 方法上打个注解；
2. **安全防护坚如磐石**：在 SQL 解析层强行注入租户与组织隔离边界，从物理层面根除了漏写条件导致越权泄密的高危风险；
3. **架构高可用**：通过注解元数据缓存 + 防重入哨兵机制，保障了高并发下微服务依然拥有极其强悍的稳定性和极低的延迟损耗。

---

*（本文已同步收录至 Alex 知识库：`02-Features/02-RBAC-用户组织权限/TechSpec-RBAC组织与动态数据权限架构.md`）*
