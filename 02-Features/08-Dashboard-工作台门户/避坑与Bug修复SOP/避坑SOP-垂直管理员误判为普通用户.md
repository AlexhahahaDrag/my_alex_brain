---
title: "避坑SOP - 垂直业务管理员(如family_admin)误判为普通用户排错复盘"
tags: [sop, bugfix, dashboard, rbac, permissions]
aliases: ["管理员角色误判SOP", "family_admin误判"]
created: 2026-09-22
updated: 2026-09-22
status: active
---

# 🚨 避坑SOP - 垂直业务管理员(如 family_admin)误判为普通用户排错复盘

返回目录：[[02-Features/08-Dashboard-工作台门户/PRD-多角色差异化工作台门户|工作台PRD]] | [[02-Features/08-Dashboard-工作台门户/TechSpec-多角色工作台分流与组件架构|工作台TechSpec]]

---

## 1. 故障现象 (Symptom)
- **用户现场**：用户账号 `臭屁宝` 登录系统，所属机构为 `莫莫家`，通过 DevTools 控制台可查看到其名下拥有 `roleCode: "family_admin"`（家庭管理员）与 `roleCode: "gift_admin"`（人情管理员），并且系统全局 Header 头像下拉框显示角色为“家庭管理员”；
- **异常表现**：首页欢迎横幅却错误渲染了绿色徽标 **【普通用户】**，且被分流进入普通个人工作台，无法看到机构/家庭大盘与人情事件流转统计。

---

## 2. 根因剖析 (Root Cause)
1. **角色判定硬编码狭隘**：
   在最初设计 `resolveDashboardRole` 时，仅对 `roleCode === 'admin'` 进行了判断；
2. **垂直业务管理员编码多样化**：
   在礼尚往来与多租户架构中，管理员角色并非仅有单一的 `admin`，而是包含了垂直领域的 `family_admin`（家庭管理员）、`gift_admin`（人情管理员）、`finance_admin`（财务审核员）等；
3. **欢迎横幅标签展示名脱离上下文**：
   原组件标签未读取当前用户的真实 `roleInfo?.roleName`，而直接用字典枚举写死输出。

---

## 3. 防御与修复策略 (Prevention & Solution)
1. **多重管理员启发式判定 (`isAdministratorRole`)**：
   ```typescript
   export const isAdministratorRole = (role?: { roleCode?: string; roleName?: string } | null): boolean => {
     if (!role) return false;
     const code = role.roleCode?.toLowerCase() || '';
     const name = role.roleName || '';
     return (
       code === 'admin' ||
       code.endsWith('_admin') ||
       code.includes('admin') ||
       name.includes('管理员') ||
       name.includes('主管') ||
       name.includes('经理')
     );
   };
   ```
2. **标签优先采用真实名称**：
   `roleTagName` 优先展示 `userStore.getRoleInfo?.roleName`（如“家庭管理员”），确保界面与右上角头像身份 100% 保持一致；
3. **家庭/企业双模语义自适应**：
   检测机构名称包含“家”或角色为家庭人情相关时，指标卡自动切换“家庭在册成员”、“家庭礼尚往来流转”，提供更自然生动的业务体感。
