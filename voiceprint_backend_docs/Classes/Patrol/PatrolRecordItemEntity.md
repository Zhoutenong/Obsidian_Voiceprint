# PatrolRecordItemEntity — 巡检记录项实体

## 基本信息

- **实体名称**：`PatrolRecordItemEntity`
- **数据库表**：`ast_patrol_record_item`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Patrol/`
- **继承关系**：`Entity<Guid>`

## 实体说明

巡检记录项实体记录每次巡检的具体点位识别结果。每个记录项对应一个巡检点位，存储识别时间、识别值、识别结论、快照图片等详细信息。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `PatrolRecordId` | `Guid` | 关联的巡检记录ID | Foreign Key |
| `PatrolTaskMonitoredPointRelId` | `Guid` | 关联的巡检任务点位ID | Foreign Key |
| `PointId` | `Guid` | 关联的区域点位ID | Foreign Key |

### 识别数据

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Ts` | `DateTime` | 识别（采集）时间戳 | - |
| `GroupId` | `string?` | 数据批次ID | 来自pointdata表 |
| `Unit` | `string?` | 单位 | 多个单位用\|分隔，枚举格式："0:灭,1:亮" |
| `Val` | `double?` | 属性值（数值型） | - |
| `StrVal` | `string?` | 属性值（字符串型） | - |
| `ValueType` | `DataValueTypeEnum` | 值类型 | 0:int,1:float,2:string,3:json,4:enum |

### 识别结果

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Result` | `PatrolDetailResultEnum` | 巡视结论 | 0:正常,1:异常,2:无法识别,3:处理中 |
| `Remark` | `string?` | 异常说明 | - |

### 图片信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `PicUrl` | `string?` | 预置位快照/图片路径 | - |
| `RecPicUrl` | `string?` | 识别区域图片路径 | - |

## 枚举类型

### PatrolDetailResultEnum — 巡检结论
```csharp
public enum PatrolDetailResultEnum
{
    Normal = 0,         // 正常
    Abnormal = 1,       // 异常
    Unrecognized = 2,    // 无法识别
    Processing = 3      // 处理中（默认值）
}
```

### DataValueTypeEnum — 数据值类型
```csharp
public enum DataValueTypeEnum
{
    Int = 0,       // 整数
    Float = 1,      // 浮点数
    String = 2,     // 字符串
    Json = 3,       // JSON对象
    Enum = 4        // 枚举值
}
```

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(PatrolRecordId))]
public PatrolRecordEntity PatrolRecord { get; set; }

[Navigate(NavigateType.OneToOne, nameof(PatrolTaskMonitoredPointRelId))]
public PatrolTaskMonitoredPointRelEntity PatrolTaskMonitoredPointRel { get; set; }

[Navigate(NavigateType.OneToOne, nameof(PointId))]
public AstPointEntity AstPoint { get; set; }
```

## 业务规则

### 记录项生成规则
1. **自动创建**：巡检执行时自动为每个点位创建记录项
2. **状态流转**：Processing → Normal/Abnormal/Unrecognized
3. **图片存储**：保存预置位图片和识别区域图片
4. **值类型**：根据识别结果类型选择Val或StrVal存储

### 识别结果分类
- **Normal（正常）**：识别成功，结果在正常范围内
- **Abnormal（异常）**：识别成功，结果超出正常范围
- **Unrecognized（无法识别）**：AI识别失败或置信度不足
- **Processing（处理中）**：初始状态，等待识别完成

## 相关实体

- [[PatrolRecordEntity]] — 巡检记录（多对一）
- [[PatrolTaskMonitoredPointRelEntity]] — 巡检任务点位关联
- [[AstPointEntity]] — AST点位实体

## 相关服务

- [[PatrolExecutionService]] — 巡检执行服务
- [[PatrolRecordService]] — 巡检记录服务

## 数据查询示例

### 查询某巡检记录的所有记录项
```sql
SELECT
    pri.id,
    pri.point_id,
    pri.ts,
    pri.val,
    pri.str_val,
    pri.result,
    pri.pic_url,
    pri.rec_pic_url
FROM ast_patrol_record_item pri
WHERE pri.patrol_record_id = 'xxx'
ORDER BY pri.ts;
```

### 查询异常的记录项
```sql
SELECT
    pr.id as record_id,
    pr.execution_time,
    pri.point_id,
    pri.result,
    pri.remark,
    pri.pic_url
FROM ast_patrol_record_item pri
JOIN ast_patrol_record pr ON pri.patrol_record_id = pr.id
WHERE pri.result = 1  -- Abnormal
ORDER BY pri.ts DESC;
```

### 查询某点位的识别历史
```sql
SELECT
    pri.ts,
    pri.val,
    pri.str_val,
    pri.result,
    pr.execution_time
FROM ast_patrol_record_item pri
JOIN ast_patrol_record pr ON pri.patrol_record_id = pr.id
WHERE pri.point_id = 'xxx'
ORDER BY pri.ts DESC
LIMIT 100;
```

