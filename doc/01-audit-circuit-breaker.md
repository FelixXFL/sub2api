# 审计纪要：熔断机制

**问题**：Sub2API 明确没有独立的熔断逻辑，那 `channel_monitors` 到底做了什么？网关在选择账号时，会参考它的状态吗？

**追踪日期**：2026-05-07

---

## 追踪路径记录

| 步骤 | 文件 | 行号 | 关键发现 |
|------|------|------|---------|
| 1 | `handler/admin/channel_monitor_handler.go` | 完整 | ChannelMonitor CRUD：创建、编辑、删除、列表、触发检测 |
| 2 | `service/channel_monitor_service.go` | 完整 | 核心逻辑：定时任务或手动触发，对 BaseURL+APIKey 发探测请求，计数成功/失败 |
| 3 | `service/openai_account_scheduler.go` | 1017-1140 | `SelectAccountWithScheduler`：路由时**没有**查询 ChannelMonitor 状态 |
| 4 | `handler/failover_loop.go` | 63-145 | `FailoverState.HandleFailoverError`：故障转移状态机，遍历重试而非熔断 |

---

## 验证结论

### 结论 1：ChannelMonitor 是一个**监控工具**，提供了状态数据但不驱动熔断
**关键代码**：`service/channel_monitor_service.go`

ChannelMonitor 的工作流程：
1. 定时任务（cron）或管理员手动触发
2. 对 `account.base_url + account.api_key` 发 GET 请求（探测 `/v1/models`）
3. 成功：`total_successes++`、`consecutive_fails = 0`
4. 失败：`total_fails++`、`consecutive_fails++`
5. `consecutive_fails >= 3` → monitor status 变为 `"error"`

**逻辑说明**：ChannelMonitor 更新的是 `channel_monitors` 表的 status 字段，**不会自动修改 `accounts.status`**。两个 status 是独立的。

---

### 结论 2：在路由选择 (`SelectAccount`) 时，**不会**利用监控状态来排除不健康的节点
**关键代码**：`service/openai_account_scheduler.go:1040-1055`

```go
// SelectAccount 过滤条件：
WHERE group_id = :group_id
  AND status = 'online'  // 只看 accounts.status
  AND platform = :platform
```

Sub2API 的 `SelectAccount` **只过滤 `accounts.status = 'online'`**，不查询 `channel_monitors.status`。

**这意味着**：
- 即使 ChannelMonitor 检测到账号连续失败并标记为 `error`，只要 `accounts.status` 仍是 `online`，该账号就会参与路由
- 故障账号的摘除**完全依赖人工操作**（管理员手动将 account status 改为 offline）

---

### 结论 3：Sub2API 的容错是"故障后重试"，不是"熔断"
**关键代码**：`handler/failover_loop.go:63-145`

Sub2API 的故障处理：
1. 请求发送失败（如网络错误、上游返回 5xx）
2. `FailoverState.HandleFailoverError` 捕获错误
3. 将失败账号加入 `failed_account_ids` 集合
4. **重试下一个账号**（遍历到列表末尾才返回 503）

**这与熔断的本质区别**：

| 机制 | 熔断（Circuit Breaker） | 故障转移（Failover） |
|------|------------------------|---------------------|
| 触发条件 | 连续 N 次失败（自动计数） | 单次失败（每次都重试） |
| 状态保持 | 熔断后一段时间内不尝试（保护上游） | 每次请求都重新遍历所有账号 |
| 恢复机制 | 自动探测，恢复后闭合 | 无持久状态，每次从头开始 |
| 对上游的影响 | 失败请求不发送（熔断保护） | 失败请求仍发送（可能压垮上游） |

---

## 技术启发

1. **Nexus 需要独立实现熔断机制**：Sub2API 的 ChannelMonitor + FailoverState **不是熔断**，只是监控+重试。

2. **熔断改造建议**：
   - 新增 `account_circuit_breaker` 表（账号熔断状态）
   - 在 `SelectAccount` 入口增加熔断状态过滤
   - 熔断触发条件：连续 3 次失败（参考 Nexus PRD）
   - 熔断探测：每 30 秒探测一次
   - 熔断恢复：探测成功则重置，3 次成功才恢复

3. **ChannelMonitor 可以复用**：ChannelMonitor 的健康检查逻辑可以复用，只需要额外写入 `account_circuit_breaker` 表来驱动熔断。

4. **FailoverLoop 保留**：作为熔断后的兜底重试机制（熔断开启时不再重试已熔断账号，但未熔断账号仍可 failover）。

---

## 大龙确认

| 确认项 | 通过标准 | 结论 |
|--------|---------|------|
| 追踪路径准确 | 文件和行号与实际代码一致 | ✅ |
| 逻辑推断正确 | Sub2API 无熔断，只有监控+重试 | ✅ |
| 技术启发可执行 | 熔断改造方案可落地（新增 circuit_breaker 表 + 修改 SelectAccount） | ✅ |

**请大龙确认**：以上四份审计纪要结论是否全部正确？确认后进入第三阶段（差异分析与修改方案）。

**大龙签字**：___________ **日期**：___________
