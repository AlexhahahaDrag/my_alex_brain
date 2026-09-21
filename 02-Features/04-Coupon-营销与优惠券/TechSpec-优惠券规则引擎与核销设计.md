---
title: TechSpec - 优惠券规则引擎与核销设计
tags: [techspec, coupon, promotion, rule-engine, checkout, transaction]
aliases: [优惠券架构方案, 规则引擎Spec]
created: 2026-09-16
updated: 2026-09-16
status: active
---

# 🛠️ TechSpec - 优惠券规则引擎与核销设计

返回功能说明：[[02-Features/04-Coupon-营销与优惠券/PRD-优惠券营销与核销功能说明|PRD-优惠券营销与核销功能说明]]

---

## 1. 核心模型设计

```mermaid
erDiagram
    COUPON_TEMPLATE_T ||--o{ USER_COUPON_T : "模板实例化为用户券"
    COUPON_TEMPLATE_T ||--o{ COUPON_SCOPE_T : "适用范围配置"

    COUPON_TEMPLATE_T {
        bigint id PK "券模板ID"
        varchar title "优惠券标题"
        int type "1满减 2折扣 3立减"
        decimal threshold_amount "门槛金额"
        decimal discount_amount "减免金额或折扣率"
        int total_count "发行总数量"
        int received_count "已发放数量"
        int limit_per_user "每人限领张数"
    }

    USER_COUPON_T {
        bigint id PK "用户券记录ID"
        bigint template_id FK "券模板ID"
        bigint user_id "持有用户ID"
        varchar coupon_code "唯一券码"
        int status "0未使用 1已锁定 2已使用 3已过期"
        datetime start_time "生效时间"
        datetime end_time "失效时间"
        bigint order_id "核销订单ID"
    }
```

---

## 2. 规则引擎匹配与计算管道

```mermaid
flowchart TD
    A[订单商品结算列表] --> B{1. 范围过滤}
    B -- 全场通用/品类匹配/SKU匹配 --> C{2. 有效期校验}
    B -- 不匹配 --> D[置灰: 范围不适用]
    C -- 当前时间在有效区间内 --> E{3. 门槛计算}
    C -- 超期 --> F[置灰: 已过期]
    E -- 匹配商品合计金额 >= 门槛 --> G[加入可用优惠券列表]
    E -- 金额不足 --> H[置灰: 还差 ¥XX 可用]
    G --> I[按抵扣金额降序排序，推荐最优券]
```

---

## 3. 两阶段核销与防超发并发控制

1. **领券防超发**：采用 Redis `DECR` 预扣减券库存，并用 Redis Set 记录 `user_id` 判定限领数量，避免穿透数据库；
2. **两阶段核销**：
   - **创建订单时（预占）**：`UPDATE user_coupon_t SET status = 1, order_id = #{orderId} WHERE id = #{id} AND status = 0;`（乐观锁保证唯一锁定）；
   - **支付成功时（核销终态）**：`UPDATE user_coupon_t SET status = 2 WHERE id = #{id} AND status = 1;`；
   - **超时取消/退款（回补）**：`UPDATE user_coupon_t SET status = 0, order_id = null WHERE id = #{id} AND status = 1;`。
