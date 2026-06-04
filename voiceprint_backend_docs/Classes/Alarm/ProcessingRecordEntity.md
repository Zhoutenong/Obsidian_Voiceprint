# ProcessingRecordEntity — 告警处理记录实体

## 基本信息

- **实体名称**：`ProcessingRecordEntity`
- **数据库表**：`ast_processing_record`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Alarm/`
- **继承关系**：`Entity<Guid>`

## 实体说明

告警处理记录实体用于记录告警的处理历史，包括处理时间、操作人、处理动作等信息。它与告警记录是一对多关系，一个告警可以有多条处理记录，形成完整的处理链条。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `AlarmRecordId` | `Guid` | 关联的告警记录ID | Foreign Key |

### 处理信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `ProcessingTime` | `DateTime` | 处理时间 | 记录处理操作的时间戳 |
| `Operator` | `string?` | 操作人姓名 | 可选，用于显示 |
| `UserId` | `Guid?` | 操作用户ID | 关联用户表 |
| `ProcessingAction` | `string` | 处理操作描述 | 详细记录处理动作 |

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(AlarmRecordId))]
public AlarmRecordAggregateRoot AlarmRecord { get; set; }
```

## 业务规则

### 处理记录生命周期
1. **初始状态**：告警创建时无处理记录
2. **状态变更**：每次告警状态变更时创建处理记录
3. **追加说明**：运维人员可在处理过程中添加处理说明
4. **历史追溯**：通过处理记录链条追溯告警处理全过程

### 典型处理动作类型
- `接受告警` - 运维人员接受告警处理
- `现场处理` - 现场人员进行设备处理
- `远程调整` - 远程调整设备参数
- `派单处理` - 将告警派发给相关团队
- `添加备注` - 添加处理进度说明
- `关闭告警` - 问题解决后关闭告警
- **问题解决**：处理记录显示问题已解决
- **经验积累**：处理记录为类似问题提供参考

## 处理记录示例

### 典型处理链条
```
2026-06-04 10:15:00 - 张三(接受告警) - 接到告警，开始处理
2026-06-04 10:20:00 - 张三(远程调整) - 远程调整设备参数，观察效果
2026-06-04 10:35:00 - 张三(添加备注) - 设备参数已调整，等待监测
2026-06-04 11:00:00 - 张三(关闭告警) - 设备运行正常，关闭告警
```

## 相关实体

- [[AlarmRecordAggregateRoot]] — 告警记录（多对一关系）
- [[UserAggregateRoot]] — 操作用户（rbac模块）

## 相关服务

- [[AlarmProcessingService]] — 告警处理服务
- [[AlarmRecordService]] — 告警记录服务

## 数据查询示例

### 查询某告警的处理历史
```sql
SELECT * FROM ast_processing_record
WHERE alarm_record_id = 'xxx'
ORDER BY processing_time ASC
```

### 查询某用户的所有处理记录
```sql
SELECT pr.*, ar.alarm_content, ar.alarm_time
FROM ast_processing_record pr
JOIN ast_alarm_record ar ON pr.alarm_record_id = ar.id
WHERE pr.user_id = 'user-guid'
ORDER BY pr.processing_time DESC
```

## 索引建议

```sql
-- 告警记录ID索引（用于查询某告警的处理历史）
CREATE INDEX idx_alarm_record ON ast_processing_record(alarm_record_id);

-- 处理时间索引（用于时间范围查询）
CREATE INDEX idx_processing_time ON ast_processing_record(processing_time);

-- 用户ID索引（用于查询某用户的处理记录）
CREATE INDEX idx_user_id ON ast_processing_record(user_id);
```

## 设计考量

### 为什么使用独立实体？
1. **历史追溯**：保持完整的处理历史，而非仅保留最新状态
2. **多人协作**：支持多人协作处理同一告警
3. **审计需求**：满足运维审计需求，记录谁在何时做了什么
4. **灵活性**：支持任意长度的处理链条

### 与告警状态的关系
- 告警状态反映当前处理阶段
- 处理记录详细记录处理过程
- 两者互补，共同构成完整的告警处理视图

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Alarm/ProcessingRecordEntity.cs`