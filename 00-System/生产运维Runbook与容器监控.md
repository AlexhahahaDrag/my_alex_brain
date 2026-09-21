---
title: 生产运维 Runbook 与容器监控
tags: [system, operations, devops, runbook, docker, jvm, drone, mysql]
aliases: [生产运维Runbook, 运维Runbook, Runbook, 容器监控]
created: 2026-05-26
updated: 2026-09-16
status: active
---

# 📖 生产运维 Runbook 与容器监控

返回首页：[[Home|知识库首页]] | 架构总览：[[00-System/全栈系统架构总览与交互流|系统架构总览]]

---

## 1. 生产容器拓扑与资源配额 (768M-JVM)

生产服务器 Docker 容器采用 `limit: 768MiB` 统一配额控制，彻底杜绝单容器占用暴涨拖垮宿主机。

| 容器服务 | 容器端口 | 容器 Memory Limit | 生产 JVM 参数 | 重启策略 |
| :--- | :--- | :--- | :--- | :--- |
| `alex_miaosha_gateway` | 30001 | **768 MiB** | `-Xms384m -Xmx384m -XX:MaxMetaspaceSize=128m` | `unless-stopped` |
| `alex_miaosha_user` | 30006 | **768 MiB** | `-Xms384m -Xmx384m -XX:MaxMetaspaceSize=128m` | `unless-stopped` |
| `alex_miaosha_finance` | 30008 | **768 MiB** | `-Xms384m -Xmx384m -XX:MaxMetaspaceSize=128m` | `unless-stopped` |
| `alex_miaosha_product` | 30007 | **768 MiB** | `-Xms384m -Xmx384m -XX:MaxMetaspaceSize=128m` | `unless-stopped` |
| `alex_miaosha_oss` | 30009 | **768 MiB** | `-Xms384m -Xmx384m -XX:MaxMetaspaceSize=128m` | `unless-stopped` |
| `alex_miaosha_ai` | 30010 | **768 MiB** | `-Xms384m -Xmx384m -XX:MaxMetaspaceSize=128m` | `unless-stopped` |

> [!TIP]
> **768M 内存调优黄金比例**：JVM 堆内存严格限制为 `384M`，预留 `384M` 给 Metaspace (128M)、DirectBuffer、线程栈及操作系统自身进程，从根本上杜绝 Linux OOM Killer 强杀信号。

---

## 2. Drone 持续部署流水线与发布前检查

项目通过 `.drone.yml` 结合 SSH 与 Docker 自动化流水线发布：
发布顺序严格遵循：`finance / product / oss / ai / user ➔ gateway`（入口网关必须最后发布）。

### 发布前观测指令 (`pre-deploy-runtime-snapshot`)
在 Drone 部署步骤中，必须输出当前宿主机容器运行健康快照：

```bash
date -Is
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}\t{{.PIDs}}'
docker events --since 30m --filter event=oom --filter event=restart || true
```

---

## 3. MySQL 慢查询与性能基准

- **日志配置**：
  - `slow_query_log = ON`
  - `slow_query_log_file = /var/log/mysql/slow.log`
  - `long_query_time = 1.0` (超过 1 秒必须优化)
  - `innodb_buffer_pool_size = 1G`
  - `max_connections = 200`
- **健康检查**：采用 `mysqladmin ping -uroot -p${MYSQL_ROOT_PASSWORD}` 探测存活。

---

## 4. 业务服务统一日志挂载

宿主机日志目录统一规划：
- 宿主机根路径：`/usr/local/soft/alex_miaosha/drone/alex_miaosha/logs`
- 挂载容器路径：`/logs`
- 子服务日志落盘：
  - `/logs/alex-finance-prod`
  - `/logs/alex-user-prod`
  - `/logs/alex-oss-prod`
  - `/logs/alex-gateway-prod`
  - `/logs/alex-product-prod`
  - `/logs/alex-ai-prod`
