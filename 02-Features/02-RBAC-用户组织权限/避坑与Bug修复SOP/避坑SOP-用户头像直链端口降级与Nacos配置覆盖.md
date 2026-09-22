# 避坑SOP-用户头像直链端口降级与Nacos配置覆盖

## 1. 问题现象
在管理端“用户信息”列表页面（`alex_miaosha_front` 用户管理/用户信息列表）中，表格中的“个人头像”图片加载裂开（显示破损图）。
排查前端请求图片地址发现 URL 格式类似：
`http://115.190.181.243:3900/alex-miaosha/upload/2026/09/...`
携带了内部 S3 存储底座（Garage）的物理端口 `3900`，导致外网公网用户直接发起连接时超时裂开。

---

## 2. 根因剖析（Root Cause）

### 2.1 链路无缓存与动态生成
用户登录信息在登录态中缓存于 Redis（`LoginKey:login:in:...`），但管理端**用户信息列表**（`TUserServiceImpl.getPage`）为**实时分页查询** MySQL `t_user` 表，通过 Feign 实时调用 OSS 服务（`ossApi.getFileInfo(fileIdList, true)`）动态生成公开免签直链。

### 2.2 BaseS3Template 降级机制
在 OSS 服务的免签持久直链生成逻辑（`BaseS3Template#buildPublicDirectUrl`）中：
```java
if (StringUtils.isNotBlank(publicUrl)) {
    base = publicUrl.endsWith("/") ? publicUrl.substring(0, publicUrl.length() - 1) : publicUrl;
} else if (StringUtils.isNotBlank(url)) {
    // 若 publicUrl 为空，降级回退到内部物理端点与端口（即 115.190.181.243:3900）
    boolean isSecure = Boolean.TRUE.equals(secure);
    base = (isSecure ? "https://" : "http://") + url
            + (port != null && port != 80 && port != 443 ? ":" + port : "");
}
```

### 2.3 Nacos 远程配置中心覆盖机制
Spring Cloud 启动时，远程 Nacos 配置中心（`minio.yaml` / `alex-oss-dev.yaml`）优先级高于微服务本地的 `application-dev.yaml`。若 Nacos 远程配置中心未显式配置 `garage.publicUrl`（或 `garage.public-url`），本地配置将被覆盖或忽略，导致 `GarageProperties.publicUrl` 注入为 `null`，触发回退逻辑并将物理端口 `3900` 拼入免签直链。

### 2.4 网络拓扑与端口安全
`3900` 为 Garage 内部 RPC/S3 物理端口，外部请求必须由 Nginx（80/443）统一反向代理。公网未对 3900 端口开放防火墙安全组，导致直接请求 3900 端口超时破损。

---

## 3. 标准配置 SOP

在 Nacos 配置中心（`115.190.181.243:8848`，命名空间 `alex-miaosha`）的 `minio.yaml`（或 `alex-oss-dev.yaml`）中，确保 `garage` 与 `minio` 均配置了 `publicUrl`：

```yaml
garage:
  url: 115.190.181.243
  port: 3900
  accessKey: GK453146754fe421b2fd6664fc
  secretKey: 3f0fb8b2db7d251b79d16acf6b60a6618f89a0f745e5757af50468aa16d49010
  bucketName: alex-miaosha
  region: garage
  secure: false
  publicUrl: http://115.190.181.243  # 指向 Nginx 80 反向代理入口
```

---

## 4. 验证检查点

1. 查看 OSS 启动日志 `alex-oss-dev-info.log`，确认存储客户端初始化输出：
   `[S3Storage] 存储客户端初始化成功: endpoint=..., publicUrl=http://115.190.181.243`
2. 刷新 PC 端用户列表，确认接口返回的 `avatarUrl` 格式为 `http://115.190.181.243/alex-miaosha/...`（无 3900 端口）。
3. 检查 Nginx 反向代理配置，确认 `location /alex-miaosha/` 转发至内部 `127.0.0.1:3900`。
