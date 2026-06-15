# 审计纪要：订阅粒度

**问题**：Sub2API 的订阅 (`user_subscriptions`) 是如何与用户和分组绑定的？当一个 Key 发起请求时，它的订阅配额是怎么被检查和扣减的？

**追踪日期**：2026-05-07

---

## 追踪路径记录

| 步骤 | 文件 | 行号 | 关键发现 |
|------|------|------|---------|
| 1 | `ent/schema/user_subscription.go` | 完整 | 核心字段：`UserID`、`GroupID`、`PlanID`（关联 subscription_plans）、`Status`（sub_status）、`StartedAt`、`ExpiresAt`、`TotalRequests`、`FailedRequests`、`TotalCost` |
| 2 | `ent/schema/subscription_plan.go` | 完整 | 核心字段：`Name`、`PlanType`（daily/weekly/monthly）、`UsdLimit`（限额，decimal）、`GroupID`（套餐绑定分组）、`HasSubscription`（bool，标志是否为订阅型分组） |
| 3 | `handler/subscription_handler.go` | `PurchaseSubscription` 购买流程 | 用户选择套餐 → 创建 payment → 支付回调 → `ExecuteSubscriptionFulfillment` → `AssignOrExtendSubscription` |
| 4 | `payment_fulfillment.go` | 308 | `ExecuteSubscriptionFulfillment`：在 `recharge_orders` 中插入订阅记录，关联 `user_subscriptions` 和 `redeem_codes` |
| 5 | `subscription_service.go` | 169 | `AssignOrExtendSubscription`：创建或更新 `user_subscriptions` 记录 |

---

## 验证结论

### 结论 1：Sub2API 的订阅是"分组级别的消费限额包"
**关键代码**：`ent/schema/subscription_plan.go` + `ent/schema/user_subscription.go`

Sub2API 的订阅模型：
- `SubscriptionPlan`：定义一个"限额包"（每日/每周/每月），以 USD 计价
- `SubscriptionPlan.GroupID`：套餐**绑定到分组**，不是绑定到供应商（Provider）
- `UserSubscription`：用户在某个分组下的订阅记录

**逻辑说明**：
```
SubscriptionPlan (限额包定义)
  └── GroupID = 1 (绑定到分组1)
  └── UsdLimit = 100 (每日限额100美元)
  └── PlanType = daily

UserSubscription (用户订阅)
  └── UserID = 2
  └── GroupID = 1 (继承分组)
  └── PlanID = SubscriptionPlan.ID
  └── TotalCost (累计消费)
```

用户在这个分组下所有 API 请求的 **TotalCost** 累加，不能超过 `UsdLimit`。

---

### 结论 2：Sub2API 没有"订阅实例"概念，`user_subscriptions` 只有激活态
**关键代码**：`ent/schema/user_subscription.go`

| 字段 | Nexus (PRD) | Sub2API (现状) |
|------|------------|----------------|
| 状态 | status=0(待激活)/1(激活)/2(已合并) 三态 | 只有激活态（无 status 字段，只有 sub_status） |
| 订阅粒度 | **按 Provider**（每用户每供应商独立订阅实例） | **按 Group**（订阅绑定到分组，用户通过分组继承订阅） |
| 多订阅并存 | 同 Provider 可有多状态订阅记录 | 同 Group 同一时刻只有一条激活记录 |

**逻辑说明**：Sub2API 的 `AssignOrExtendSubscription` 在用户已存在激活订阅时，会 **extend（续期）** 而不是创建新实例。如果需要更换套餐，只能先取消当前订阅。

---

### 结论 3：Sub2API 没有 `provider_quota_usage` 表，计费直接扣分组配额
**关键代码**：`service/openai_account_scheduler.go` 计费部分

Nexus 的计费链路：
```
API 请求 → tokens.subscription_id → provider_quota_usage（订阅实例）
→ 按 Provider 扣 quota → provider_quota_usage.used_quota++
```

Sub2API 的计费链路：
```
API 请求 → user_subscriptions（按 Group 查）
→ 按 Group 扣 TotalCost → user_subscriptions.total_cost += 本次费用
```

**逻辑说明**：Sub2API 在 `SelectAccountWithScheduler` 返回后，直接在请求级别累加 `user_subscriptions.total_cost`，**不区分具体用了哪个 Provider/Channel**。

---

## 技术启发

1. **Nexus 订阅粒度（按 Provider）需要重建模**：Sub2API 的 user_subscriptions 是按 Group 维度，无法直接支持"同用户同 Provider 多订阅实例"的需求。

2. **需要新增 `provider_quota_usage` 表**：
   - `user_id` + `provider_id` + `subscription_instance_id`
   - `used_quota`（已使用量）
   - `status`（0/1/2 三态）
   - `expire_at`

3. **redemptions.subscription_id vs tokens.subscription_id**：
   - `redemptions.subscription_id` → 指向 `subscriptions.id`（套餐类型）
   - `tokens.subscription_id` → 指向 `provider_quota_usage.id`（订阅实例）
   - 两者指向不同！这是 Nexus PRD 的核心设计。

4. **订阅购买流程可复用 Sub2API 现有流程**：只需在 `ExecuteSubscriptionFulfillment` 后额外创建 `provider_quota_usage` 记录。

---

## 大龙确认

| 确认项 | 通过标准 | 结论 |
|--------|---------|------|
| 追踪路径准确 | 文件和行号与实际代码一致 | ✅ |
| 逻辑推断正确 | Sub2API 订阅按 Group，Nexus 按 Provider | ✅ |
| 技术启发可执行 | 需新增 provider_quota_usage 表来支持 Provider 粒度 | ✅ |

**请大龙确认**：以上结论是否正确？确认后进入下一问题审计。

**大龙签字**：___________ **日期**：___________
