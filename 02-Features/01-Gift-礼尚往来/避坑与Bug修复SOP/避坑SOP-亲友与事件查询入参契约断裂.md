# 避坑SOP：礼尚往来模块查询条件字段断裂与契约对齐规范

## 1. 现象描述

在 PC 端「亲友与人情关系档案」（`#/finance/gift/person`）页面：
- 在搜索框中输入姓名（如“赵偶男”）并点击“查询结果”，表格仍展示全量 54 条数据，查询过滤完全不生效。
- 在网络面板中抓包显示请求 `/gift-person-info-t/business-page` 返回 200，但返回结果包含了所有未过滤的记录。

## 2. 根本原因（Root Cause）

前后端接口入参字段名不一致（契约断裂）：

1. **前端模板错误绑定**：
   - `person/index.vue` 模板中搜索框绑定了 `searchInfo.personName`。
   - 过滤下拉框绑定了 `searchInfo.scope`。
2. **后端入参实体定义**：
   - 后端 `GiftPersonQuery` 定义的全文模糊检索字段是 **`keyword`**（用于在 SQL 中对 `person_name` 和 `phone` 联合执行 LIKE 匹配）。
   - 范围过滤字段是 **`personScope`**（取值 `CONTACT` / `ORG_MEMBER`）。
   - 后端并没有 `personName` 和 `scope` 属性，Jackson 反序列化时自动忽略了这两个未知字段，导致 `query.getKeyword()` 与 `query.getPersonScope()` 为 `null`。
3. **MyBatis 动态 SQL 未触发**：
   - XML 映射中的 `<if test="query.keyword != null and query.keyword != ''">` 因参数为 `null` 而被跳过，最终退化为无条件查询。

## 3. 规范与修复方案

### 前端对齐（规范标准）

在各业务列表页统一遵守以下契约规范：

| 业务页面 | 搜索输入项 | 规范绑定字段 | 后端 Query 字段 | SQL 匹配列 |
| :--- | :--- | :--- | :--- | :--- |
| **亲友档案** (`gift/person`) | 姓名/电话/称呼 | `searchInfo.keyword` | `GiftPersonQuery.keyword` | `p.person_name`, `p.phone` |
| **亲友档案** (`gift/person`) | 人员范围 | `searchInfo.personScope` | `GiftPersonQuery.personScope` | `p.bind_user_id IS (NOT) NULL` |
| **事件管理** (`gift/event`) | 事件名称 | `searchInfo.keyword` | `GiftEventQuery.keyword` | `e.event_name` |
| **礼金记录** (`gift/record`) | 姓名/事由/备注 | `searchInfo.keyword` | `GiftRecordQuery.keyword` | `p.person_name`, `e.event_name`, `r.remark` |

并在前端 `loadPage` 组装参数时加入向后兼容防御：
```typescript
const queryParams = { ...searchInfo.value };
if (!queryParams.keyword && (queryParams as any).personName) {
    queryParams.keyword = (queryParams as any).personName;
}
if (!queryParams.personScope && (queryParams as any).scope) {
    queryParams.personScope = (queryParams as any).scope;
}
```
