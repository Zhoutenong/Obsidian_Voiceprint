# MonitoredPointAlarmCategoryRelEntity — 监测点位-告警分类关联实体

## 基本信息

- **实体名称**：`MonitoredPointAlarmCategoryRelEntity`
- **数据库表**：`ast_monitored_point_alarm_category_rel`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/`
- **继承关系**：`Entity<Guid>` → `ICreationAuditedObject`

## 实体说明

监测点位-告警分类关联实体用于定义"哪些点位可以产生哪些类型的告警"。这是告警系统的核心配置，支持灵活的告警分类管理。

例如：
- 温度点位 关联 温度告警、设备故障告警
- 振动点位 关联 振动告警、机械故障告警

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `MonitoredPointId` | `Guid` | 监测点位ID | Foreign Key |
| `AlarmCategoryId` | `Guid` | 告警分类ID | Foreign Key |

### 关联信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `OrderNum` | `int` | 排序号 | 控制告警分类的显示/处理顺序 |

### 审计信息

| 字段名 | 数据类型 | 说明 | 来源 |
|-------|---------|------|------|
| `CreationTime` | `DateTime` | 创建时间 | ICreationAuditedObject |
| `CreatorId` | `Guid?` | 创建者ID | ICreationAuditedObject |

## 业务规则

### 关联配置规则
1. **多对多关系**：一个点位可以关联多个告警分类，一个分类也可以关联多个点位
2. **优先级控制**：通过 `OrderNum` 控制告警分类的优先级
3. **告警匹配**：点位异常时，根据关联的告警分类确定告警类型
4. **灵活性**：支持运行时动态调整告警分类关联

### 告警分类示例

#### 温度点位告警分类
```
温度点位_Temp001
├── 温度告警（OrderNum: 1）- 温度超过阈值
├── 设备故障告警（OrderNum: 2）- 温度异常可能表示设备故障
└── 环境超标告警（OrderNum: 3）- 环境温度超标
```

#### 振动点位告警分类
```
振动点位_Vib001
├── 振动告警（OrderNum: 1）- 振动超过阈值
├── 机械故障告警（OrderNum: 2）- 振动异常可能表示机械故障
└── 设备老化告警（OrderNum: 3）- 长期振动异常可能表示设备老化
```

## 相关实体

- [[MonitoredPointEntity]] — 监测点位
- [[AlarmCategoryAggregateRoot]] — 告警分类
- [[AlarmRecordAggregateRoot]] — 告警记录

## 相关服务

- [[MonitoredPointAlarmCategoryRelService]] — 点位告警分类关联服务
- [[AlarmRecordService]] — 告警记录服务
- [[AlarmProcessingService]] — 告警处理服务

## 数据查询示例

### 查询某点位的所有告警分类
```sql
SELECT
    ac.id as category_id,
    ac.name as category_name,
    ac.type as category_type,
    rel.order_num
FROM ast_monitored_point_alarm_category_rel rel
JOIN ast_alarm_category ac ON rel.alarm_category_id = ac.id
WHERE rel.monitored_point_id = 'xxx'
ORDER BY rel.order_num;
```

### 查询某告警分类关联的所有点位
```sql
SELECT
    mp.id as point_id,
    mp.name as point_name,
    mp.unit,
    rel.order_num
FROM ast_monitored_point_alarm_category_rel rel
JOIN ast_monitored_point mp ON rel.monitored_point_id = mp.id
WHERE rel.alarm_category_id = 'xxx'
ORDER BY rel.order_num;
```

### 查询点位可能产生的告警类型
```sql
-- 用于告警配置或告警预测
SELECT
    mp.id as point_id,
    mp.name as point_name,
    ac.id as category_id,
    ac.name as category_name,
    ac.type as category_type,
    rel.order_num
FROM ast_monitored_point_alarm_category_rel rel
JOIN ast_monitored_point mp ON rel.monitored_point_id = mp.id
JOIN ast_alarm_category ac ON rel.alarm_category_id = ac.id
WHERE mp.id IN ('point1', 'point2', 'point3')
ORDER BY mp.name, rel.order_num;
```

## 告警生成流程

### 点位异常时的告警分类匹配
```
1. 点位数据采集
   ↓
2. 数据异常检测（阈值判断、趋势分析等）
   ↓
3. 查询点位关联的告警分类（按 OrderNum 排序）
   ↓
4. 依次尝试匹配告警分类：
   a. 检查是否满足该分类的告警条件
   b. 如果满足，创建该分类的告警记录
   c. 继续检查下一个分类（可能产生多个告警）
   ↓
5. 生成告警记录并通知
```

### 告警分类匹配示例
```csharp
// 点位异常处理
public async Task HandlePointValueAsync(Guid pointId, decimal value)
{
    // 1. 检测异常
    var isAbnormal = await CheckIfAbnormalAsync(pointId, value);
    if (!isAbnormal) return;

    // 2. 获取点位关联的告警分类
    var categoryRels = await _relRepository.GetListAsync(
        rel => rel.MonitoredPointId == pointId,
        orderBy: rel => rel.OrderNum,
        orderByOrderByType: OrderByType.Asc
    );

    // 3. 依次检查每个告警分类
    foreach (var categoryRel in categoryRels)
    {
        var category = await _categoryRepository.GetAsync(categoryRel.AlarmCategoryId);

        // 检查是否满足该分类的告警条件
        if (await CheckAlarmConditionAsync(pointId, value, category))
        {
            // 创建告警记录
            await CreateAlarmRecordAsync(pointId, value, category);
        }
    }
}
```

## 索引建议

```sql
-- 点位索引（用于查询某点位的告警分类）
CREATE INDEX idx_monitored_point ON ast_monitored_point_alarm_category_rel(monitored_point_id);

-- 告警分类索引（用于查询某分类关联的点位）
CREATE INDEX idx_alarm_category ON ast_monitored_point_alarm_category_rel(alarm_category_id);

-- 排序索引（用于按优先级处理）
CREATE INDEX idx_order_num ON ast_monitored_point_alarm_category_rel(order_num);

-- 复合唯一索引（点位+分类唯一）
CREATE UNIQUE INDEX ux_point_category ON ast_monitored_point_alarm_category_rel(monitored_point_id, alarm_category_id);
```

## 业务价值

**核心作用**：
1. **灵活分类**：支持一个点位产生多种类型的告警
2. **优先级控制**：通过排序控制告警分类的优先级
3. **告警过滤**：前端可以按分类过滤告警列表
4. **统计分析**：支持按分类统计告警数据

## 设计考量

### 为什么使用关联表？
1. **多对多关系**：一个点位可以产生多种告警，一种告警可以来自多个点位
2. **灵活配置**：支持运行时动态调整告警分类关联
3. **独立排序**：关联表支持独立的排序，不受点位或分类自身排序影响
4. **性能优化**：通过索引快速查询点位的告警分类

### 与告警策略的关系
- 告警分类定义告警的"类型"
- 告警策略定义告警的"条件"（阈值、趋势等）
- 两者配合实现完整的告警配置

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/MonitoredPointAlarmCategoryRelEntity.cs`