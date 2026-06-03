# 点位值缓存服务 (PointValueCacheService)

## 概述
点位值缓存服务提供内存级别的点位值历史记录管理，支持重复值过滤功能，避免高频重复数据的过度处理和存储。

## 职责
- 记录点位值上报历史
- 基于时间窗口的重复值检测
- 自动清理过期缓存记录
- 线程安全的并发访问控制
- 内存使用量控制（防止内存泄漏）

## 主要接口

### 检查是否应该上报值
```csharp
Task<bool> ShouldReportValueAsync(string pointId, object value, int timeWindowMinutes, int maxReportsInWindow)
```

**功能说明**：基于时间窗口和重复次数限制，判断当前点位值是否应该被上报处理

**参数说明**：
- `pointId`: 点位ID（唯一标识）
- `value`: 点位值（任意类型）
- `timeWindowMinutes`: 时间窗口长度（分钟）
- `maxReportsInWindow`: 时间窗口内允许的最大上报次数

**返回值**：
- `true`: 应该上报（重复次数未超限）
- `false`: 应该过滤（重复次数超限）

**判断逻辑**：
```
当前时间 - timeWindowMinutes → 窗口起始时间
    ↓
清理该点位的历史记录（保留窗口内）
    ↓
统计窗口内相同值的上报次数
    ↓
比较: sameValueCount < maxReportsInWindow
    ↓
返回判断结果
```

**使用场景**：
- 避免传感器稳定状态下产生大量重复数据
- 减少不必要的策略计算和数据库写入
- 保留必要的最小上报频率用于状态追踪

**调用示例**：
```csharp
// 10分钟内最多上报3次相同值
bool shouldReport = await _cacheService.ShouldReportValueAsync(
    "sensor-001-temperature",
    25.5,
    10,
    3
);
```

### 记录点位值上报
```csharp
Task RecordValueReportAsync(string pointId, object value, DateTime reportTime)
```

**功能说明**：记录点位值的上报时间和值，用于后续的重复值判断

**参数说明**：
- `pointId`: 点位ID
- `value`: 点位值（任意类型）
- `reportTime`: 上报时间

**记录内容**：
- 点位值（转换为字符串存储）
- 上报时间（精确到 DateTime）

**内存控制**：
- 每个点位最多保留100条历史记录
- 超过限制时删除最旧的记录
- 自动防止内存无限增长

**线程安全**：
- 使用点位级别的锁对象
- 支持多线程并发访问
- 保证数据一致性

### 清理过期记录
```csharp
Task CleanExpiredRecordsAsync(DateTime expiredBefore)
```

**功能说明**：批量清理指定时间之前的所有缓存记录，释放内存空间

**参数说明**：
- `expiredBefore`: 清理此时间之前的所有记录

**清理流程**：
1. 遍历所有点位的历史记录
2. 使用点位级别锁保证线程安全
3. 移除过期记录
4. 如果点位无记录则删除整个条目
5. 统计清理结果并记录日志

**清理统计**：
- 清理的点位数量
- 清理的记录总数

**调用时机**：
- 定时后台任务（如每小时一次）
- 内存压力较大时手动触发
- 系统维护期间批量清理

## 数据结构

### 内部存储

**点位值历史记录**：
```csharp
ConcurrentDictionary<string, List<PointValueRecord>> _valueHistory
```
- **Key**: 点位ID（string）
- **Value**: 历史记录列表（List<PointValueRecord>）

**点位锁对象**：
```csharp
ConcurrentDictionary<string, object> _pointLocks
```
- **Key**: 点位ID（string）
- **Value**: 锁对象（object）
- **作用**: 保证点位级别的线程安全

### 历史记录项

```csharp
private class PointValueRecord
{
    public object? Value { get; set; }        // 点位值
    public DateTime ReportTime { get; set; }   // 上报时间
}
```

## 生命周期管理

### 服务生命周期
```csharp
[Dependency(ServiceLifetime.Singleton)]
public class PointValueCacheService : IPointValueCacheService
```
- **Singleton**: 单例模式
- **原因**: 缓存数据需要在整个应用生命周期内保持
- **注意**: Logger 通过构造函数注入，单例服务需要特殊处理

