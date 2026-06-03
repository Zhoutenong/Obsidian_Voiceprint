# 实时监测点位服务 (RealtimeMonitoringPointService)

## 概述
实时监测点位服务提供实时点位数据的查询和推送功能，是实时监控的核心服务。

## 职责
- 实时点位数据查询
- WebSocket 实时数据推送
- 点位数据缓存
- 批量数据查询
- 数据订阅管理

## 主要接口

### 获取最新点位数据
```csharp
Task<MonitoredPointLatestDataDto> GetLatestDataAsync(Guid pointId)
```

**返回数据**：
- `PointId`: 点位ID
- `Value`: 当前值
- `Timestamp`: 时间戳
- `Quality`: 数据质量
- `Unit`: 单位
- `Status`: 点位状态

### 批量获取点位数据
```csharp
Task<List<MonitoredPointLatestDataDto>> GetBatchDataAsync(List<Guid> pointIds)
```

### 获取历史数据
```csharp
Task<List<MonitoredPointHistoryDataDto>> GetHistoryDataAsync(Guid pointId, DateTime startTime, DateTime endTime)
```

### 订阅点位数据更新
```csharp
Task SubscribeAsync(Guid pointId, string connectionId)
```

### 取消订阅
```csharp
Task UnsubscribeAsync(Guid pointId, string connectionId)
```

## SignalR Hub

### Hub 名称
`RealtimeDataHub`

### 连接地址
`/hubs/realtime-data`

### 推送方法

#### OnPointDataUpdate
点位数据更新推送

**数据格式**：
```json
{
  "pointId": "guid",
  "value": 25.5,
  "timestamp": "2026-06-03T10:30:00Z",
  "quality": "Good",
  "unit": "°C"
}
```

#### OnPointStatusChange
点位状态变化推送

**数据格式**：
```json
{
  "pointId": "guid",
  "oldStatus": "Active",
  "newStatus": "Fault",
  "timestamp": "2026-06-03T10:30:00Z"
}
```

#### OnPointAlarm
点位告警推送

**数据格式**：
```json
{
  "pointId": "guid",
  "alarmType": "Upper",
  "alarmLevel": "Critical",
  "value": 85.5,
  "threshold": 80.0,
  "timestamp": "2026-06-03T10:30:00Z"
}
```

## 数据质量

| 质量 | 说明 | 代码 |
|-----|------|------|
| Good | 数据有效 | 192 |
| Uncertain | 数据不确定 | 64 |
| Bad | 数据无效 | 0 |
| Overflow | 数据溢出 | 32 |
| CommFailure | 通信失败 | 8 |

## 缓存策略

### Redis 缓存
```csharp
// 缓存键格式
point:latest:{pointId}

// 缓存内容
{
  "value": 25.5,
  "timestamp": "2026-06-03T10:30:00Z",
  "quality": "Good"
}

// 缓存过期
- 活跃点位：5 分钟
- 非活跃点位：30 分钟
```

### 缓存更新
- 数据更新时写入缓存
- 定时刷新过期数据
- 订阅时立即刷新

## 数据推送流程

```
传感器数据更新 → 保存数据库 → 更新缓存 → 查询订阅列表 → SignalR 推送
```

### 推送优化
- 只推送订阅的点位
- 批量推送（积攒多个更新）
- 推送频率限制（避免过度推送）
- 连接断开自动取消订阅

## 订阅管理

### 订阅状态
- 维护订阅关系（ConnectionId ↔ PointId）
- 心跳检测
- 自动清理过期订阅

### 订阅场景
1. **实时监控页面** - 订阅当前页面显示的点位
2. **设备详情页面** - 订阅该设备的所有点位
3. **告警处理页面** - 订阅告警相关的点位
4. **移动应用** - 订阅关注的点位

## 性能优化

### 查询优化
- 使用缓存数据
- 批量查询接口
- 查询结果缓存

### 推送优化
- 批量推送消息
- 推送节流（100ms）
- 只推送变化的数据

### 连接管理
- 限制单个连接订阅数量
- 定期清理无效连接
- 连接复用

## 依赖服务

- [[MonitoredPointService]] - 监测点位服务
- [[PointDataService]] - 点位数据服务
- [[SignalR]] - 实时通信框架
- [[Redis]] - 分布式缓存

## 相关实体

- [[MonitoredPoint]] - 监测点位
- [[PointData]] - 点位数据
- [[Alarm]] - 告警记录

## 配置项

```json
{
  "RealtimeData": {
    "CacheExpiration": 300,
    "PushThrottle": 100,
    "MaxSubscriptionsPerConnection": 100,
    "HeartbeatInterval": 30000,
    "ConnectionTimeout": 60000
  }
}
```

## 使用示例

### 前端订阅
```typescript
// 建立连接
const connection = new HubConnectionBuilder()
  .withUrl('/hubs/realtime-data')
  .build();

// 订阅点位数据更新
connection.on('OnPointDataUpdate', (data) => {
  console.log(`点位 ${data.pointId} 更新: ${data.value}`);
});

// 启动连接
await connection.start();

// 订阅点位
await connection.invoke('Subscribe', pointId);
```

### 后端推送
```csharp
// 推送点位数据更新
await _hubContext.Clients.Group(pointId.ToString())
  .SendAsync("OnPointDataUpdate", new
  {
    PointId = pointId,
    Value = value,
    Timestamp = DateTime.UtcNow,
    Quality = "Good"
  });
```

## 相关文档

- [[MonitoredPointService]] - 监测点位服务
- [[PointDataService]] - 点位数据服务
- [[AlarmNotificationHub]] - 告警推送 Hub

## API 路径

- `GET /api/app/realtime-points/{pointId}/latest` - 获取最新点位数据
- `GET /api/app/realtime-points/batch` - 批量获取点位数据
- `GET /api/app/realtime-points/{pointId}/history` - 获取历史数据
- `POST /api/app/realtime-points/{pointId}/subscribe` - 订阅点位更新
- `DELETE /api/app/realtime-points/{pointId}/unsubscribe` - 取消订阅

## 注意事项

1. **连接限制**：单个连接订阅数量有限制
2. **推送频率**：高频数据会节流推送
3. **缓存一致性**：缓存可能短暂延迟
4. **连接管理**：需要处理连接断开重连
5. **权限控制**：只能订阅有权限的点位
