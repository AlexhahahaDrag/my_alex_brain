---
title: PRD - 文件中心与附件存取功能说明
tags: [prd, oss, storage, upload, attachment, fullstack]
aliases: [OSS功能说明, 文件中心PRD]
created: 2026-09-16
updated: 2026-09-16
status: active
---

# 📋 PRD - 文件中心与附件存取功能说明

返回首页：[[Home|知识库首页]] | 架构总览：[[00-System/全栈系统架构总览与交互流|系统架构总览]]

---

## 1. 业务全景与服务定位

`alex_miaosha_oss`（端口 `30009`）作为全系统的基础资源服务，为 PC 端、移动端以及内部微服务（用户头像、商品轮播图、Excel 导出报表、礼簿附件等）提供统一的高性能、安全可靠的文件上传、存储与流式下载能力。

```mermaid
graph TD
    Client[PC端 / 移动端] -->|1. Multipart 上传| OSS_Service[alex_miaosha_oss (Port: 30009)]
    OSS_Service -->|2. 流式管道写入| StorageEngine{存储后端适配器}
    StorageEngine -->|私有化| MinIO[MinIO 集群]
    StorageEngine -->|公有云| CloudOSS[阿里云 OSS / 腾讯云 COS]
    StorageEngine -->|本地开发| LocalDisk[本地持久化目录]
    OSS_Service -->|3. 记录元数据审计| DB[(t_oss_file_info)]
    OSS_Service -->>|4. 返回 CDN / 直链 URL| Client
```

---

## 2. 端侧功能与业务流程

### 2.1 业务存取场景
1. **用户头像存取**：用户中心与个人资料头像修改，实时裁剪上传，返回公开访问 URL；
2. **商品图文素材**：商品 SPU 主图、规格 SKU 缩略图、富文本详情配图批量上传；
3. **大数据报表暂存**：礼金大盘、订单台账、财务流水大批量异步导出生成 Excel 文件后的临时中转下载链接。

### 2.2 安全审计与防伪
1. **MIME 类型白名单**：严格限制允许上传的文件扩展名（图片 `jpg/png/webp`，文档 `pdf/xlsx/docx`），严禁上传 `.exe`, `.sh`, `.bat`, `.jsp`, `.php` 等可执行文件；
2. **文件名 UUID 混淆**：物理存储文件名全部采用 `UUID + 原始扩展名`，隔离真实物理路径，杜绝路径遍历注入；
3. **元数据溯源审计**：数据库表 `t_oss_file_info` 记录上传人 ID、原始文件名、文件哈希（SHA-256）、文件大小与存储引擎。

### 2.3 多附件批量上传接口 (`POST /api/v1/file-info/multi-upload`)
1. **多附件并行上传**：支持通过 `POST /file-info/multi-upload`（或向后兼容的 `/batch`）上传多个附件，底层采用 `CompletableFuture` 并行冲刷至 S3 存储引擎，将吞吐耗时从 $O(N)$ 降至 $\max(T_i)$；
2. **批次配额与白名单校验**：单批次最大支持 9 个附件；前端传入空列表或超过 9 个附件即刻校验阻断；严格过滤扩展名，仅允许安全白名单后缀（`jpg, jpeg, png, gif, webp, bmp, svg, pdf, xlsx, xls, docx, doc, txt, zip, rar, csv`），拦截 `.exe, .sh, .bat` 等危险可执行后缀；
3. **即时回填预签名预览**：批量上传成功落库后，服务端自动对所有文件（包括主图及缩略图）生成临时只读预签名 URL 并回填到 `preUrl` 与 `preThumbnailUrl` 字段，前端 `a-upload` 或 `van-uploader` 接收响应后可直接本地预览，无需二次反查；
4. **Saga 补偿机制保证一致性**：多文件并行上传中，若某分片上传失败或后续元数据批量落库时抛出数据库异常，自动触发 Saga 补偿事务，追溯清理本批次已成功上传至 S3 的孤儿临时文件，保证跨系统存储零脏数据泄漏。

### 2.4 文件信息查询与直链生成接口 (`GET /getFileInfo`)
1. **接口入参**：
   - `fileIdList`: 文件 ID 列表（必填）；
   - `isPublic`: 是否公开预览直链（可选，`Boolean`）。
2. **直链决策与业务语义**：
   - `isPublic = true`：强制输出免签持久直链（永久有效，适用于用户头像、商品图等需公开展现的静态资源）；
   - **`isPublic` 不是 `true`（包括 `false` 及 `null` 缺省未传）**：不论什么存储桶，**一律输出带 1 小时时效的 S3 预签名 URL**，默认保障系统对象高安全防盗链。
3. **返回数据封装**：`FileInfoVo` 包含 `preUrl`（主图预览直链）、`preThumbnailUrl`（缩略图预览直链）及回显的 `isPublic` 标识。

---

## 3. 验收标准 (Acceptance Criteria)

- **[AC1] 严格流式管道传输**：严禁将大文件一次性加载到 JVM 内存 `byte[]` 数组中，杜绝 768M 限制下的 OOM；上传必须基于 `file.getSize()` 传递完整长度，严禁依赖 `available()`；
- **[AC2] 文件防篡改与安全阻断**：当上传包含未知扩展名或恶意后缀的文件时，服务层立即阻断并抛出 `400 Invalid File Type` 业务异常；
- **[AC3] 预签名 URL 拓扑隔离**：支持外部 `publicUrl` 域名映射，预签名访问链接严禁将底层内部存储集群物理 IP 与内部端口直接暴漏给外网客户端；
- **[AC4] 消费端流式下载完整性**：文件下载服务交付开放的流或直接管道化冲刷至 `HttpServletResponse`，禁止在工具层过早关流导致客户端报 `Stream closed`；
- **[AC5] 多附件并发与 Saga 事务性补偿**：`POST /file-info/multi-upload` 接口支持并发流式上传并返回补齐 `preUrl` 的 VO 列表；当批次内任何文件校验失败或 DB 批量持久化失败时，精准触发 Saga 补偿撤销并删除已写入 S3 的物理对象，且单元测试通过断言验证覆盖。
