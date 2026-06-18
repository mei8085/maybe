# 账户余额快照修正路径说明

## 一、核心概念

### 1.1 余额快照（Balance）

余额快照是每日账户余额的"切片"，存储在 `balances` 表中。每一天一条记录，包含该日的起始余额、各类流入流出、调整项、最终余额等完整信息。

核心字段：

| 字段 | 含义 |
|------|------|
| `start_balance` / `end_balance` | 当日期初 / 期末总余额 |
| `start_cash_balance` / `end_cash_balance` | 当日期初 / 期末现金部分余额 |
| `start_non_cash_balance` / `end_non_cash_balance` | 当日期初 / 期末非现金部分余额（如持仓、本金等） |
| `cash_inflows` / `cash_outflows` | 当日现金流入 / 流出 |
| `non_cash_inflows` / `non_cash_outflows` | 当日非现金流入 / 流出（如买入证券、还本金） |
| `net_market_flows` | 当日市值变动（投资类账户） |
| `cash_adjustments` / `non_cash_adjustments` | 当日现金 / 非现金调整项（手工修正的体现） |
| `flows_factor` | 流向因子（资产=1，负债=-1） |

相关文件：
- [balance.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance.rb)

### 1.2 估值锚点（Valuation）

估值记录（`valuations` 表，通过 `entries` 关联）是余额计算的"锚点"，有三种类型：

| 类型 | 作用 | 相关代码 |
|------|------|----------|
| `opening_anchor` | 期初余额锚点 — 账户起始日的余额基准 | [opening_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/opening_balance_manager.rb) |
| `current_anchor` | 当前余额锚点 — 关联账户（如Plaid）的最新余额 | [current_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/current_balance_manager.rb) |
| `reconciliation` | 对账调整 — 用户手动设置的某一天余额（用于修正差异） | [reconciliation_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconciliation_manager.rb) |

相关文件：
- [valuation.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/valuation.rb)

---

## 二、重算（Recalculation）

### 2.1 重算的触发时机

当任何可能影响余额的操作发生时，系统不会立即重算，而是通过 `sync_later` 调度一个异步同步任务：

```
用户操作 → 修改 Valuation/Transaction/Trade → sync_later → SyncJob → 重算余额
```

触发重算的常见操作：
- 修改期初余额（`set_opening_anchor_balance`）
- 修改当前余额（`set_current_balance`）
- 创建/更新对账记录（`create_reconciliation` / `update_reconciliation`）
- 创建账户（`create_and_sync`）
- 导入交易后同步

相关文件：
- [syncable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/concerns/syncable.rb)
- [anchorable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/anchorable.rb)
- [reconcileable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconcileable.rb)

### 2.2 重算的执行流程

重算由 `Balance::Materializer` 统一调度：

```
Account::Syncer#perform_sync
  ↓
Balance::Materializer#materialize_balances
  ├─ materialize_holdings (持仓物化，投资类账户)
  ├─ calculate_balances (调用计算器计算)
  ├─ persist_balances (批量 upsert 到 balances 表)
  ├─ purge_stale_balances (删除过期快照)
  └─ update_account_info (更新 account 缓存字段)
```

相关文件：
- [materializer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/materializer.rb)
- [syncer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/syncer.rb)

### 2.3 两种计算策略

根据账户类型不同，使用不同的计算方向：

| 策略 | 适用账户 | 计算方向 | 起点 | 终点 |
|------|----------|----------|------|------|
| `:forward`（正向） | 手工账户 | 从旧到新 | opening_anchor | 最后一条记录日期 |
| `:reverse`（反向） | 关联账户（Plaid） | 从新到旧 | current_anchor | opening_anchor |

#### 正向计算（ForwardCalculator）

从期初余额开始，逐日累加交易和调整，推算出每天的期末余额：

```ruby
# 伪代码逻辑
start_balance = opening_anchor_balance
for date in opening_date..latest_date:
  if date has valuation:
    end_balance = valuation.amount  # 锚点日直接使用估值
  else:
    end_balance = start_balance + net_flows  # 非锚点日根据交易推算
  
  adjustments = end_balance - start_balance - net_flows  # 计算调整项
  save_balance_record(date, start_balance, end_balance, adjustments, ...)
  
  start_balance = end_balance  # 下一天的期初 = 当天的期末
```

关键细节：
- 遇到 `reconciliation` 类型的估值时，直接使用该估值作为当日期末余额
- `adjustments` = 期末余额 - 期初余额 - 净流量（即"无法用交易解释的差额"）
- 投资类账户还会计算 `net_market_flows`（市值变动）

相关文件：
- [forward_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/forward_calculator.rb)
- [base_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/base_calculator.rb)

