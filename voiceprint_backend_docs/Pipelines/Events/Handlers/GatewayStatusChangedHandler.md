# GatewayStatusChangedHandler — 网关状态变更事件处理器

## 基本信息

- **处理器名称**：`GatewayStatusChangedHandler`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/`
- **事件类型**：`GatewayStatusChangedEventArgs` (ILocalEventHandler)
- **生命周期**：`ITransientDependency` - 瞬态依赖

## 处理器概述

GatewayStatusChangedHandler 负责处理网关状态变更事件，当网关的状态发生变化时，该处理器会更新数据库中对应网关实体的状态信息。

## 核心职责

1. **状态更新** - 更新网关的当前状态
2. **心跳记录** - 记录网关的最后心跳时间
3. **状态验证** - 检查状态是否真的发生了变化
4. **日志记录** - 记录状态变更的详细信息

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ISqlSugarRepository<GatewayAggregateRoot, Guid>` | 网关实体仓储 |
| `ILogger<GatewayStatusChangedHandler>` | 日志记录器 |

## 事件数据结构

### GatewayStatusChangedEventArgs

```csharp
public class GatewayStatusChangedEventArgs
{
    public Guid GatewayId { get; set; }          // 网关ID
    public string GatewayName { get; set; }      // 网关名称
    public GatewayStatusEnum Status { get; set; } // 新状态
    public string Reason { get; set; }          // 状态变更原因
    public DateTime ChangeTime { get; set; }     // 变更时间
}
```

## 处理流程

```
1. 记录事件开始
   │
   ├─ 记录日志：网关ID、网关名称、新状态、原因
   │
2. 查询网关实体
   │
   ├─ 通过 GatewayId 查询数据库
   │
   ├─ 如果网关不存在
   │  ├─ 记录警告日志
   │  └─ 直接返回
   │
3. 状态变化检查
   │
   ├─ 比较当前状态与新状态
   │
   ├─ 如果状态未变化
   │  ├─ 只更新心跳时间
   │  ├─ 保存到数据库
   │  ├─ 记录调试日志
   │  └─ 直接返回
   │
4. 更新网关状态
   │
   ├─ 记录旧状态
   ├─ 设置新状态
   ├─ 更新心跳时间
   ├─ 保存到数据库
   │
5. 记录成功日志
   │
   └─ 输出：网关ID、名称、旧状态、新状态、原因
```

## 核心方法

### HandleEventAsync - 处理状态变更事件

**签名**：
```csharp
public async Task HandleEventAsync(GatewayStatusChangedEventArgs eventData)
```

**功能**：
处理网关状态变更事件的核心方法。

**处理逻辑**：

```csharp
// 1. 记录事件开始
_logger.LogInformation("处理网关状态变更事件: 网关ID={GatewayId}, 网关名称={GatewayName}, " +
    "新状态={Status}, 原因={Reason}", 
    eventData.GatewayId, eventData.GatewayName, eventData.Status, eventData.Reason);

// 2. 查询网关实体
var gateway = await _gatewayRepository.GetByIdAsync(eventData.GatewayId);
if (gateway == null)
{
    _logger.LogWarning("网关不存在: {GatewayId}", eventData.GatewayId);
    return;
}

// 3. 检查状态是否真的发生了变化
if (gateway.Status == eventData.Status)
{
    // 状态未变化，只更新心跳时间
    gateway.LastHeartbeatTime = DateTime.Now;
    await _gatewayRepository.UpdateAsync(gateway);
    _logger.LogDebug("网关状态未发生变化: {GatewayId}, 当前状态={Status}", 
        eventData.GatewayId, gateway.Status);
    return;
}

// 4. 更新状态
var oldStatus = gateway.Status;
gateway.Status = eventData.Status;
gateway.LastHeartbeatTime = eventData.ChangeTime;
await _gatewayRepository.UpdateAsync(gateway);

// 5. 记录成功日志
_logger.LogInformation("网关状态更新成功: 网关ID={GatewayId}, 网关名称={GatewayName}, " +
    "状态从 {OldStatus} 变更为 {NewStatus}, 原因={Reason}", 
    eventData.GatewayId, eventData.GatewayName, oldStatus, gateway.Status, eventData.Reason);
```

## 业务规则

### 1. 状态变更条件

- 只有当新状态与当前状态不同时，才进行状态更新
- 如果状态相同，只更新心跳时间

### 2. 心跳时间更新

- 无论状态是否变化，都会更新心跳时间
- 使用事件中的 `ChangeTime` 作为心跳时间（状态变化时）
- 使用当前时间作为心跳时间（状态未变化时）

### 3. 网关不存在处理

- 如果网关不存在于数据库中，记录警告日志
- 不抛出异常，直接返回

## 日志记录

### 信息日志 (LogInformation)

```csharp
// 处理事件开始
"处理网关状态变更事件: 网关ID={GatewayId}, 网关名称={GatewayName}, 新状态={Status}, 原因={Reason}"

// 状态更新成功
"网关状态更新成功: 网关ID={GatewayId}, 网关名称={GatewayName}, 状态从 {OldStatus} 变更为 {NewStatus}, 原因={Reason}"
```

### 警告日志 (LogWarning)

```csharp
// 网关不存在
"网关不存在: {GatewayId}"
```

### 调试日志 (LogDebug)

```csharp
// 状态未发生变化
"网关状态未发生变化: {GatewayId}, 当前状态={Status}"
```

### 错误日志 (LogError)

```csharp
// 处理失败
"处理网关状态变更事件失败: 网关ID={GatewayId}, 新状态={Status}"
```

## 错误处理

### 异常处理

```csharp
try
{
    // 处理逻辑
}
catch (Exception ex)
{
    _logger.LogError(ex, "处理网关状态变更事件失败: 网关ID={GatewayId}, 新状态={Status}", 
        eventData.GatewayId, eventData.Status);
    throw;
}
```

- 捕获所有异常
- 记录详细的错误日志
- 重新抛出异常，让上层处理

## 相关实体

- [[GatewayAggregateRoot]] - 网关聚合根

## 相关枚举

- [[GatewayStatusEnum]] - 网关状态枚举

## 相关事件

- `GatewayStatusChangedEventArgs` - 网关状态变更事件参数

## 使用场景

### 1. 网关上线

当网关从离线状态变为在线状态：
```csharp
eventData.Status = GatewayStatusEnum.Online;
eventData.Reason = "设备启动";
```

### 2. 网关下线

当网关从在线状态变为离线状态：
```csharp
eventData.Status = GatewayStatusEnum.Offline;
eventData.Reason = "连接超时";
```

### 3. 网关故障

当网关检测到故障：
```csharp
eventData.Status = GatewayStatusEnum.Fault;
eventData.Reason = "硬件故障";
```

### 4. 心跳更新

当网关正常发送心跳但状态未变化：
```csharp
// 状态相同，只更新心跳时间
eventData.Status = gateway.Status;
```

## 性能考虑

1. **单条记录更新** - 每次事件只更新一个网关记录
2. **索引查询** - 通过主键ID查询，性能高效
3. **条件更新** - 只有状态真正变化时才更新

## 设计特点

1. **幂等性** - 相同状态的事件不会导致重复更新
2. **心跳追踪** - 即使状态不变也更新心跳时间
3. **详细日志** - 记录完整的状态变更信息
4. **容错性** - 网关不存在时优雅处理

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/GatewayStatusChangedHandler.cs`
