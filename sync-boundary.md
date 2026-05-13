# 账户同步与家庭范围同步的职责边界与协作

## 一、架构概览

系统采用了**层次化同步模型**，由上到下分为三层同步单元，通过父子 Sync 记录关联：

```
Family (顶层)
  ├── PlaidItem (中间层，Plaid 连接)
  │     └── Account (底层，具体账户)
  └── Manual Account (底层，手动账户)
```

所有可同步对象都通过 `Syncable` 混入获得同步能力。

---

## 二、职责划分

### 1. 账户级同步 (Account::Syncer)

**定位**：单个账户的数据处理与计算层

**核心职责**：

| 任务 | 说明 | 实现位置 |
|------|------|----------|
| 余额物化 | 根据账户类型（linkable vs manual）选择 `reverse` 或 `forward` 策略生成历史余额序列 | `app/models/account/syncer.rb:19-22` |
| 市场数据补充 | 导入该账户所需的汇率和证券价格（图表展示依赖） | `app/models/account/syncer.rb:31-36` |
| 后同步转账匹配 | 触发家庭级转账自动匹配（但实际逻辑在 Family） | `app/models/account/syncer.rb:14-16` |

**不负责**：
- 不处理 Plaid API 数据拉取（那是 PlaidItem 的职责）
- 不跨账户操作（除了触发转账匹配）
- 不维护家庭级状态

**输入边界**：
- 仅接收 `sync` 对象（包含窗口日期）
- 从数据库读取该账户已有的 entries/transactions

**输出边界**：
- 写入/更新 `balances` 表
- 可能写入 `exchange_rates` 和 `security_prices`（通过市场数据导入器）

---

### 2. 家庭级同步 (Family::Syncer)

**定位**：协调者与家庭范围业务层

**核心职责**：

| 任务 | 说明 | 实现位置 |
|------|------|----------|
| 试用状态同步 | 保持家庭试用状态的最终一致性 | `app/models/family/syncer.rb:10` |
| 规则调度 | 为家庭的每个规则触发异步应用 | `app/models/family/syncer.rb:13-15` |
| 子同步编排 | 调度 `PlaidItem` 和手动账户的子同步 | `app/models/family/syncer.rb:18-20` |
| 后同步转账匹配 | 跨账户检测并自动标记转账 | `app/models/family/syncer.rb:23-25` |

**不负责**：
- 不直接操作任何账户的数据
- 不执行具体余额计算
- 不调用外部 API

**输入边界**：
- 接收 `sync` 对象，其中的窗口日期会向下传递给子同步

**输出边界**：
- 通过 `sync_later` 创建子 Sync 记录
- 触发 `RuleJob` 异步任务
- 更新 `family.trial_status`

---

### 3. Plaid 连接同步 (PlaidItem::Syncer)

**定位**：外部数据源桥接层

**核心职责**：

| 任务 | 说明 | 实现位置 |
|------|------|----------|
| Plaid 数据拉取 | 调用 Plaid API 获取最新的账户、交易等数据 | `app/models/plaid_item/syncer.rb:11` |
| 数据处理 | 将 Plaid 原始数据转换为内部领域对象 | `app/models/plaid_item/syncer.rb:13` |
| 账户同步调度 | 为该 PlaidItem 关联的所有账户触发账户级同步 | `app/models/plaid_item/syncer.rb:16-20` |

---

## 三、协作方式与数据流

### 3.1 完整同步流程（以家庭同步为例）

```
用户触发 sync_later
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  Family::Syncer.perform_sync                            │
│  ├─ sync_trial_status!      (家庭状态更新)              │
│  ├─ 为每个规则调用 rule.apply_later (异步)              │
│  └─ 对 child_syncables 调用 sync_later                  │
│     ├─ PlaidItem.sync_later                             │
│     └─ ManualAccount.sync_later                         │
└─────────────────────────────────────────────────────────┘
       │
       ├────────────────────┬─────────────────────────────┐
       ▼                    ▼                             │
┌───────────────┐   ┌───────────────┐                    │
│PlaidItem Sync │   │ Manual Account│                    │
│               │   │    Sync       │                    │
│ import_latest │   │ import_market │                    │
│  _plaid_data  │   │  _data        │                    │
│ process_accou │   │ materialize_  │                    │
│ nts           │   │ balances      │                    │
│               │   │               │                    │
│ schedule_acc  │   │               │                    │
│ ount_syncs ───┼──►│               │                    │
└───────────────┘   │               │                    │
                    └───────────────┘                    │
                                    │                    │
                                    ▼                    │
                         ┌───────────────────┐           │
                         │ Account Sync 完成 │           │
                         │ 后同步：          │           │
                         │ auto_match_trans- │           │
                         │ fers!             │           │
                         └───────────────────┘           │
                                    │                    │
                                    ▼                    │
                         ┌───────────────────┐           │
                         │ Family Sync 最终 │◄──────────┘
                         │ 化(等待所有子同  │
                         │ 步完成后)         │
                         │ 后同步：          │
                         │ auto_match_trans- │
                         │ fers!             │
                         └───────────────────┘
```

### 3.2 父子同步的状态传播机制

通过 `Sync.finalize_if_all_children_finalized` 实现自底向上的状态传播：

