---
title: 避坑SOP - Vue组件Props数组防御与Playwright测试桩
aliases: [Vue Props防御, Playwright测试桩避坑]
created: 2026-10-15
status: active
tags: [sop, vue3, playwright, gift, frontend]
---

# 🛑 避坑SOP - Vue组件Props数组防御与Playwright测试桩

返回导航：[[Home]] | 业务设计：[[02-Features/01-Gift-礼尚往来/TechSpec-礼尚往来业务领域模型与契约|TechSpec-礼尚往来业务领域模型与契约]]

---

## 1. 现象与报错

在执行 PC 端领域 AI 冒烟测试（`pnpm test:ai:smoke`）时，出现如下异常崩溃：

```text
[BROWSER_PAGE_ERR] PlaywrightError [TypeError]: Cannot read properties of undefined (reading 'length')
    at GiftRecordPartiesField.vue:205:30
[BROWSER_PAGE_ERR] PlaywrightError [TypeError]: list.forEach is not a function
    at setOptions (gift-org-member-picker/index.vue:55:9)
    at applyCachedOrgMembers (gift-org-member-picker/index.vue:66:4)
```

## 2. 根因剖析

1. **模板中直接使用 `props.array.length`**：
   - 子组件在定义 `defineProps<{ recentContacts: Array<...> }>()` 时若未指定默认值（`withDefaults`），父组件传入 `undefined` 或未准备完毕时，模板求值 `recentContacts.length > 0` 会直接抛出 `TypeError` 并中断 Vue 渲染。
2. **Mock 兜底数据与真实契约不匹配**：
   - Playwright 的网络拦截桩在未命中具体路径时，默认兜底返回了 `{ code: '200', data: {} }`（对象而非数组）；
   - 当异步拉取组织成员或联系人列表时，`data || []` 仍然取到了 `{}`（非 null/undefined，但没有 `forEach` 方法），导致后续 `list.forEach` 崩溃。
3. **Ant Design Vue 输入框定位伪失效**：
   - `<a-input>` / `<a-input-number>` 带有 prefix/suffix 时外层由 `ant-input-affix-wrapper` 或自定义 div 包裹，若将 `data-testid` 直接挂在组件根或外层表单项，Playwright `page.locator('[data-testid="xxx"]').fill(...)` 可能会报错 "element is not an <input>"。

## 3. 防御性规范与修复措施

### 规则 1：子组件数组型 Props 必须用 `withDefaults` 并结合空值合并
```ts
const props = withDefaults(
	defineProps<{
		recentContacts?: Array<{ id: string; name: string }>;
		familyMembersList?: Array<{ id: string; name: string }>;
	}>(),
	{
		recentContacts: () => [],
		familyMembersList: () => [],
	},
);
```
模板中使用：
```vue
<div v-if="(recentContacts?.length ?? 0) > 0" class="recent-contacts-tags">
```

### 规则 2：遍历前强制防御 `Array.isArray`
在 Picker、Cache 或状态同步方法中，对传入的列表无论 TS 类型声明如何，一律进行数组形态防护：
```ts
const setOptions = (list: GiftPersonInfo[], isSearch = false) => {
	// 防御非数组脏数据
	(Array.isArray(list) ? list : []).forEach((person) => {
		if (person.id) {
			map.set(String(person.id), formatMemberOption(person));
		}
	});
};
```

### 规则 3：Playwright 定位选择器兼容复合包裹
在测试用例中定位包含前缀/后缀/数字调节器的输入框时，优先使用：
```js
const input = page.locator('[data-testid="gift-record-ai-input"] input, input[data-testid="gift-record-ai-input"]').first();
```
或者将 `data-testid` 放置在 `<a-form-item>` 上，选择器指定为 `[data-testid="xxx"] input`，确保 100% 稳定击中真实的 `<input>` 元素。
