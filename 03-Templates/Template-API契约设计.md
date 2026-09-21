---
title: API 契约: {{title}}
tags: [standard, api, contract, {{module}}]
aliases: [{{alias}}]
created: {{date}}
updated: {{date}}
status: draft
---

# 🔒 API 契约设计: {{title}}

返回导航：[[00-Overview/前后端ID安全与数据契约护城河|00-Overview/前后端ID安全与数据契约护城河]]

---

## 1. 接口概述
- **功能描述**：
- **请求路径**：`POST /api/{{module}}/{{endpoint}}`
- **认证方式**：Bearer Token (Header: `Authorization`)

---

## 2. 请求契约 (Request DTO)

```typescript
export interface {{InterfaceName}}Query {
  // 必须保持 string 避免 JavaScript 精度丢失
  id?: string;
  pageNum: number;
  pageSize: number;
}
```

---

## 3. 响应契约 (Response VO)

```json
{
  "code": 200,
  "data": {
    "id": "1823910283918239102",
    "name": "示例数据"
  },
  "message": "success"
}
```

> [!IMPORTANT]
> - 后端 VO 字段必须标注 `@JsonSerialize(using = Long2StringSerializer.class)`；
> - 前端必须解构接收：`const { code, data, message } = await api()`。
