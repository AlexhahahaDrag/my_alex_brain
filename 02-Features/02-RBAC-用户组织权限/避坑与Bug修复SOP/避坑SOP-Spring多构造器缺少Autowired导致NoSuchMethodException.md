---
title: 避坑 SOP - Spring 多构造器缺少 @Autowired 导致 NoSuchMethodException
aliases: [Spring多构造函数SOP, NoDefaultConstructorFoundSOP]
created: 2026-10-22
updated: 2026-10-22
status: active
tags: [sop, spring, backend, rbac, di]
---
# 🚨 避坑 SOP - Spring 多构造器缺少 @Autowired 导致 NoSuchMethodException

所属功能模块：[[02-Features/02-RBAC-用户组织权限/PRD-用户体系与组织权限功能说明|PRD-用户体系与组织权限功能说明]]

---

## 1. 现象与危害
微服务（如 `alex-user-dev`）启动阶段在 Undertow / Tomcat 容器初始化 `ServletWebServerApplicationContext` 刷新时直接中断崩溃，抛出如下关键异常日志：

```text
Caused by: org.springframework.beans.factory.BeanCreationException: 
    Error creating bean with name 'menuInfoServiceImp' defined in file [.../MenuInfoServiceImp.class]: 
    Instantiation of bean failed; nested exception is org.springframework.beans.BeanInstantiationException: 
    Failed to instantiate [com.alex.user.menuInfo.service.impl.MenuInfoServiceImp]: No default constructor found; 
    nested exception is java.lang.NoSuchMethodException: com.alex.user.menuInfo.service.impl.MenuInfoServiceImp.<init>()
```

由于 `menuInfoServiceImp` 被 `userPermissionContextServiceImpl`、`TUserServiceImpl`、`JwtAuthenticationTokenFilter` 等核心安全链条级联依赖，此 Bean 实例化失败将导致微服务完全无法启动。

---

## 2. 根本原因

在 Spring Framework 的依赖注入装配规范中：
1. **单一构造函数**：当一个 Bean 类中**仅声明了一个显式构造函数**时，Spring 4.3+ 允许省略 `@Autowired` 注解，容器会自动利用该构造函数装配入参依赖；
2. **多构造函数歧义**：当类中声明了**两个及以上构造函数**（例如为了兼容单元测试重载了一个少参数的轻量构造函数）时，Spring **无法也绝不会自动猜测**该调用哪个构造函数；
3. **错误注解位置**：如果在多参构造器的参数上标注了 `@Autowired(required = false)`，但**构造函数签名本身未添加 `@Autowired`**，Spring 的构造函数决策机制（`ConstructorResolver`）判定没有任何构造函数被显式标记注入；
4. **回退无参构造器失败**：此时 Spring 默认策略为回退去寻找无参默认构造函数 `<init>()`。由于类中已声明了带参构造函数且未提供无参构造函数，Java 编译器不会自动生成无参构造函数，最终抛出 `NoSuchMethodException: ...<init>()`。

---

## 3. 正确修复代码

在需要由 Spring 容器装配的目标多参构造函数上，**显式添加 `@Autowired` 注解**；若某个依赖为可选，可配合 `@Nullable` 或 `@Autowired(required = false)` 使用：

```java
@Service
@Slf4j
public class MenuInfoServiceImp extends ServiceImpl<MenuInfoMapper, MenuInfo> implements MenuInfoService {

    private final MenuInfoMapper menuInfoMapper;
    private final RedisUtils redisUtils;
    private final UserUtils userUtils;
    private final MultiLevelCacheService multiLevelCacheService;

    // ✅ 必须在多构造函数的目标注入构造器上显式标注 @Autowired
    @Autowired
    public MenuInfoServiceImp(MenuInfoMapper menuInfoMapper, 
                              RedisUtils redisUtils, 
                              UserUtils userUtils,
                              @Autowired(required = false) MultiLevelCacheService multiLevelCacheService) {
        this.menuInfoMapper = menuInfoMapper;
        this.redisUtils = redisUtils;
        this.userUtils = userUtils;
        this.multiLevelCacheService = multiLevelCacheService;
    }

    // 用于轻量单测实例化的重载构造函数，不加 @Autowired
    public MenuInfoServiceImp(MenuInfoMapper menuInfoMapper, RedisUtils redisUtils, UserUtils userUtils) {
        this(menuInfoMapper, redisUtils, userUtils, null);
    }
}
```

---

## 4. 预防与测试规约

1. **红线约定**：任何 Spring `@Service`、`@Component` 类，若存在一个以上的构造函数，**必须且仅能在预期的容器注入构造函数上显式声明 `@Autowired`**；
2. **优先使用 Lombok**：单构造函数场景推荐使用 `@RequiredArgsConstructor`；仅当有测试特殊需求必须重载多构造函数时，手动编写并标注 `@Autowired`；
3. **测试防漏**：纯单测通过 `new Class(...)` 往往覆盖不到 Spring 容器装配行为，微服务必须包含至少一个集成测试或启动扫描验证，确保所有 Service 的 Bean 实例化与依赖注入链路在 CI/CD 中被覆盖。
