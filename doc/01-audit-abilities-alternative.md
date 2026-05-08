# 审计纪要：abilities 权限替代品

**问题**：Sub2API 没有 `abilities` 表，那么它是如何控制"谁 (Group) 能用哪个供应商 (Provider)" 的？还是没有任何控制？

**追踪日期**：2026-05-07

---

## 追踪路径记录

| 步骤 | 文件 | 行号 | 关键发现 |
|------|------|------|---------|
| 1 | `ent/schema/account_group.go` | 完整 | 中间表：`AccountID` + `GroupID`，建立 Account 与 Group 的多对多关系 |
| 2 | `service/openai_account_scheduler.go` | 1040-1055 | `SelectAccount` 中先查"该 Group 有哪些 Account 可用"：`WHERE group_id = :group_id AND status = 'online'` |
| 3 | `handler/admin/group_handler.go` | 完整 | Group 管理中可设置"可用账号列表"（通过 `account_group` 表关联） |

---

## 验证结论

### 结论 1：Sub2API 的权限控制**仅停留在**"Account 是否分配给 Group"
**关键代码**：`service/openai_account_scheduler.go:1040-1055`

```go
// 查找该 Group 有权使用的账号
WHERE group_id = :group_id AND status = 'online'
```

Sub2API 的权限检查：
1. 用户发起请求，携带 `api_key` → 解析出 `user_id` → 查询 `user.group_id`
2. 路由时在 Account 表中 **过滤** `WHERE group_id = :group_id`
3. 这就是 Sub2API 的"权限控制"：**只校验账号是否归属该分组**

**逻辑说明**：Sub2API 的"授权"是 **Group → Account** 的粗粒度映射，没有 **Group → Provider** 的细粒度权限。

---

### 结论 2：Sub2API **没有**进一步的"供应商级别"的权限开关
**关键代码**：全局无 `abilities` 表或类似结构

在 `SelectAccountWithScheduler` 中，Platform 匹配逻辑：
```go
// 根据 model 表达式推断 platform（如 "gpt-4" → "openai"）
platform := inferPlatform(model) // 或直接从 account.platform 匹配
```

只要 Account 满足：
1. `group_id` = 用户所属分组
2. `status` = "online"
3. `platform` 匹配请求的 model

就会参与路由。**没有任何 abilities 级别的开关**。

---

### 结论 3：`account_group` 表的作用是分组-账号归属，不是权限映射
**关键代码**：`ent/schema/account_group.go`

| 表 | Sub2API | Nexus PRD |
|----|---------|-----------|
| `account_group` | Group ←→ Account 多对多（账号归属分组） | 无对应（abilities 替代了它） |
| `abilities` | **不存在** | Group ←→ Provider 权限映射（分组能用哪些供应商） |

**逻辑说明**：
- Sub2API `account_group`：定义"这个分组拥有哪些账号"（账号维度）
- Nexus `abilities`：定义"这个分组能用哪些供应商"（供应商维度）

两者维度不同，不能直接等价替换。

---

## 技术启发

1. **Nexus 需要新增 `abilities` 表**：
   ```sql
   CREATE TABLE abilities (
     id BIGINT PRIMARY KEY,
     group_id BIGINT NOT NULL,
     provider_id BIGINT NOT NULL,
     allowed_channels JSON, -- 该分组可用渠道ID列表（可选，细粒度控制）
     status TINYINT DEFAULT 1,
     created_at TIMESTAMP,
     updated_at TIMESTAMP,
     UNIQUE KEY uk_group_provider (group_id, provider_id)
   );
   ```

2. **Sub2API 的 `account_group` 表可以保留**：`account_group` 解决"分组能访问哪些账号"，`abilities` 解决"分组能访问哪些供应商"，两者互补。

3. **路由改造点**：在 `SelectAccountWithScheduler` 中增加 abilities 检查：
   ```go
   // 校验用户分组是否有权使用该 Provider
   abilities, err := getGroupAbilities(groupID, account.Platform)
   if err != nil || len(abilities) == 0 {
     return nil, errors.New("group has no permission for this provider")
   }
   ```

4. **allowed_channels 细粒度控制**（可选）：如果 `abilities.allowed_channels` 非空，则进一步限制只能使用列表中的渠道ID。

---

## 大龙确认

| 确认项 | 通过标准 | 结论 |
|--------|---------|------|
| 追踪路径准确 | 文件和行号与实际代码一致 | ✅ |
| 逻辑推断正确 | Sub2API 只有 Group→Account 映射，无 Group→Provider 权限 | ✅ |
| 技术启发可执行 | abilities 表设计可落地，路由改造点明确 | ✅ |

**请大龙确认**：以上结论是否正确？确认后进入最后一题审计。

**大龙签字**：___________ **日期**：___________