### 记录生命周期

```
记录创建 → 随时间推移 → 超出时间窗口 → 自动清理
                ↓
         超过数量限制（100条） → 删除最旧记录
                ↓
         整个点位无记录 → 删除点位条目
```

## 并发控制

### 线程安全机制

**点位级别锁**：
```csharp
var lockObj = _pointLocks.GetOrAdd(pointId, _ => new object());
lock (lockObj)
{
    // 临界区操作
}
```

**优势**：
- 不同点位可以并行处理
- 同一点位串行处理保证一致性
- 避免全局锁的性能瓶颈

### 并发数据结构

- `ConcurrentDictionary`: 线程安全的字典
- `GetOrAdd()`: 原子操作，线程安全
- `TryRemove()`: 原子删除操作

## 性能优化

### 时间窗口清理
```csharp
var windowStart = now.AddMinutes(-timeWindowMinutes);
history.RemoveAll(r => r.ReportTime < windowStart);
```
- 在每次检查前先清理过期记录
- 减少无效数据的存储和遍历
- 保持数据集的紧凑性

### 数量限制
```csharp
const int maxHistoryCount = 100;
if (history.Count > maxHistoryCount)
{
    history.RemoveRange(0, history.Count - maxHistoryCount);
}
```
- 防止单个点位占用过多内存
- 自动删除最旧记录
- 保留最新数据用于判断

### 批量清理优化
```csharp
foreach (var kvp in _valueHistory.ToList())
{
    // 使用 ToList() 避免迭代时修改集合
}
```
- 先转换为列表再迭代
- 安全地在迭代中删除元素
- 避免并发修改异常

## 使用场景

### 传感器数据过滤
```
温度传感器稳定在25℃
    ↓
10分钟内上报100次相同值
    ↓
配置: 10分钟内最多3次
    ↓
只处理前3次，过滤后97次
    ↓
减少策略计算和数据库写入
```

### 告警抑制
```
设备持续告警状态
    ↓
避免产生大量重复告警记录
    ↓
保留必要的告警频率
    ↓
降低告警疲劳和系统负载
```

### 数据采集优化
```
高频采集设备（1秒1次）
    ↓
业务处理只需要分钟级数据
    ↓
通过缓存过滤中间数据
    ↓
降低系统整体负载
```

## 配置建议

### 时间窗口设置
| 数据类型 | 建议窗口 | 说明 |
|---------|---------|------|
| 快变信号 | 5-10分钟 | 温度、压力等变化较快的量 |
| 慢变信号 | 30-60分钟 | 油位、位置等变化较慢的量 |
| 状态信号 | 60-120分钟 | 开关状态、运行状态等 |

### 上报次数设置
| 场景 | 建议次数 | 说明 |
|-----|---------|------|
| 正常监控 | 3-5次 | 保证最小监控频率 |
| 告警抑制 | 1-2次 | 减少重复告警 |
| 趋势分析 | 10-20次 | 保留趋势特征 |

## 注意事项

### 内存管理
- **监控缓存大小**: 观察系统内存使用情况
- **定期清理**: 建议配置定时清理任务
- **点位数量**: 注意点位总数对内存的影响

### 数据一致性
- **值比较**: 使用字符串转换后的值进行比较
- **时间精度**: 使用 DateTime 精度，注意时区
- **并发安全**: 所有操作都是线程安全的

### 日志级别
- **DEBUG**: 重复值检查详情、记录上报详情
- **INFO**: 批量清理完成统计
- **ERROR**: 未预期的异常（应该很少发生）

### 性能考虑
- **锁粒度**: 点位级别锁，不是全局锁
- **数据结构**: 使用 ConcurrentDictionary 提高并发性能
- **清理策略**: 每次检查前先清理过期记录

## 依赖服务

**无直接依赖**，服务独立运行：
- 仅依赖 Microsoft.Extensions.Logging
- 使用内存存储，无数据库依赖
- 单例模式，全局共享缓存

## 相关文档

- [[PointValueProcessingService]] - 点位值处理服务
- [[DataBindingItemService]] - 数据绑定项服务
- [[数据绑定架构]] - 数据绑定整体架构
- [[重复值过滤策略]] - 缓存策略配置说明
