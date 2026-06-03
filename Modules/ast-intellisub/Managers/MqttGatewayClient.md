# MqttGatewayClient

**路径**: `module/ast-intellisub/Ast.IntelliSub.Application/Managers/MqttGatewayClient.cs`

**依赖**: `IDisposable` (手动管理)

## 概述

MQTT 网关客户端封装，为每个边缘网关提供独立的 MQTT 连接管理。支持持久会话、自动重连、健康检查和事件驱动架构，是物联网数据采集的核心组件。

## 核心功能

### 1. 连接管理

```csharp
public async Task<MqttClientConnectResult> ConnectAsync()
```

**职责**：
- 使用 MQTTnet 连接到代理服务器
- 支持持久会话（CleanSession=0）
- 固定 ClientID 以恢复会话状态
- 超时控制和异常处理

**连接特性**：
- 持久会话：断线重连后恢复订阅
- 保活机制：KeepAlivePeriod（默认 60s）
- 超时控制：ConnectionTimeout（默认 10s）

### 2. 订阅管理

```csharp
public async Task SubscribeAsync(string topic)
```

**职责**：
- 订阅指定主题
- 使用配置的 QoS 级别
- 自动处理订阅确认

**QoS 级别**：
- `0` - 最多一次
- `1` - 至少一次（默认）
- `2` - 只有一次

### 3. 消息发布

```csharp
public async Task PublishAsync(string topic, string payload)
```

**职责**：
- 发布消息到指定主题
- 支持字符串负载
- 异常捕获和日志记录

### 4. 断开连接

```csharp
public async Task DisconnectAsync()
```

**职责**：
- 优雅断开连接
- 清理资源
- 取消事件订阅（防止内存泄漏）

## 事件架构

### 连接事件

```csharp
public event Func<Guid, Task> Connected;
public event Func<Guid, Task> Disconnected;
public event Func<Guid, string, string, Task> MessageReceived;
```

**事件参数**：
- `Connected` - 网关 ID
- `Disconnected` - 网关 ID
- `MessageReceived` - 网关 ID、主题、负载

### 事件订阅示例

```csharp
client.Connected += async (gatewayId) =>
{
    _logger.LogInformation("网关 {GatewayId} 已连接", gatewayId);
    await SubscribeToTopicsAsync(gatewayId);
};

client.Disconnected += async (gatewayId) =>
{
    _logger.LogWarning("网关 {GatewayId} 已断开", gatewayId);
    await ScheduleReconnectAsync(gatewayId);
};

client.MessageReceived += async (gatewayId, topic, payload) =>
{
    await HandleMessageAsync(gatewayId, topic, payload);
};
```

## 配置选项

### MqttOptions

```json
{
  "Mqtt": {
    "ProtocolVersion": "V311",
    "KeepAlivePeriodSeconds": 60,
    "ConnectionTimeoutSeconds": 10,
    "QualityOfService": 1,
    "EnablePersistentSession": true,
    "HealthCheckIntervalMinutes": 5
  }
}
```

**字段说明**：
- `ProtocolVersion` - MQTT 协议版本（V310/V311/V500）
- `KeepAlivePeriodSeconds` - 保活间隔（秒）
- `ConnectionTimeoutSeconds` - 连接超时（秒）
- `QualityOfService` - QoS 级别（0/1/2）
- `EnablePersistentSession` - 启用持久会话
- `HealthCheckIntervalMinutes` - 健康检查间隔（分钟）

## 协议版本支持

| 版本 | 说明 | 使用场景 |
|------|------|----------|
| V310 | MQTT 3.1 | 老旧设备 |
| V311 | MQTT 3.1.1 | 通用标准（默认） |
| V500 | MQTT 5.0 | 新特性支持 |

## 使用示例

### 创建客户端

```csharp
var gateway = new GatewayAggregateRoot
{
    Id = Guid.NewGuid(),
    Name = "Gateway-01",
    Url = "mqtt://192.168.1.100:1883",
    Key = "gateway_user",
    Secret = "gateway_password"
};

var client = new MqttGatewayClient(gateway, _logger, _mqttOptions);
```

### 连接和订阅

```csharp
await client.ConnectAsync();

await client.SubscribeAsync("gateway/data/#");
await client.SubscribeAsync("gateway/command/+");
```

### 发布消息

```csharp
var payload = JsonConvert.SerializeObject(new
{
    timestamp = DateTime.UtcNow,
    temperature = 25.5,
    humidity = 60.2
});

await client.PublishAsync("gateway/data/sensor1", payload);
```

### 事件处理