#### 反向计算（ReverseCalculator）

从当前余额（current_anchor）开始，逐日往回推算：

```ruby
# 伪代码逻辑
end_balance = current_anchor_balance
for date in current_date.downto(opening_date):
  if date == opening_anchor_date:
    end_balance = opening_anchor_balance  # 到期初锚点时，使用期初估值
    start_balance = end_balance
  else:
    start_balance = end_balance - net_flows  # 往回推算期初余额
  
  save_balance_record(date, start_balance, end_balance, ...)
  end_balance = start_balance  # 前一天的期末 = 当天的期初
```

注意：反向计算不使用 `reconciliation` 估值，仅使用 `opening_anchor` 和 `current_anchor` 两个锚点。

相关文件：
- [reverse_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/reverse_calculator.rb)

---

## 三、手工修正（Manual Adjustment）

手工修正有三种入口，分别对应不同的场景和锚点类型：

### 3.1 期初余额修正（Opening Balance）

**场景**：设置账户的起始余额（如房产的购入价、账户初始余额）。

**入口**：
- 创建账户时自动设置（`Account.create_and_sync`）
- 导入账户时设置（`AccountImport`）
- 通过 `set_opening_anchor_balance` 方法

**流程**：

```
用户修改期初余额
  ↓
OpeningBalanceManager#set_opening_balance
  ├─ 存在 opening_anchor → 更新 amount / date
  └─ 不存在 → 创建新的 opening_anchor 估值记录
  ↓
sync_later（触发异步重算）
  ↓
重算时，ForwardCalculator 以 opening_anchor 为起点逐日计算
```

**代码要点**：
- 期初余额日期必须早于最早的交易记录日期
- 期初余额会影响后续所有日期的余额计算（正向传播）

相关文件：
- [opening_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/opening_balance_manager.rb)

### 3.2 当前余额修正（Current Balance）

**场景**：用户在账户编辑页直接修改"当前余额"。

**入口**：
- 账户更新接口（`AccountableResource#update`）
- 资产余额更新（`PropertiesController#update_balances`）

**流程**：

```
用户修改当前余额
  ↓
CurrentBalanceManager#set_current_balance
  ├─ 关联账户 → 更新 current_anchor 估值
  └─ 手工账户 → 分两种策略：
      ├─ 有对账记录 → 追加一条 reconciliation 估值（价值追踪策略）
      └─ 无对账记录 → 调整 opening_balance（交易调整策略）
  ↓
更新 account.balance 缓存字段（立即可见）
  ↓
sync_later（触发异步重算）
```

**两种自动策略（手工账户）**：

1. **价值追踪策略**（有对账记录时）：
   - 在今日追加一条 reconciliation 估值
   - 适用于用户主要通过对账来跟踪账户价值的场景（如房产、投资）
   - 每次修改余额都会新增一条"对账"记录

2. **交易调整策略**（无对账记录时）：
   - 调整 opening_balance 的数值（"回推"到期初）
   - 适用于用户主要通过交易来跟踪余额的场景（如现金账户）
   - 修改余额不会产生额外的对账记录

相关文件：
- [current_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/current_balance_manager.rb)

### 3.3 对账调整（Reconciliation）

**场景**：用户在特定日期设置余额，用于修正与实际值的差异。

**入口**：`create_reconciliation` / `update_reconciliation`

**流程**：

```
用户设置某日对账余额
  ↓
ReconciliationManager#reconcile_balance
  ├─ 存在该日估值 → 更新 amount
  └─ 不存在 → 创建新的 reconciliation 估值记录
  ↓
sync_later（触发异步重算）
  ↓
重算时，ForwardCalculator 遇到该日 valuation，直接使用其金额作为期末余额
  ↓
当日的 adjustments = 期末余额 - 期初余额 - 净流量
（即：对账值与"按交易推算值"的差额，记入 adjustments）
```

相关文件：
- [reconciliation_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconciliation_manager.rb)

---

## 四、展示值更新（Display Value）

### 4.1 展示组件

余额快照的展示由 `UI::Account::BalanceReconciliation` 组件负责，它根据账户类型组织不同的展示项目。

展示结构（以默认类型为例）：

```
Start balance         $1,000.00   ← balance.start_balance
Net cash flow           +$50.00   ← cash_inflows - cash_outflows
End balance          $1,050.00   ← （如果有调整项才显示）
Adjustments            -$10.00   ← cash_adjustments + non_cash_adjustments
Final balance        $1,040.00   ← balance.end_balance
```

### 4.2 不同账户类型的展示差异

