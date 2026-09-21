---
title: SOP - Garage 对象存储配置与常见报错排查
tags: [sop, oss, garage, s3, troubleshooting, security-group, presigned-url]
aliases: [Garage排障SOP, Garage配置避坑指南]
created: 2026-09-19
updated: 2026-09-19
status: active
---

# 📖 SOP - Garage 对象存储配置与常见报错排查

返回功能说明：[[02-Features/06-OSS-对象存储与文件中心/PRD-文件中心与附件存取功能说明|PRD-文件中心与附件存取功能说明]] | [[02-Features/06-OSS-对象存储与文件中心/TechSpec-对象存储抽象与流式传输设计|TechSpec-对象存储抽象与流式传输设计]]

---

## 1. 架构拓扑与关键端口映射

系统在 Linux 云主机（如火山引擎 ECS）中通过 Docker 部署 Garage 及配套组件：

| 组件名称 | 容器内原生端口 | 宿主机映射端口 | 协议类型 | 作用说明 |
| :--- | :--- | :--- | :--- | :--- |
| **Garage (S3 API)** | `3900` | `3900` | TCP / HTTP | S3 兼容 API 入口，文件上传与下载 |
| **Garage (RPC)** | `3901` | 未映射 | TCP / RPC | Garage 集群内部通信 |
| **Garage (Web)** | `3902` | `3902` | TCP / HTTP | 静态网站与公开匿名读取 |
| **Garage (Admin)** | `3903` | `3903` | TCP / HTTP | 管理 API，供 CLI 与 WebUI 调用 |
| **Garage WebUI** | `3909` | **`3904`** | TCP / HTTP | 可视化管理控制台（宿主机端口为 3904） |

---

## 2. 常见核心故障与排错指南

### 2.1 故障一：浏览器访问 Garage WebUI 报 502 Bad Gateway / ERR_CONNECTION_TIMED_OUT

#### 现象
- 浏览器打开 `http://<IP>:3904` 显示 `502 Bad Gateway` 或超时失败。

#### 根因排查
1. **端口少输或输错**：
   - WebUI 宿主机映射端口是 **`3904`**（`0.0.0.0:3904->3909/tcp`），切忌误输为 3900 或 3909；
   - 检查浏览器地址栏 IP 是否少输或打错（如 `115.190.181.24` 与 `115.190.181.243`）。
2. **云厂商安全组未放行**：
   - 云服务器实例所属安全组入方向未放行 `3900-3904` TCP 端口。
3. **白名单陷阱与本地代理 502**：
   - 若去掉了安全组的 `0.0.0.0/0`，改填局域网私网 IP（如 `192.168.x.x`）是无效的，必须填写宽带的公网出口 IP（可通过 `ip138.com` 查询）；
   - 当本机开启了代理工具（如 Clash、v2ray 等）时，安全组阻断握手会导致本地代理软件作为 Gateway 向浏览器返回 `502 Bad Gateway`。

---

### 2.2 故障二：请求头像等文件返回 400 Bad Request (`Date is too old`)

#### 现象
```xml
<Error>
  <Code>InvalidRequest</Code>
  <Message>Bad request: Date is too old</Message>
  <Resource>/user-bucket/user/2026-03-08/...jpg</Resource>
  <Region>garage</Region>
</Error>
```

#### 根因
- 后端使用 `minioClient.getPresignedObjectUrl` 生成带 S3 签名的 URL，有效时长默认为 1 小时（`expiry: 3600s`）；
- 用户登录后，`avatarUrl` 长期缓存在 Redis 与前端 LocalStorage 中，超过 1 小时后 S3 签名失效，Garage 时间戳校验拒绝访问。

#### 规范解决步骤
1. **基础设施配置**：
   - 登录 Garage WebUI（`http://<IP>:3904`）；
   - 进入 `Buckets` -> `user-bucket` -> `Overview`；
   - 将 **Website Access** 滑块开关切换为 **Enabled（开启）**；
2. **微服务逻辑**：
   - 系统微服务已在 `BucketNameEnum` 将 `user-bucket` / `goods-bucket` 标识为公开桶，`BaseS3Template#preview` 会自动输出不带时效签名的持久免签直链（`http://<IP>:3900/user-bucket/...` 或 CDN 域名），彻底消除时效过期问题。

---

## 3. 防坑检查清单 (Pre-flight Checklist)

- [ ] 火山引擎/阿里云/腾讯云安全组入方向已放行 `TCP 3900`（S3 API）与 `TCP 3904`（WebUI）；
- [ ] 若配置安全组白名单，必须填写当前宽带的公网出口 IP（或 C 段 `/24`），禁止填写 `192.168.x.x`；
- [ ] 头像桶 `user-bucket` 与商品桶 `goods-bucket` 已在 WebUI 中开启 **Website Access**；
- [ ] 线上生产环境推荐在前端 Nginx 配置反向代理（`location ~ ^/([a-zA-Z0-9_-]+)-bucket/`），并将 `garage.publicUrl` 指向外部 80/443 域名，实现物理端口隐藏与内外网拓扑隔离。
