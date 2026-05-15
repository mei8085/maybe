# 批量更新字段影响一致性分析报告

## 字段影响矩阵核对结果

### 核对结论汇总

| 字段 | 是否支持批量更新 | 是否影响余额计算 | 真实代码是否触发同步 | 一致性 |
|-----|----------------|----------------|---------------------|-------|
| `date` | ✅ 是 | ✅ 是（核心字段） | ❌ 未触发 | ❌ 不一致 |
| `notes` | ✅ 是 | ❌ 否 | - | ✅ 一致 |
| `category_id` | ✅ 是 | ❌ 否 | - | ✅ 一致 |
| `merchant_id` | ✅ 是 | ❌ 否 | - | ✅ 一致 |
| `tag_ids` | ✅ 是 | ❌ 否 | - | ✅ 一致 |

---

## 详细逐条核对

### 1. `date` - 交易日期

**代码实现位置**：`app/models/entry.rb:75`
```ruby
bulk_attributes = {
  date: bulk_update_params[:date],  # 支持批量更新
  ...
}
```

**影响判断**：✅ **严重影响余额计算**

余额计算的核心逻辑是**按日期**进行迭代计算（`ForwardCalculator#calculate` / `ReverseCalculator#calculate`）：
- 交易日期变动会直接改变其所在日期的现金流计算
- 日期向后移动：原日期少一笔，新日期多一笔
- 日期向前移动：原日期多一笔，新日期少一笔
- 影响范围：从 `min(原日期, 新日期)` 到当前的**所有日期余额**

**真实代码行为**：❌ **未触发同步**

对比参照：
- 单条交易更新（`app/controllers/transactions_controller.rb:88`）：
  ```ruby
  @entry.sync_account_later  # 正确触发
  ```
- 批量删除（`app/controllers/transactions/bulk_deletions_controller.rb:4`）：
  ```ruby
  destroyed.map(&:account).uniq.each(&:sync_later)  # 正确触发
  ```
- 批量更新：**无任何同步触发代码**

**不一致结论**：`date` 字段批量更新后**缺少同步触发**，会导致余额曲线与实际交易数据不一致。

---

### 2. `notes` - 备注

**代码实现位置**：`app/models/entry.rb:76`
```ruby
notes: bulk_update_params[:notes],
```

**影响判断**：❌ 不影响余额计算

纯文本字段，仅用于用户备注，不参与任何财务计算。

**真实代码行为**：无需触发同步。

**一致性结论**：✅ 一致。

---

### 3. `category_id` - 分类ID

**代码实现位置**：`app/models/entry.rb:78`
```ruby
entryable_attributes: {
  category_id: bulk_update_params[:category_id],
  ...
}
```

**影响判断**：❌ 不影响余额计算

仅用于支出/收入报表分类，不影响账户余额的计算逻辑。余额计算仅依赖：
- 日期
- 金额
- 账户归属

**真实代码行为**：无需触发同步。

**一致性结论**：✅ 一致。

---

### 4. `merchant_id` - 商户ID

**代码实现位置**：`app/models/entry.rb:79`
```ruby
merchant_id: bulk_update_params[:merchant_id],
```

**影响判断**：❌ 不影响余额计算

仅用于商户分析和统计，不参与任何财务计算。

**真实代码行为**：无需触发同步。

**一致性结论**：✅ 一致。

---

### 5. `tag_ids` - 标签ID列表

**代码实现位置**：`app/models/entry.rb:80`
```ruby
tag_ids: bulk_update_params[:tag_ids]
```

**影响判断**：❌ 不影响余额计算

仅用于用户自定义筛选和分组，不影响财务计算。

**真实代码行为**：无需触发同步。

**一致性结论**：✅ 一致。

---

## 风险优先级结论

### P0 级风险：`date` 批量更新不同步

**风险等级**：🔴 P0 - 严重