1. 子同步完成 → 检查自身所有子节点是否完成
2. 若自身子节点都完成 → 标记自身为 complete/failed
3. 递归向上调用 `parent.finalize_if_all_children_finalized`
4. 最终传播到顶层 Family Sync

关键代码：`app/models/sync.rb:83-104`

### 3.3 数据传递的边界

**边界原则：Sync 记录是唯一的"控制面"传递介质，数据通过数据库共享**

| 维度 | 传递方式 | 边界 |
|------|----------|------|
| **同步窗口** | `window_start_date` / `window_end_date` 字段 | 从 Family 同步创建时设置，向下传递给子同步 |
| **同步状态** | `status` 字段 + `finalize_if_all_children_finalized` | 子同步状态通过 `children` 关联影响父同步 |
| **错误信息** | `error` 字段 | 子同步失败 → 父同步 `has_failed_children?` → 父同步标记为 failed |
| **业务数据** | **不通过 Sync 对象传递** | 通过数据库表（accounts, entries, transactions 等）共享，各同步层读写自己负责的表 |

**重要设计决策**：
- 没有使用事件总线或消息队列传递业务数据
- 各同步器之间通过数据库实现"共享内存"通信
- 只有控制信息（窗口、状态、父子关系）通过 Sync 模型传递

---

## 四、职责重叠与处理策略

### 4.1 转账自动匹配（后同步阶段）

**现象**：
- `Account::Syncer.perform_post_sync` → `account.family.auto_match_transfers!`
- `Family::Syncer.perform_post_sync` → `family.auto_match_transfers!`

**原因分析**：
这是一个**最终一致性**策略。转账匹配需要跨账户视图，理论上只应该在所有账户同步完成后执行一次（在 Family 后同步）。

但账户同步器也触发一次，有两个可能原因：
1. 单个账户也可能单独触发同步（非家庭级），此时需要在账户级后同步触发匹配
2. 增加幂等调用作为冗余，确保匹配总是被触发

**边界清晰化建议**：
- 账户同步器中的调用应理解为："该账户完成后，尝试看看有没有新的可匹配转账"
- 家庭同步器中的调用才是："所有账户都完成了，做一次完整匹配"

### 4.2 手动账户 vs 链接账户

| 类型 | 同步触发源 | 流程差异 |
|------|-----------|----------|
| 手动账户 | Family 同步直接调度 | 直接进入 Account::Syncer，余额策略 = forward |
| 链接账户 | 通过 PlaidItem 间接调度 | 先经过 PlaidItem::Syncer 拉取数据，再进入 Account::Syncer，余额策略 = reverse |

---

## 五、关键协作点总结

```
┌──────────────────────────────────────────────────────────────┐
│                      家庭同步入口                            │
│  Family.sync_later()                                          │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Family::Syncer.perform_sync (协调层)                   │ │
│  │  ├─ 更新试用状态 (状态同步)                             │ │
│  │  ├─ 调度规则应用 (异步任务)                             │ │
│  │  └─ 调度子同步                                         │ │
│  │     ├─ PlaidItem.sync_later ──► PlaidItem::Syncer     │ │
│  │     │                      ├─ 拉取 Plaid 数据         │ │
│  │     │                      ├─ 处理领域对象            │ │
│  │     │                      └─ 调度 Account 同步 ──┐   │ │
│  │     └─ ManualAccount.sync_later ──────────────────┤   │ │
│  │                                                   │   │ │
│  └───────────────────────────────────────────────────┼───┘ │
│                                                      ▼     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Account::Syncer.perform_sync (计算层)                  │ │
│  │  ├─ 补充市场数据 (汇率/证券价格)                        │ │
│  │  └─ 物化余额 (根据策略生成历史序列)                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                      │     │
│                                                      ▼     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  后同步阶段 (post-sync)                                 │ │
│  │  ├─ Account 级: auto_match_transfers! (部分匹配)       │ │
│  │  └─ Family 级: auto_match_transfers! (完整匹配)        │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 六、边界检查表

| 问题 | 答案 |
|------|------|
| 谁负责调用外部 API？ | PlaidItem::Syncer |
| 谁负责计算余额历史？ | Account::Syncer |
| 谁负责跨账户业务逻辑？ | Family::Syncer（通过 post-sync） |
| 谁负责调度子同步？ | Family::Syncer + PlaidItem::Syncer |
| 同步窗口如何传递？ | 通过 Sync 模型的 `window_start_date` / `window_end_date` 字段 |
| 业务数据如何传递？ | **不传递**，通过数据库表共享 |
| 失败如何向上传播？ | 通过 `Sync.finalize_if_all_children_finalized` 自底向上传播 |
| 哪些层有 post-sync？ | 所有层都有，但 Family 和 Account 有实际业务逻辑 |

---

## 七、代码位置索引

| 组件 | 文件路径 |
|------|----------|
| 同步状态机与父子管理 | `app/models/sync.rb` |
| 可同步接口 | `app/models/concerns/syncable.rb` |
| 家庭同步器 | `app/models/family/syncer.rb` |
| 账户同步器 | `app/models/account/syncer.rb` |
| PlaidItem 同步器 | `app/models/plaid_item/syncer.rb` |
| 同步任务入口 | `app/jobs/sync_job.rb` |
| 家庭模型 | `app/models/family.rb` |
| 账户模型 | `app/models/account.rb` |
| PlaidItem 模型 | `app/models/plaid_item.rb` |
