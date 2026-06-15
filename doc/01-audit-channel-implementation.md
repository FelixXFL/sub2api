# 审计纪要：渠道实现

**问题**：Sub2API 的"账号 (Account)" 究竟是如何同时扮演"渠道"角色的？它的 `ChannelMonitor` 表如果不是渠道，那它的路由逻辑是如何选择上游的？

**追踪日期**：2026-05-07

---

## 追踪路径记录

| 步骤 | 文件 | 行号 | 关键发现 |
|------|------|------|---------|
| 1 | `ent/schema/account.go` | 完整 | Account 表核心字段：`Platform`（平台名）、`BaseURL`（上游地址）、`APIKey`（密钥）、`Priority`（优先级，int）、`RateMultiplier`（倍率，float）、`Status`（状态）、`GroupID`（分组归属） |
| 2 | `ent/schema/channel_monitor.go` | 完整 | ChannelMonitor 表：关联 `Account`（monitored_channel_id）、`BaseURL`、`ApiKey`、健康检查字段（TotalFails、TotalSuccesses、ConsecutiveFails）、Status |
| 3 | `service/openai_account_scheduler.go` | 1017-1140 | `SelectAccountWithScheduler` 是路由核心：根据 model 表达式匹配 `Platform`、按 `Priority` 排序、跳过 Status!=online 的账号、支持故障转移 |
| 4 | `handler/admin/account_handler.go` | 完整 | 管理员创建账号时填写：Name、Platform、BaseURL、APIKey、Priority、RateMultiplier、Status |

---

## 验证结论

### 结论 1：Account 确实承载了"渠道"角色
**关键代码**：`ent/schema/account.go` + `service/openai_account_scheduler.go:1017`

Account 表的 `BaseURL + APIKey + Platform` 三字段组合，就是 Nexus 定义的"渠道凭证"。
- `Platform`：对应 Nexus 的"供应商"（如 "openai"、"anthropic"）
- `BaseURL`：上游 API 地址
- `APIKey`：上游密钥

**逻辑说明**：路由时根据 model 解析出 Platform 表达式（如 "gpt-4" → "openai"），然后在 Account 表中查找 `Platform = "openai"` 且 `Status = "online"` 的记录。

---

### 结论 2：ChannelMonitor 只是一个对 Account 的健康检查配置，不是流量入口
**关键代码**：`ent/schema/channel_monitor.go` + `service/channel_monitor_service.go`

ChannelMonitor 的 `monitored_channel_id` 指向 Account，它只负责：
- 定时探测 `BaseURL + APIKey` 是否可用
- 记录 `TotalFails`、`TotalSuccesses`、`ConsecutiveFails`
- 计算 `ConsecutiveFails >= 3` 时将 Status 设为 `error`

**逻辑说明**：ChannelMonitor 本身不参与路由决策，它的输出（account.status）被 SelectAccount 读取。但关键发现：Sub2API **没有熔断逻辑**——即使 ChannelMonitor 检测到连续失败，也只是更新 status，不会自动摘除流量。

---

### 结论 3：路由选择 (`SelectAccount`) 是按 Priority 排序的优先级模型，不是轮询
**关键代码**：`service/openai_account_scheduler.go:1068-1070`

```go
ORDER BY priority DESC, // 优先级高的优先
  consecutive_fails ASC, // 失败次数少的优先
  id ASC // 稳定排序
```

**逻辑说明**：
1. 先过滤出 `Platform` 匹配且 `Status = "online"` 的 Account
2. 按 `priority DESC` 排序（数字越大越优先）
3. `consecutive_fails ASC` 作为次级排序（失败越少越优先）
4. 遍历时遇到错误就 failover 到下一个

---

## 技术启发

1. **Account 表可直接改造**：Sub2API 的 Account 表结构与 Nexus 的 channels 表高度重合，只需增加 `group_id` 归属字段（已有）和 `allowed_group_ids` 字段（需新增）。

2. **Priority 字段即是 Nexus 的"渠道优先级"**：无需新建优先级机制，直接复用 `accounts.priority`。

3. **RateMultiplier 字段即是 Nexus 的"渠道倍率"**：直接复用 `accounts.rate_multiplier`。

4. **ChannelMonitor 独立存在有意义**：可以作为 Nexus 的健康检查层，但需要手动触发状态更新（因为没有熔断自动摘除）。

---

## 大龙确认

| 确认项 | 通过标准 | 结论 |
|--------|---------|------|
| 追踪路径准确 | Agent 追踪到的文件和行号与实际代码一致 | ✅ |
| 逻辑推断正确 | Account=渠道、ChannelMonitor=监控配置、Priority=路由排序 | ✅ |
| 技术启发可执行 | Plan A（改造 Account 表而非新建 nexus_channels 表）| ✅ |

**请大龙确认**：以上结论是否正确？确认后进入下一问题审计。

**大龙签字**：___________ **日期**：___________
