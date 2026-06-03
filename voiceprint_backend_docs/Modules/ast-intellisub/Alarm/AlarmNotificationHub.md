# 告警通知 Hub (AlarmNotificationHub)

## 概述
告警通知 Hub 是 SignalR Hub，负责实时推送告警消息到前端客户端。

## 职责
- 管理告警推送连接
- 推送告警触发消息
- 推送告警恢复消息
- 推送告警处理状态
- 管理订阅关系

## Hub 配置

### Hub 名称
`AlarmNotificationHub`

### 连接地址
`/hubs/alarm-notification`

## 推送方法

### OnAlarmTriggered
告警触发推送

**数据格式**：
```json
{
  "alarmId": "guid",
  "deviceName": "1#电机",
  "pointName": "振动监测",
  "alarmLevel": "Critical",
  "alarmValue": 8.5,
  "threshold": 5.0,
  "alarmTime": "2026-06-03T10:00:00Z",
  "message": "振动值超标：8.5 mm/s"
}
```

### OnAlarmRecovered
告警恢复推送

**数据格式**：
```json
{
  "alarmId": "guid",
  "deviceName": "1#电机",
  "recoveredTime": "2026-06-03T11:00:00Z",
  "message": "告警已恢复"
}
```

### OnAlarmHandled
告警处理推送

**数据格式**：
```json
{
  "alarmId": "guid",
  "handledBy": "admin",
  "handledAt": "2026-06-03T10:30:00Z",
  "handleNote": "已处理"
}
```

## 订阅管理

### 订阅设备告警
```csharp
await Groups.AddToGroupAsync(Context.ConnectionId, deviceId);
```

### 取消订阅
```csharp
await Groups.RemoveFromGroupAsync(Context.ConnectionId, deviceId);
```

## 连接事件

### OnConnectedAsync
```csharp
public override async Task OnConnectedAsync()
{
    // 记录连接
    await _connectionService.RecordConnectionAsync(Context.ConnectionId);

    // 获取用户订阅的设备
    var deviceIds = await GetUserSubscribedDevices();
    foreach (var deviceId in deviceIds)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, deviceId);
    }
}
```

### OnDisconnectedAsync
```csharp
public override async Task OnDisconnectedAsync(Exception exception)
{
    // 清理连接
    await _connectionService.RemoveConnectionAsync(Context.ConnectionId);
}
```

## 推送逻辑

### 告警触发推送
```csharp
await _hubContext.Clients.Group(deviceId)
    .SendAsync("OnAlarmTriggered", new
    {
        AlarmId = alarm.Id,
        DeviceName = device.Name,
        PointName = point.Name,
        AlarmLevel = alarm.Level,
        AlarmValue = alarm.Value,
        Threshold = alarm.Threshold,
        AlarmTime = alarm.Time,
        Message = $"告警：{alarm.Value} {alarm.Unit}"
    });
```

### 全局推送
```csharp
await _hubContext.Clients.All
    .SendAsync("OnAlarmTriggered", alarmData);
```

## 依赖服务

- [[AlarmNotificationService]] - 告警通知服务
- [[AlarmCategoryService]] - 告警分类服务
- [[DeviceService]] - 设备服务

## 配置项

```json
{
  "SignalR": {
    "EnableDetailedErrors": false,
    "HubTimeout": 30,
    "KeepAliveInterval": 15,
    "HandshakeTimeout": 15
  }
}
```

## 前端使用示例

```typescript
// 建立连接
const connection = new HubConnectionBuilder()
  .withUrl('/hubs/alarm-notification')
  .withAutomaticReconnect()
  .build();

// 订阅告警触发
connection.on('OnAlarmTriggered', (alarm) => {
  console.log(`告警触发: ${alarm.deviceName} - ${alarm.message}`);
  showNotification(alarm);
});

// 订阅告警恢复
connection.on('OnAlarmRecovered', (alarm) => {
  console.log(`告警恢复: ${alarm.deviceName}`);
});

// 启动连接
await connection.start();
```

## 相关文档

- [[AlarmAPI]] - 告警模块 API 文档
- [[告警管理流程]] - 告警管理流程时序图
- [[AlarmNotificationService]] - 告警通知服务

## 注意事项

1. **连接管理**：需要处理连接断开和重连
2. **权限控制**：用户只能接收有权限设备的告警
3. **消息过滤**：避免过度推送影响用户体验
4. **性能优化**：大量告警时需要批量推送
5. **离线消息**：支持离线期间的告警缓存