```csharp
client.MessageReceived += async (gatewayId, topic, payload) =>
{
    _logger.LogInformation("收到消息: Gateway={GatewayId}, Topic={Topic}", 
        gatewayId, topic);
    
    var data = JsonConvert.DeserializeObject<SensorData>(payload);
    await SaveSensorDataAsync(gatewayId, data);
};
```

## 生命周期管理

### 初始化

```csharp
public MqttGatewayClient(
    GatewayAggregateRoot gateway, 
    ILogger logger, 
    MqttOptions mqttOptions = null)
{
    Gateway = gateway;
    _mqttOptions = mqttOptions ?? new MqttOptions();
    
    var mqttFactory = new MqttClientFactory();
    _mqttClient = mqttFactory.CreateMqttClient();
    
    InitializeEvents();
    BuildOptions();
}
```

### 事件初始化

```csharp
private void InitializeEvents()
{
    _connectedHandler = async (e) => await Connected?.Invoke(Gateway.Id);
    _disconnectedHandler = async (e) => await Disconnected?.Invoke(Gateway.Id);
    _messageReceivedHandler = async (e) => 
    {
        var payload = e.ApplicationMessage.ConvertPayloadToString();
        await MessageReceived?.Invoke(Gateway.Id, e.ApplicationMessage.Topic, payload);
    };
    
    _mqttClient.ConnectedAsync += _connectedHandler;
    _mqttClient.DisconnectedAsync += _disconnectedHandler;
    _mqttClient.ApplicationMessageReceivedAsync += _messageReceivedHandler;
}
```

### 选项构建

```csharp
private void BuildOptions()
{
    var clientId = $"IntelliSub-Gateway-{Gateway.Id}";
    var uri = new Uri(Gateway.Url);
    var cleanSession = !_mqttOptions.EnablePersistentSession;
    
    var protocolVersion = _mqttOptions.ProtocolVersion.ToUpper() switch
    {
        "V310" => MqttProtocolVersion.V310,
        "V311" => MqttProtocolVersion.V311,
        "V500" => MqttProtocolVersion.V500,
        _ => MqttProtocolVersion.V311
    };
    
    _options = new MqttClientOptionsBuilder()
        .WithTcpServer(uri.Host, uri.Port)
        .WithCredentials(Gateway.Key, Gateway.Secret)
        .WithClientId(clientId)
        .WithKeepAlivePeriod(TimeSpan.FromSeconds(_mqttOptions.KeepAlivePeriodSeconds))
        .WithCleanSession(cleanSession)
        .WithProtocolVersion(protocolVersion)
        .WithTimeout(TimeSpan.FromSeconds(_mqttOptions.ConnectionTimeoutSeconds))
        .Build();
}
```

### 资源清理

```csharp
public void Dispose()
{
    // 取消事件订阅
    _mqttClient.ConnectedAsync -= _connectedHandler;
    _mqttClient.DisconnectedAsync -= _disconnectedHandler;
    _mqttClient.ApplicationMessageReceivedAsync -= _messageReceivedHandler;
    
    _mqttClient?.Dispose();
}
```

## 错误处理

### 连接失败

```csharp
try
{
    var result = await _mqttClient.ConnectAsync(_options);
    if (result.ResultCode != MqttClientConnectResultCode.Success)
    {
        _logger.LogError("MQTT连接失败: {Code}", result.ResultCode);
    }
}
catch (Exception ex)
{
    _logger.LogError(ex, "MQTT连接异常");
}
```

### 订阅失败

```csharp
try
{
    await _mqttClient.SubscribeAsync(topic);
}
catch (Exception ex)
{
    _logger.LogError(ex, "订阅主题失败: {Topic}", topic);
}
```

### 发布失败

```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "发布消息失败: Topic={Topic}", topic);
    throw;
}
```

## 监控指标

### 连接状态

```csharp
public bool IsConnected => _mqttClient?.IsConnected ?? false;
```

### 网关信息

```csharp
public GatewayAggregateRoot Gateway { get; private set; }
```

## 日志示例

### 连接日志

```
网关 Gateway-01 MQTT配置: ClientID=IntelliSub-Gateway-abc123, 
CleanSession=False, QoS=1, Protocol=V311
网关 Gateway-01 已连接
```

### 消息日志

```
收到消息: Gateway=abc123, Topic=gateway/data/sensor1, 
Payload={"temperature":25.5,"humidity":60.2}
```

### 错误日志

```
MQTT连接失败: NotAuthorized
订阅主题失败: gateway/data/#
```

## 相关文档

- [MqttService](../../Services/MqttService.md) - MQTT 服务管理
- [网关管理](../../Services/GatewayService.md) - 网关设备管理
- [MQTT 配置](../../Configuration/MqttConfiguration.md)