| 账户类型 | 展示重点 |
|----------|----------|
| 储蓄/其他资产/负债 | 起始余额 + 净现金流 + 调整项 + 最终余额 |
| 信用卡 | 起始余额 + 消费 + 还款 + 调整项 + 最终余额 |
| 投资 | 起始余额 + 现金变动 + 持仓买卖 + 市价变动 + 调整项 + 最终余额 |
| 贷款 | 起始本金 + 本金变动 + 调整项 + 最终本金 |
| 房产/车辆 | 起始价值 + 价值变动 + 调整项 + 最终价值 |
| 加密货币 | 起始余额 + 买入 + 卖出 + 市价变动 + 调整项 + 最终余额 |

相关文件：
- [balance_reconciliation.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/components/UI/account/balance_reconciliation.rb)
- [balance_reconciliation.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/components/UI/account/balance_reconciliation.html.erb)

---

## 五、三者衔接的完整链路

### 5.1 完整流程图

```
┌──────────────────────────────────────────────────────────────┐
│                     用户操作层                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │ 修改期初余额 │  │ 修改当前余额 │  │ 创建对账调整     │    │
│  └──────┬──────┘  └──────┬───────┘  └────────┬─────────┘    │
└─────────┼────────────────┼───────────────────┼──────────────┘
          │                │                   │
          ▼                ▼                   ▼
┌──────────────────────────────────────────────────────────────┐
│                    估值锚点层（Valuation）                    │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐  │
│  │opening_anchor│  │current_anchor │  │  reconciliation  │  │
│  │  (期初锚点)   │  │  (当前锚点)    │  │   (对账锚点)      │  │
│  └──────┬───────┘  └───────┬───────┘  └────────┬─────────┘  │
└─────────┼──────────────────┼───────────────────┼────────────┘
          │                  │                   │
          └──────────┬───────┴───────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│                    异步同步层（Sync）                         │
│  sync_later → SyncJob → perform_sync → Materializer          │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    重算层（Calculator）                       │
│  ForwardCalculator / ReverseCalculator                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 锚点日：直接使用 valuation.amount 作为期末余额       │    │
│  │ 非锚点日：期初余额 + 净流量 = 期末余额（推算）        │    │
│  │ 调整项 = 期末余额 - 期初余额 - 净流量                │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                   快照存储层（Balance）                      │
│  balances 表：每天一条记录，包含完整的收支明细               │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                   展示层（BalanceReconciliation）             │
│  根据账户类型，组织不同的展示项目呈现给用户                   │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 关键衔接点说明

1. **手工修正 → 重算**
   - 手工修正操作本身只修改 `Valuation`（估值锚点）
   - 然后通过 `sync_later` 触发异步重算
   - 重算时，计算器读取所有估值锚点，在锚点之间进行插值推算

2. **调整项（adjustments）的产生**
   - adjustments 不是用户直接输入的
   - 它是重算时的"计算副产品"：期末余额 - 期初余额 - 净流量
   - 在有估值锚点的日期，adjustments 反映了"锚点值"与"按交易推算值"之间的差异
   - 在没有估值锚点的日期，如果所有变动都能被交易解释，adjustments 通常为 0

3. **展示值的及时性**
   - `account.balance` 缓存字段会在修改时立即更新（立即可见）
   - 但历史每日余额快照（balances 表）需要等待异步重算完成才会更新
   - 如果重算尚未完成，展示组件读取的可能是旧快照数据

4. **两种计算策略的选择**
   - 手工账户用 forward：因为 opening_anchor 是可信起点，交易是已知的
   - 关联账户用 reverse：因为 current_anchor（来自Plaid）是最可信的最新值，从后往前推更准确

---

## 六、关键代码索引

| 模块 | 文件 |
|------|------|
| 余额模型 | [balance.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance.rb) |
| 余额物化器 | [materializer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/materializer.rb) |
| 正向计算器 | [forward_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/forward_calculator.rb) |
| 反向计算器 | [reverse_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/reverse_calculator.rb) |
| 基础计算器 | [base_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/base_calculator.rb) |
| 期初余额管理 | [opening_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/opening_balance_manager.rb) |
| 当前余额管理 | [current_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/current_balance_manager.rb) |
| 对账管理 | [reconciliation_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconciliation_manager.rb) |
| 账户同步器 | [syncer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/syncer.rb) |
| 同步模块 | [syncable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/concerns/syncable.rb) |
| 锚点模块 | [anchorable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/anchorable.rb) |
| 对账模块 | [reconcileable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconcileable.rb) |
| 展示组件 | [balance_reconciliation.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/components/UI/account/balance_reconciliation.rb) |
| 估值模型 | [valuation.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/valuation.rb) |
| 账户模型 | [account.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account.rb) |
