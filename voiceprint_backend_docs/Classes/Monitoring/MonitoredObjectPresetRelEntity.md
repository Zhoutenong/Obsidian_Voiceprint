# MonitoredObjectPresetRelEntity — 监测对象-预置位关联实体

## 基本信息

- **实体名称**：`MonitoredObjectPresetRelEntity`
- **数据库表**：`ast_monitored_object_preset_rel`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/`
- **继承关系**：`Entity<Guid>`

## 实体说明

监测对象-预置位关联实体用于定义"哪些监测对象关联哪些摄像头预置位"。这是视频巡检系统的核心配置，支持自动巡检时自动调用预置位。

例如：
- 变压器A 关联 预置位1（正面视角）、预置位2（侧面视角）
- 开关柜B 关联 预置位3（仪表盘视角）

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `MonitoredObjectId` | `Guid` | 监测对象ID | Foreign Key |
| `AstSensorId` | `Guid` | 预置位ID（传感器ID） | Foreign Key |

### 关联信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `OrderNum` | `int?` | 排序号 | 默认为1，控制调用顺序 |

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(MonitoredObjectId))]
public MonitoredObjectAggregateRoot MonitoredObject { get; set; }

[Navigate(NavigateType.OneToOne, nameof(AstSensorId))]
public SensorEntity AstSensor { get; set; }
```

## 业务规则

### 预置位关联规则
1. **多对多关系**：一个监测对象可以关联多个预置位，一个预置位也可以被多个对象关联
2. **顺序控制**：通过 `OrderNum` 控制预置位的调用顺序
3. **自动巡检**：巡检时按顺序调用所有关联的预置位进行图像采集
4. **视角完整**：建议为每个对象配置多个角度的预置位

### 配置建议

#### 变压器预置位配置
```
变压器A
├── 预置位1 - 正面视角（OrderNum: 1）
├── 预置位2 - 侧面视角（OrderNum: 2）
├── 预置位3 - 顶部视角（OrderNum: 3）
└── 预置位4 - 仪表盘视角（OrderNum: 4）
```

#### 开关柜预置位配置
```
开关柜B
├── 预置位1 - 柜门视角（OrderNum: 1）
├── 预置位2 - 仪表盘视角（OrderNum: 2）
└── 预置位3 - 母线室视角（OrderNum: 3）
```

## 相关实体

- [[MonitoredObjectAggregateRoot]] — 监测对象
- [[SensorEntity]] — 传感器/预置位实体
- [[PatrolRecordEntity]] — 巡检记录

## 相关服务

- [[PatrolExecutionService]] — 巡检执行服务
- [[CameraService]] — 摄像机服务
- [[PresetService]] — 预置位服务

## 数据查询示例

### 查询某监测对象的所有预置位
```sql
SELECT
    s.id as sensor_id,
    s.name as sensor_name,
    s.preset_position as preset_position,
    rel.order_num
FROM ast_monitored_object_preset_rel rel
JOIN ast_sensor s ON rel.ast_sensor_id = s.id
WHERE rel.monitored_object_id = 'xxx'
ORDER BY rel.order_num;
```

### 查询某预置位关联的所有监测对象
```sql
SELECT
    mo.id as object_id,
    mo.name as object_name,
    rel.order_num
FROM ast_monitored_object_preset_rel rel
JOIN ast_monitored_object mo ON rel.monitored_object_id = mo.id
WHERE rel.ast_sensor_id = 'xxx'
ORDER BY rel.order_num;
```

### 批量查询巡检对象的预置位列表
```sql
-- 查询多个监测对象及其预置位
SELECT
    mo.id as object_id,
    mo.name as object_name,
    s.id as sensor_id,
    s.name as sensor_name,
    s.preset_position,
    rel.order_num
FROM ast_monitored_object mo
LEFT JOIN ast_monitored_object_preset_rel rel ON mo.id = rel.monitored_object_id
LEFT JOIN ast_sensor s ON rel.ast_sensor_id = s.id
WHERE mo.id IN ('obj1', 'obj2', 'obj3')
ORDER BY mo.name, rel.order_num;
```

## 索引建议

```sql
-- 监测对象索引（用于查询某对象的预置位）
CREATE INDEX idx_monitored_object ON ast_monitored_object_preset_rel(monitored_object_id);

-- 预置位索引（用于查询某预置位关联的对象）
CREATE INDEX idx_ast_sensor ON ast_monitored_object_preset_rel(ast_sensor_id);

-- 排序索引（用于按顺序调用）
CREATE INDEX idx_order_num ON ast_monitored_object_preset_rel(order_num);

-- 复合唯一索引（对象+预置位唯一）
CREATE UNIQUE INDEX ux_object_sensor ON ast_monitored_object_preset_rel(monitored_object_id, ast_sensor_id);
```

## 巡检流程集成

### 自动巡检流程
```
1. 开始巡检
   ↓
2. 获取巡检任务中的监测对象列表
   ↓
3. 对每个监测对象：
   a. 查询对象关联的所有预置位（按 OrderNum 排序）
   b. 依次调用每个预置位进行图像采集
   c. 采集的图像用于 AI 识别
   ↓
4. 生成巡检报告
```

### 预置位调用示例
```csharp
// 获取对象的预置位列表
var presets = await _monitoredObjectPresetRelRepository.GetListAsync(
    rel => rel.MonitoredObjectId == objectId,
    orderBy: rel => rel.OrderNum,
    orderByOrderByType: OrderByType.Asc
);

// 依次调用预置位
foreach (var preset in presets)
{
    var sensor = await _sensorRepository.GetAsync(preset.AstSensorId);

    // 调用摄像机云台，转到预置位
    await _cameraService.MoveToPresetAsync(sensor.CameraId, sensor.PresetPosition);

    // 等待云台稳定
    await Task.Delay(2000);

    // 采集图像
    var image = await _cameraService.CaptureImageAsync(sensor.CameraId);

    // AI 识别
    var recognitionResult = await _aiService.RecognizeAsync(image);

    // 保存巡检记录
    await SavePatrolRecordAsync(objectId, preset.Id, recognitionResult);
}
```

## 业务价值

**核心作用**：
1. **自动巡检**：支持巡检时自动调用预置位，无需手动操作
2. **视角完整**：通过多个预置位获得对象的完整视角信息
3. **顺序控制**：支持定义预置位调用顺序，优化巡检效率
4. **灵活配置**：支持每个对象独立配置预置位组合

## 设计考量

### 为什么使用关联表？
1. **多对多关系**：一个对象可以有多个预置位，一个预置位也可以服务多个对象
2. **顺序控制**：关联表支持独立于预置位自身的排序
3. **运行时配置**：支持运行时动态调整预置位关联
4. **性能优化**：通过索引快速查询对象的预置位列表

### 与 SensorEntity 的关系
- `SensorEntity` 定义预置位本身的信息（名称、位置、所属摄像机）
- `MonitoredObjectPresetRelEntity` 定义对象与预置位的关联关系和调用顺序
- 两者配合实现灵活的视频巡检配置

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/MonitoredObjectPresetRelEntity.cs`