### 统计巡检识别成功率
```sql
SELECT
    COUNT(*) as total_count,
    SUM(CASE WHEN result = 0 THEN 1 ELSE 0 END) as normal_count,
    SUM(CASE WHEN result = 1 THEN 1 ELSE 0 END) as abnormal_count,
    SUM(CASE WHEN result = 2 THEN 1 ELSE 0 END) as unrecognized_count,
    CAST(SUM(CASE WHEN result IN (0,1) THEN 1 ELSE 0 END) AS FLOAT) / COUNT(*) * 100 as success_rate
FROM ast_patrol_record_item
WHERE ts >= DATEADD(DAY, -7, GETDATE());
```

## 索引建议

```sql
-- 巡检记录ID索引（用于查询某记录的所有项）
CREATE INDEX IX_PatrolRecordId ON ast_patrol_record_item(patrol_record_id);

-- 任务点位关联索引（用于查询特定点位的记录）
CREATE INDEX IX_PatrolTaskMonitoredPointRelId ON ast_patrol_record_item(patrol_task_monitored_point_rel_id);

-- 识别结果索引（用于查询异常记录）
CREATE INDEX IX_Result ON ast_patrol_record_item(result);

-- 点位ID + 时间戳复合索引（用于查询点位历史）
CREATE INDEX IX_PointId_Ts ON ast_patrol_record_item(point_id, ts);
```

## 业务价值

**核心作用**：
1. **详细追溯**：记录每个点位的详细识别结果
2. **异常定位**：快速定位异常点位及其详情
3. **图片证据**：保存快照和识别区域图片作为证据
4. **统计分析**：支持识别成功率和异常率统计

## 使用场景

### 场景1：正常识别记录
```
点位：开关柜温度识别
识别时间：2026-06-04 10:15:30
值类型：Float
值：25.5
单位：℃
结果：Normal
图片：temperature_snapshot.jpg
```

### 场景2：异常识别记录
```
点位：仪表盘读数识别
识别时间：2026-06-04 10:15:35
值类型：Float
值：超出范围
单位：MPa
结果：Abnormal
说明：读数超过正常阈值
图片：gauge_abnormal.jpg
```

### 场景3：无法识别记录
```
点位：设备状态识别
识别时间：2026-06-04 10:15:40
值类型：Enum
值：-
结果：Unrecognized
说明：图片模糊，无法识别
图片：blur_snapshot.jpg
```

## 识别流程

### 视频巡检识别流程
```
1. 开始巡检任务
   ↓
2. 获取巡检点位列表
   ↓
3. 对每个点位：
   a. 调用预置位（视频转动到指定位置）
   b. 等待云台稳定（2秒）
   c. 采集快照图片
   d. 调用AI识别服务
   e. 保存识别结果（创建PatrolRecordItem）
   ↓
4. 生成巡检报告
   ↓
5. 标记异常项并告警
```

### 结果处理
```csharp
// AI识别完成后处理结果
public async Task ProcessRecognitionResultAsync(Guid recordItemId, RecognitionResult result)
{
    var item = await _repository.GetAsync(recordItemId);
    
    if (result.Success)
    {
        if (result.IsAbnormal)
        {
            item.Result = PatrolDetailResultEnum.Abnormal;
            item.Remark = $"识别值 {result.Value} 超出正常范围";
            // 触发告警
            await _alarmService.CreatePatrolAlarmAsync(item);
        }
        else
        {
            item.Result = PatrolDetailResultEnum.Normal;
        }
        
        // 保存识别值
        if (result.ValueType == DataValueTypeEnum.Int || 
            result.ValueType == DataValueTypeEnum.Float)
        {
            item.Val = Convert.ToDouble(result.Value);
        }
        else
        {
            item.StrVal = result.Value.ToString();
        }
    }
    else
    {
        item.Result = PatrolDetailResultEnum.Unrecognized;
        item.Remark = "AI识别失败：" + result.ErrorMessage;
    }
    
    await _repository.UpdateAsync(item);
}
```

## 图片存储策略

### 图片类型
- **预置位图片（PicUrl）**：摄像头转到预置位后的全景图片
- **识别区域图片（RecPicUrl）**：AI识别时的裁剪区域图片

### 存储路径建议
```
/patrol-records/
  ├── {patrol_record_id}/
  │   ├── presets/
  │   │   ├── {point_id}_1.jpg
  │   │   └── {point_id}_2.jpg
  │   └── recognition/
  │       ├── {point_id}_rec_1.jpg
  │       └── {point_id}_rec_2.jpg
```

### 图片URL生成
```csharp
var picUrl = $"/patrol-records/{recordId}/presets/{pointId}.jpg";
var recPicUrl = $"/patrol-records/{recordId}/recognition/{pointId}_rec.jpg";
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Patrol/PatrolRecordItemEntity.cs`
