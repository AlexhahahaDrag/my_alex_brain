---
title: 避坑SOP - 端到端烟测弹窗遮罩拦截与移动端权限上下文断裂
aliases: [E2E弹窗遮罩避坑, 移动端权限上下文断裂, Playwright遮罩拦截]
created: 2026-10-15
status: active
tags: [sop, playwright, e2e, pinia, permission, mobile, pc]
---

# 🛑 避坑SOP - 端到端烟测弹窗遮罩拦截与移动端权限上下文断裂

返回导航：[[Home]] | 测试规约：[[01-Standards/全栈测试标准与质量工程规约|全栈测试标准与质量工程规约]] | 业务设计：[[02-Features/01-Gift-礼尚往来/TechSpec-礼尚往来业务领域模型与契约|TechSpec-礼尚往来业务领域模型与契约]]

---

## 1. 现象与报错

在执行 PC 端与移动端自动化 AI 烟测（`pnpm test:ai:smoke` / `run-mobile-ai-smoke.mjs`）时，暴露了两处系统级隐患：

### 现象 1：抽屉/弹窗残存拦截指针 (Pointer Intercept Pitfall)
```text
TimeoutError: locator.click: Timeout 10000ms exceeded.
Call log:
  - waiting for locator('[data-testid="gift-record-add-btn"]')
  - locator resolved to <button data-testid="gift-record-add-btn">...</button>
  - attempting click action
  - waiting for element to be visible, enabled and stable
  - element is visible, enabled and stable
  - scrolling into view if needed
  - done scrolling
  - <div class="ant-drawer-mask"></div> intercepts pointer events
```
**表现**：前序用例打开了 Ant Design Drawer 或 Vant Popup 后，未显式关闭或关闭动画未完成，后续用例尝试点击底层的主界面按钮时被浮层遮罩拦截，导致 Playwright 点击超时挂起。

### 现象 2：移动端权限上下文断裂与异常跳出
```text
[PAGE LOG] [usePermission] userStore.getPermissionContext is not a function
[PAGE LOG] [usePermission] 权限校验失败: gift:view, 用户无权访问
[ROUTER] 路由守卫拦截，强制重定向至 /gift/giftList
```
**表现**：移动端在跳转亲友画像详情页（`giftPersonDetail`）时，由于权限校验直接失败，页面被路由守卫强行踢回列表页，详情页元素完全未渲染。

---

## 2. 根因剖析

1. **E2E 用例生命周期污染**：
   - 很多自动化测试脚本只关注“打开弹窗 -> 输入 -> 校验结果”，往往在断言成功后立即退出当前 step，留下了展开的 Drawer/Dialog；
   - Playwright 执行 `click()` 时具有严格的 Actionability 检查机制，只要有 `mask` / `overlay` 处于更高 `z-index`，即便底层元素可见也坚决不会派发鼠标事件，造成假死。
2. **跨端 Pinia Store 契约不一致**：
   - PC 端与移动端虽然复用相同的权限体系，但移动端 `store/modules/user/user.ts` 未暴露 `getPermissionContext` getter；
   - `src/composables/usePermission.ts` 在执行 `hasPermission` 时，读取 `userStore.getPermissionContext` 为 `undefined`，导致 `apiPermList` 和 `buttonPermList` 均为空，甚至普通用户与超级管理员（`super_super`）都无法放行。
3. **HTTP 谓词（Method）匹配不精准**：
   - 在等待网络响应 `page.waitForResponse` 时，混淆了 `PUT` 与 `POST`（例如 `markGiftReturned` 在底层是 `putData`），如果监听器过滤了 `req.method() === 'POST'`，将永远等不到响应触发。

---

## 3. 防御性规范与修复方案

### 规范 1：弹窗/抽屉用例必须执行闭环关闭与幂等清理
在每个涉及弹窗或抽屉的测试用例结束前，必须显式触发关闭动作并等待遮罩脱离文档树；在用例入口增加幂等检查：
```js
// 1. 关闭抽屉与遮罩
const drawerClose = page.locator('.ant-drawer-close, [aria-label="Close"]');
if (await drawerClose.isVisible({ timeout: 1000 }).catch(() => false)) {
  await drawerClose.first().click();
  await page.waitForSelector('.ant-drawer-mask', { state: 'hidden', timeout: 5000 }).catch(() => {});
}

// 2. 移动端 Vant Popup 遮罩清理
const overlay = page.locator('.van-overlay');
if (await overlay.isVisible({ timeout: 500 }).catch(() => false)) {
  await overlay.first().click({ position: { x: 5, y: 5 } }).catch(() => {});
  await page.waitForSelector('.van-overlay', { state: 'hidden', timeout: 3000 }).catch(() => {});
}
```

### 规范 2：移动端用户 Store 补齐权限上下文 Getter 并支持超级管理员短路
在 `alex_miaosha_mobile/src/store/modules/user/user.ts` 中补齐 getter：
```ts
getters: {
  getPermissionContext(): UserPermissionContext | null {
    return this.permissionContext || null;
  },
  getSuperAdmin(): boolean {
    return this.roleList?.includes('super_super') || false;
  }
}
```
并在 `src/composables/usePermission.ts` 中加入最高权限短路防御：
```ts
export function usePermission() {
  const userStore = useUserStore();

  const hasPermission = (code: string | string[]): boolean => {
    // 1. 超级管理员直接放行
    if (userStore.getSuperAdmin) return true;

    // 2. 防御性获取上下文
    const ctx = userStore.getPermissionContext;
    if (!ctx) return false;
    
    const codes = Array.isArray(code) ? code : [code];
    return codes.some(c => ctx.buttonPermList?.includes(c) || ctx.apiPermList?.includes(c));
  };

  return { hasPermission };
}
```

### 规范 3：自动化脚本精准匹配 HTTP 谓词
在 Mock 拦截和 `waitForResponse` 监听时，必须与 API 定义严格对齐，支持多谓词兼容：
```js
// 兼容 PUT 与 POST 回礼标记
const respPromise = page.waitForResponse(
  req => req.url().includes('/gift-record-info-t/mark-returned') &&
         (req.method() === 'PUT' || req.method() === 'POST')
);
```

---

## 4. 验证效果

1. **PC 端领域烟测**：`pnpm test:ai:smoke` 7/7 自动化用例全数通过，无任何遮罩拦截超时；
2. **移动端领域烟测**：`run-mobile-ai-smoke.mjs` 4/4 自动化用例全数通过，亲友画像详情页权限校验畅通，正常渲染无异常重定向。