**风险依据**：
1. **数据不一致**：批量修改交易日期后，余额曲线完全错误，但用户无法感知
2. **影响范围广**：所有选中交易的账户都会受到影响，且影响从最早日期开始的所有历史余额
3. **难以排查**：用户发现余额不对时，难以追溯到是某次批量更新导致的
4. **修复困难**：必须手动触发账户同步或单条编辑每条交易才能修复

**对比参照（正确实现）**：

| 操作 | 触发同步方式 |
|-----|------------|
| 单条交易创建 | `@entry.sync_account_later` |
| 单条交易更新 | `@entry.sync_account_later` |
| 单条交易删除 | `@entry.sync_account_later` |
| 批量删除交易 | 对每个账户 `sync_later` |
| 批量更新日期 | ❌ **无** |

**代码缺陷位置**：
```ruby
# app/controllers/transactions/bulk_updates_controller.rb:5-11
def create
  updated = Current.family
                   .entries
                   .where(id: bulk_update_params[:entry_ids])
                   .bulk_update!(bulk_update_params)  # 批量更新

  redirect_back_or_to transactions_path, notice: "#{updated} transactions updated"
  # ❌ 缺失同步触发逻辑
end
```

**最小影响修复方案**：
```ruby
def create
  entries = Current.family.entries.where(id: bulk_update_params[:entry_ids])
  updated = entries.bulk_update!(bulk_update_params)
  
  # 如果更新包含日期，触发同步
  if bulk_update_params[:date].present?
    entries.map(&:account).uniq.each(&:sync_later)
  end
  
  redirect_back_or_to transactions_path, notice: "#{updated} transactions updated"
end
```

---

### 其他字段：无风险

`notes`、`category_id`、`merchant_id`、`tag_ids` 批量更新不影响余额计算，无需触发同步，当前实现正确。

---

## 完整修复建议

### 修复目标

批量更新 `date` 字段时，触发受影响账户的余额重算。

### 推荐实现方案

**方案 A：Controller 层触发（推荐，侵入最小）**

```ruby
# app/controllers/transactions/bulk_updates_controller.rb
def create
  entries = Current.family.entries.where(id: bulk_update_params[:entry_ids])
  updated = entries.bulk_update!(bulk_update_params)
  
  # 仅当日期变更时触发同步，避免不必要的计算
  if bulk_update_params[:date].present?
    affected_accounts = entries.map(&:account).uniq
    affected_accounts.each do |account|
      account.sync_later(window_start_date: bulk_update_params[:date])
    end
  end
  
  redirect_back_or_to transactions_path, notice: "#{updated} transactions updated"
end
```

**方案 B：Model 层触发**

```ruby
# app/models/entry.rb
def bulk_update!(bulk_update_params)
  # ... 现有逻辑 ...
  
  if bulk_update_params[:date].present?
    all.map(&:account).uniq.each do |account|
      account.sync_later(window_start_date: bulk_update_params[:date])
    end
  end
  
  all.size
end
```

**方案优势对比**：
| 方案 | 优势 | 劣势 |
|-----|------|------|
| Controller 层 | 职责清晰，与 bulk_deletions 模式一致 | 需要处理跨 Controller 调用场景 |
| Model 层 | 任何调用都能保证一致性 | Model 层依赖太多业务逻辑 |

**推荐采用方案 A**，与 `bulk_deletions_controller.rb` 的实现模式保持一致。

---

## 附录：同步窗口计算逻辑回顾

### 单条更新窗口

```ruby
# app/models/entry.rb:46-49
def sync_account_later
  sync_start_date = [ date_previously_was, date ].compact.min unless destroyed?
  account.sync_later(window_start_date: sync_start_date)
end
```

### 批量更新窗口（建议实现）

```ruby
# 所有交易统一更新到同一个目标日期
target_date = bulk_update_params[:date]

# 窗口起始 = min(所有交易的原日期, 目标日期)
min_original_date = entries.minimum(:date)
window_start = [min_original_date, target_date].min

# 对每个受影响账户同步
accounts.each { |a| a.sync_later(window_start_date: window_start) }
```

---

*分析日期：2025-07-24*
*代码版本：基于 schema version 2025_07_24_115507*