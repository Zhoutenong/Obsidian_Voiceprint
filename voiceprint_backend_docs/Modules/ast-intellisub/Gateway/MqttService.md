# MQTT 服务 (MqttService)

## 概述
MQTT 服务负责管理 MQTT 连接，发布和订阅消息，是系统与边缘网关通信的核心服务。

## 职责
- 管理 MQTT 连接
- 消息发布
- 消息订阅
- 消息处理
- 连接状态监控

## 主要接口

### 发布消息
```csharp
Task PublishAsync(string topic, string payload, int qos = 1, bool retain = false)
```

### 订阅主题
```csharp
Task SubscribeAsync(string topic, int qos = 1)
```

### 取消订阅
```csharp
Task UnsubscribeAsync(string topic)
```

### 获取连接状态
```csharp
Task<MqttConnectionStatus> GetStatusAsync()
```

## MQTT 配置

### 客户端配置

```json
{
  "MqttClient": {
    "Broker": "localhost",
    "Port": 1883,
    "ClientId": "intellisub-server",
    "Username": "intellisub",
    "Password": "***",
    "CleanSession": true,
    "KeepAlive": 60,
    "Timeout": 10000,
    "AutoReconnect": true,
    "AutoReconnectDelay": 5000
  }
}
```

### Topic 结构

```
intellisub/{gatewayCode}/sensor/data          # 传感器数据上报
intellisub/{gatewayCode}/device/status       # 设备状态上报
intellisub/{gatewayCode}/alarm/trigger        # 告警触发
intellisub/{gatewayCode}/alarm/recover        # 告警恢复
intellisub/{gatewayCode}/config/request       # 配置请求
intellisub/{gatewayCode}/config/response      # 配置响应
intellisub/{gatewayCode}/command/execute      # 命令执行
intellisub/{gatewayCode}/command/result       # 命令结果
intellisub/+/+/status                         # 全网关状态（通配符）
```

## 消息处理

### 订阅消息

系统订阅以下 Topic：

```csharp
// 订阅所有网关的传感器数据
SubscribeAsync("intellisub/+/sensor/data");

// 订阅所有网关的设备状态
SubscribeAsync("intellisub/+/device/status");

// 订阅所有网关的告警
SubscribeAsync("intellisub/+/alarm/#");

// 订阅配置响应
SubscribeAsync("intellisub/+/config/response");
```

### 消息处理流程

```
消息接收 → Topic 解析 → 消息验证 → 数据转换 → 业务处理 → 结果返回
```

### 消息格式

#### 传感器数据消息
```json
{
  "messageId": "guid",
  "gatewayCode": "GW001",
  "deviceId": "guid",
  "pointId": "guid",
  "value": 25.5,
  "unit": "°C",
  "timestamp": "2026-06-03T10:30:00Z",
  "quality": "Good"
}
```

#### 设备状态消息
```json
{
  "messageId": "guid",
  "gatewayCode": "GW001",
  "deviceId": "guid",
  "status": "Online",
  "timestamp": "2026-06-03T10:30:00Z"
}
```

#### 告警消息
```json
{
  "messageId": "guid",
  "gatewayCode": "GW001",
  "alarmType": "Upper",
  "alarmLevel": "Critical",
  "deviceId": "guid",
  "pointId": "guid",
  "value": 85.5,
  "threshold": 80.0,
  "timestamp": "2026-06-03T10:30:00Z"
}
```

## QoS 级别

| QoS | 说明 | 使用场景 |
|-----|------|---------|
| 0 | 最多一次 | 频繁更新的传感器数据 |
| 1 | 至少一次 | 告警消息、配置消息 |
| 2 | 恰好一次 | 命令消息（极少使用） |

## 连接管理

### 连接状态

| 状态 | 说明 |
|-----|------|
| Disconnected | 未连接 |
| Connecting | 连接中 |
| Connected | 已连接 |
| Disconnecting | 断开中 |

### 自动重连
- 连接断开时自动重连
- 重连延迟：5秒
- 最大重连次数：无限制
- 重连成功后自动重新订阅

### 心跳保活
- Keep Alive：60秒
- 超时未收到心跳：自动断开
- 重连后重新订阅

## 消息队列

### 消息缓冲
- 离线消息缓冲（CleanSession=false）
- 缓冲最大消息数：1000
- 缓冲时间：24小时

### 发布确认
- QoS 1/2 消息需要确认
- 未确认消息重发
- 重发间隔：1秒

## 异常处理

### 连接异常
- **连接失败**：记录日志，定时重连
- **连接超时**：增加超时时间，重试
- **连接断开**：自动重连，重发未确认消息

### 消息异常
- **消息格式错误**：记录日志，丢弃消息
- **消息处理失败**：记录日志，发送错误响应
- **消息超时**：标记消息失效

### 网络异常
- **网络中断**：保持重连
- **网络恢复**：自动重新连接
- **网络抖动**：增加超时时间

## 依赖服务

- [[MqttMessageBackgroundService]] - MQTT 消息后台服务
- [[GatewayService]] - 网关服务
- [[DeviceService]] - 设备服务
- [[MonitoredPointService]] - 监测点位服务

## 配置项

```json
{
  "Mqtt": {
    "Broker": "localhost",
    "Port": 1883,
    "ClientId": "intellisub-server",
    "Username": "intellisub",
    "Password": "***",
    "CleanSession": true,
    "KeepAlive": 60,
    "Timeout": 10000,
    "AutoReconnect": true,
    "AutoReconnectDelay": 5000,
    "MaxMessageQueueSize": 1000,
    "MessageRetentionHours": 24
  }
}
```

## 使用示例

### 发布消息
```csharp
await _mqttService.PublishAsync(
    $"intellisub/{gatewayCode}/command/execute",
    JsonConvert.SerializeObject(command),
    qos: 1
);
```

### 处理接收消息
```csharp
private async Task OnMessageReceived(MqttApplicationMessage message)
{
    var topic = message.Topic;
    var payload = Encoding.UTF8.GetString(message.Payload);

    // 解析 Topic
    var topicParts = topic.Split('/');
    var gatewayCode = topicParts[1];
    var messageType = topicParts[2];

    // 根据消息类型处理
    switch (messageType)
    {
        case "sensor":
            await HandleSensorData(gatewayCode, payload);
            break;
        case "alarm":
            await HandleAlarm(gatewayCode, payload);
            break;
    }
}
```

## 相关文档

- [[GatewayService]] - 网关服务文档
- [[MqttMessageBackgroundService]] - MQTT 消息后台服务文档
- [[GatewaySyncJob]] - 网关同步任务文档

## 注意事项

1. **消息大小**：单条消息大小不超过 256KB
2. **订阅数量**：订阅主题数量不超过 100
3. **消息频率**：高频消息考虑消息合并
4. **安全性**：生产环境使用 TLS 加密
5. **权限控制**：使用用户名密码认证

## 性能优化

### 消息批处理
- 合并高频消息
- 批量发布消息
- 减少网络开销

### 连接复用
- 使用长连接
- 避免频繁创建连接
- 连接池管理

### 消息压缩
- 大数据量消息压缩
- 使用 Gzip 压缩
- 减少带宽占用
