---
title: TechSpec - 收支核算与预付卡流水设计
aliases: [个人财务Spec, 资产账户设计]
created: 2026-09-16
updated: 2026-09-16
status: active
tags: [techspec, finance, rbac, fullstack]
---
# 🛠️ TechSpec - 收支核算与预付卡流水设计

返回功能说明：[[02-Features/05-SelfFinance-个人资产财务/PRD-个人资产与收支记账功能说明|PRD-个人资产与收支记账功能说明]]

---

## 1. 实体领域模型与核心表

```mermaid
erDiagram
    FINANCE_ACCOUNT_INFO_T ||--o{ FINANCE_ACCOUNT_RECORD_INFO_T : "账户产生多笔流水"

    FINANCE_ACCOUNT_INFO_T {
        bigint id PK "账户主键ID"
        bigint user_id "所属用户ID"
        varchar account_name "账户名称(如工商银行卡)"
        varchar account_type "账户类型(BANK/WECHAT/ALIPAY/PREPAID)"
        decimal balance "当前账户实时余额"
        varchar currency "币种(CNY/USD)"
        int status "状态 1正常 0禁用"
    }

    FINANCE_ACCOUNT_RECORD_INFO_T {
        bigint id PK "流水明细ID"
        bigint account_id FK "关联账户ID"
        bigint user_id "所属用户ID"
        varchar record_type "流水类型(INCOME/EXPENSE/TRANSFER)"
        decimal amount "交易金额"
        decimal balance_after "变动后账户余额"
        bigint category_id "收支类别ID"
        datetime transaction_time "交易发生时间"
        varchar remark "备注"
    }
```

---

## 2. 核心技术规格

### 2.1 账户转账与事务防并发处理

```java
@Transactional(rollbackFor = Exception.class)
public boolean transfer(TransferVo req, Long userId) {
    // 1. 悲观锁锁定源账户与目标账户，按账户ID从小到大锁定避免死锁
    Long firstId = Math.min(req.getFromAccountId(), req.getToAccountId());
    Long secondId = Math.max(req.getFromAccountId(), req.getToAccountId());
    
    financeAccountMapper.selectByIdForUpdate(firstId);
    financeAccountMapper.selectByIdForUpdate(secondId);
    
    // 2. 检查余额充足
    FinanceAccountInfo from = financeAccountMapper.selectById(req.getFromAccountId());
    if (from.getBalance().compareTo(req.getAmount()) < 0) {
        throw new BusinessException("账户余额不足");
    }
    
    // 3. 执行双边流水扣减与增加
    financeAccountMapper.updateBalance(req.getFromAccountId(), req.getAmount().negate());
    financeAccountMapper.updateBalance(req.getToAccountId(), req.getAmount());
    
    // 4. 插入两笔对称流水日志
    insertRecord(from, req.getAmount().negate(), "TRANSFER_OUT");
    insertRecord(to, req.getAmount(), "TRANSFER_IN");
    return true;
}
```

### 2.2 数据权限与多租户

所有个人财务与流水操作必须在 Service 层面强制传入 `userId = SecurityUtils.getLoginUserId()` 进行硬编码约束或使用 `@DataPermission(tableAlias = "f", userField = "user_id")` 进行 SQL 级行过滤，防止越权操作他人物料。

### 2.3 财务收支多维分析双柱核算架构

- **聚合计算规范**：日/月度收支走势分析中，收入（`income_amount`）与支出（`expense_amount`）独立通过 `CASE income_and_expenses WHEN 'income'/'expense' THEN amount ELSE 0 END` 聚合求和，解耦正负抵消。
- **前后端契约**：`AnalysisVo` 提供独立字段 `BigDecimal incomeAmount` 与 `BigDecimal expenseAmount`，保留 `amount` 保证历史向下兼容。
- **前端可视化呈现**：`barChart.vue` 提供双柱分组对比（收入绿色 `#10b981`、支出暖橙 `#f97316`），全面反映同一时间粒度下的资金流入与流出真实全貌。

### 2.4 历史单表随礼菜单下线与清理规范 (2026-09-19)

- **数据库级联清理**：
  - `t_menu_info`：逻辑删除主菜单（`id = 1810856881968091138`，`name: personalGift`，`path: /selfFinance/personalGift`）与详情菜单（`id = 1810856882504962050`，`name: personalGiftDetail`），置 `is_delete = 1`；
  - `t_permission_info`：逻辑删除权限项 `selfFinance:personalGift`（`id = 1810856883473846273`）；
  - `t_role_permission_info`：逻辑删除对应角色绑定记录（IDs: `28`, `29`）。
- **Redis 缓存失效与预热**：
  - 级联清除 `LoginKey:login:in:menu_all_tree` 全量共享菜单树；
  - 遍历清除各在线用户的 `LoginKey:login:in:permission_context:*` 权限上下文，保证用户刷新页面即刻生效。
- **前端废弃视图源码移除**：
  - 彻底删除 `alex_miaosha_front/src/views/finance/personalGift/` 目录，消除无用的冗余打包体积与历史维护负担。

### 2.5 零花钱月度预算与分类消费核算 (Ponytail极简/0-Cron Job架构 - 2026-10-08)

- **核心需求与原则**：
  - 用户按月设置零花钱额度（`budget_amount`）并可选限定生效类别列表（`category_codes`，如仅计入"餐饮,零食"等高频日常消费）；
  - 严格践行 **Ponytail** 哲学：坚决不引入每月1号0点数据复制的定时任务（0-Cron Job），拒绝无意义的空跑批与脏数据膨胀；
  - **读链路（Query-time Fallback）**：查询指定月份时，若当月无独立设置，则通过 `selectLatestBefore` 自动回溯继承历史最近月份的配置（`is_inherited = true`）；若无任何历史记录则预算为0；
  - **写链路（Lazy Snapshot）**：用户调整当月预算时，仅更新/插入当月的 `finance_budget_info` 记录（`year_month = :currentMonth`），历史月份快照绝对隔离不受污染；
  - **精准支出抵扣与防越权**：
    - 联动 `finance_info`（`financeManage`）的明细数据，支出按 `type_code IN (...)` 精准统计，自动排除内部转账（`type_code = '转账'`）；
    - 所有 ID 交互前端强保持 `string`、后端通过 `@JsonSerialize(using = Long2StringSerializer.class)` 序列化，杜绝前端精度丢失；
    - 接口层提供 `GET /finance-budget/status` 与 `POST /finance-budget/save`，并在 PC 端（`alex_miaosha_front`）与移动端（`alex_miaosha_mobile`）完成双端联动呈现；
    - **归属人（`belong_to`）双重防崩兜底契约**：前端未选中特定筛选人时优先自动传递当前登录用户 ID（`userStore.getUserInfo?.id`）；后端请求 DTO（`FinanceBudgetSaveReq`）解绑 `@NotNull` 硬校验，当入参为空时自动由 Service 拦截器从 HTTP 上下文 Token（通过 `userUtils.getUserId()` 内聚解析）智能解析，彻底杜绝 400 校验阻断。


