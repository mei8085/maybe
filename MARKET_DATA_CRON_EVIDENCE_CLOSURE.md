# 定时任务证据闭环报告

## 一、证据清单

### 1.1 已证实的证据

| 证据项 | 代码位置 | 内容摘要 |
|--------|---------|---------|
| **schedule.yml 存在** | `config/schedule.yml` | 定义了 `import_market_data` 任务，cron 表达式为 `0 22 * * 1-5` |
| **sidekiq-cron 依赖** | `Gemfile:35` | `gem "sidekiq-cron"` |
| **sidekiq-cron 版本** | `Gemfile.lock:540` | `sidekiq-cron (2.3.0)` |
| **Sidekiq 配置** | `config/initializers/sidekiq.rb` | 配置了 `reschedule_grace_period`，**未设置时区** |
| **Dockerfile** | `Dockerfile` | 未设置时区环境变量 |

### 1.2 缺失的证据

| 证据项 | 搜索模式 | 结果 |
|--------|---------|------|
| **schedule.yml 加载入口** | `load_from_hash` | ❌ 未找到 |
| **时区显式配置** | `Sidekiq::Cron::Job.timezone` | ❌ 未找到 |
| **TZ 环境变量** | `ENV\[".*TZ.*"\]` | ❌ 未找到 |

---

## 二、部署前提分析

### 2.1 schedule.yml 加载机制

**sidekiq-cron 默认行为**（sidekiq-cron 2.3.0）：
- 若存在 `config/schedule.yml`，gem 会**自动加载**该文件
- 无需显式调用 `load_from_hash`
- 此行为由 sidekiq-cron gem 在初始化时自动触发

**证据**：sidekiq-cron gem 源码（外部依赖）会在 Rails 初始化时扫描 `config/schedule.yml`

### 2.2 时区来源

**Sidekiq Cron 时区优先级**（从高到低）：
1. 显式设置 `Sidekiq::Cron::Job.timezone = "America/New_York"`
2. 环境变量 `ENV['TZ']`
3. 服务器系统时区（`/etc/timezone` 或 `/etc/localtime`）

**当前状态**：第1、2项均缺失，依赖第3项

---

## 三、不同部署前提下的触发时间

### 3.1 前提一：服务器时区 = UTC

| 时段 | Cron 解释 | 纽约时间 | 与收盘时间差 |
|------|----------|---------|-------------|
| 冬令时 (EST) | 22:00 UTC | 17:00 EST（下午5点） | +1小时 ✅ |
| 夏令时 (EDT) | 22:00 UTC | 18:00 EDT（下午6点） | +2小时 ⚠️ |

**前提依据**：
- Docker 镜像通常默认 UTC 时区
- 云服务（AWS、Render等）默认使用 UTC

### 3.2 前提二：服务器时区 = America/New_York

| 时段 | Cron 解释 | 纽约时间 | 与收盘时间差 |
|------|----------|---------|-------------|
| 冬令时 (EST) | 22:00 EST | 22:00 EST（晚上10点） | +6小时 ❌ |
| 夏令时 (EDT) | 22:00 EDT | 22:00 EDT（晚上10点） | +6小时 ❌ |

**前提依据**：需要显式配置服务器时区

### 3.3 前提三：显式设置时区 + 修正 Cron

```ruby
# config/initializers/sidekiq.rb
Sidekiq::Cron::Job.timezone = "America/New_York"
```

```yaml
# config/schedule.yml
import_market_data:
cron: "0 17 * * 1-5"  # 纽约时间下午5点
```

| 时段 | Cron 解释 | 纽约时间 | 与收盘时间差 |
|------|----------|---------|-------------|
| 冬令时 | 17:00 EST | 17:00 EST（下午5点） | +1小时 ✅ |
| 夏令时 | 17:00 EDT | 17:00 EDT（下午5点） | +1小时 ✅ |

---

## 四、结论分栏

### 4.1 已证实

| 结论 | 证据来源 | 可信度 |
|------|---------|-------|
| schedule.yml 定义了 `import_market_data` 任务 | `config/schedule.yml` | 100% |
| cron 表达式为 `0 22 * * 1-5` | `config/schedule.yml:2` | 100% |
| 使用 sidekiq-cron gem 进行定时调度 | `Gemfile:35`, `Gemfile.lock:540` | 100% |
| 仓库中未显式设置 `Sidekiq::Cron::Job.timezone` | `config/initializers/sidekiq.rb` | 100% |
| schedule.yml 会被 sidekiq-cron 自动加载 | sidekiq-cron gem 默认行为 | 95% |

### 4.2 未证实（依赖部署前提）

| 结论 | 依赖前提 | 触发时间 |
|------|---------|---------|
| 任务在纽约时间下午5点执行 | 服务器时区=UTC（冬令时） | ✅ 仅冬令时 |
| 任务在纽约时间下午5点执行 | 服务器时区=UTC（夏令时） | ❌ 下午6点 |
| 任务在纽约时间下午5点执行 | 服务器时区=America/New_York | ❌ 晚上10点 |
| 任务在纽约时间下午5点执行 | 显式设置时区 + 修正 Cron | ✅ 全年 |

---

## 五、代码证据清单

### 5.1 已找到的代码证据

```ruby
# config/schedule.yml - 任务定义
import_market_data:
cron: "0 22 * * 1-5" # 5:00 PM EST / 6:00 PM EDT (NY time) Monday through Friday
class: "ImportMarketDataJob"
queue: "scheduled"
description: "Imports market data daily at 5:00 PM EST (1 hour after market close)"
```

```ruby
# config/initializers/sidekiq.rb - Sidekiq 配置（无时区设置）
Sidekiq::Cron.configure do |config|
config.reschedule_grace_period = 600
end
```

```ruby
# Gemfile - sidekiq-cron 依赖
gem "sidekiq-cron"
```

### 5.2 缺失的代码证据

```ruby
# 缺失：显式加载 schedule.yml（sidekiq-cron 自动加载）
# Sidekiq::Cron::Job.load_from_hash YAML.load_file(File.expand_path("schedule.yml", __dir__))

# 缺失：时区配置
# Sidekiq::Cron::Job.timezone = "America/New_York"

# 缺失：Docker 时区设置
# ENV["TZ"] = "America/New_York"
```

---

## 六、最终结论

### 6.1 当前状态

**仓库内配置**：
- ✅ 任务定义存在
- ✅ sidekiq-cron 依赖存在
- ❌ 时区未配置
- ❌ Cron 表达式未正确设置

**实际触发时间**：
- 取决于服务器系统时区
- 若服务器时区=UTC：冬令时正确（下午5点），夏令时延迟1小时（下午6点）
- 若服务器时区=America/New_York：全年延迟6小时（晚上10点）

### 6.2 建议修复

```ruby
# config/initializers/sidekiq.rb
Sidekiq::Cron::Job.timezone = "America/New_York"
```

```yaml
# config/schedule.yml
import_market_data:
cron: "0 17 * * 1-5"
class: "ImportMarketDataJob"
queue: "scheduled"
description: "Imports market data daily at 5:00 PM ET (1 hour after market close)"
```

### 6.3 验证方法

```ruby
# Rails console 验证
Sidekiq::Cron::Job.timezone  # 应返回 "America/New_York"

job = Sidekiq::Cron::Job.find("import_market_data")
job.cron                      # 应返回 "0 17 * * 1-5"
job.next_time_to_run          # 应显示纽约时间下午5点
```
