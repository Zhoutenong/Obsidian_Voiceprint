# 网关服务 (GatewayService)

## 概述
网关服务负责管理边缘网关设备，处理与网关的通信，是系统与边缘设备之间的桥梁。

## 职责
- 管理网关设备信息
- 处理网关连接
- 同步网关配置
- 监控网关状态
- 处理网关数据上报

## 主要接口

### 创建网关
```csharp
Task<GatewayDto> CreateAsync(CreateGatewayDto input)
```

**请求参数**：
- `Name`: 网关名称
- `Code`: 网关编码
- `Ip`: IP 地址
- `Port`: 端口
- `Protocol`: 通信协议
- `Username`: 用户名
- `Password`: 密码
- `Location`: 位置

### 更新网关
```csharp
Task<GatewayDto> UpdateAsync(Guid id, UpdateGatewayDto input)
```

### 删除网关
```csharp
Task DeleteAsync(Guid id)
```

### 获取网关列表
```csharp
Task<PagedResultDto<GatewayDto>> GetListAsync(GetGatewayListDto input)
```

### 获取网关详情
```csharp
Task<GatewayDetailDto> GetDetailAsync(Guid id)
```

### 测试连接
```csharp
Task<bool> TestConnectionAsync(Guid id)
```

### 同步配置
```csharp
Task SyncConfigAsync(Guid id)
```

## 网关状态

| 状态 | 说明 |
|-----|------|
| Online | 在线 |
| Offline | 离线 |
| Fault | 故障 |
| Maintenance | 维护中 |

## 通信协议

### 支持的协议

| 协议 | 说明 | 默认端口 |
|-----|------|---------|
| MQTT | 消息队列遥测传输 | 1883 |
| TCP | TCP 通信 | 8080 |
| UDP | UDP 通信 | 8080 |
| HTTP | HTTP 通信 | 80 |
| HTTPS | HTTPS 通信 | 443 |

### MQTT 配置

```json
{
  "Mqtt": {
    "Broker": "localhost",
    "Port": 1883,
    "Username": "gateway",
    "Password": "***",
    "ClientId": "intellisub-server",
    "TopicPrefix": "intellisub",
    "Qos": 1,
    "KeepAlive": 60
  }
}
```

## 网关数据上报

### 数据上报流程

```
边缘设备 → 网关采集 → MQTT 发布 → 服务端订阅 → 数据处理 → 保存数据库
```

### 上报数据类型

1. **传感器数据**
   - 振动数据
   - 温度数据
   - 噪声数据
   - 环境数据

2. **设备状态数据**
   - 设备在线状态
   - 设备健康状态
   - 电池电量
   - 信号强度

3. **告警数据**
   - 告警触发
   - 告警恢复
   - 告警确认

### MQTT Topic 结构

```
intellisub/{gatewayCode}/sensor/data          # 传感器数据
intellisub/{gatewayCode}/device/status       # 设备状态
intellisub/{gatewayCode}/alarm/trigger        # 告警触发
intellisub/{gatewayCode}/alarm/recover        # 告警恢复
intellisub/{gatewayCode}/config/response      # 配置响应
```

## 网关管理功能

### 配置同步
- 点位配置同步
- 告警规则同步
- 采集策略同步
- 时间同步

### 远程控制
- 远程重启
- 远程配置更新
- 远程数据采集
- 远程日志获取

### 监控功能
- 心跳检测
- 状态监控
- 数据质量监控
- 通信质量监控

## 异常处理

### 连接异常
- 连接失败 → 标记网关离线
- 连接超时 → 重试连接
- 连接断开 → 自动重连

### 数据异常
- 数据格式错误 → 记录日志，丢弃数据
- 数据超时 → 标记数据无效
- 数据缺失 → 使用默认值

### 网关异常
- 网关离线 → 停止数据接收
- 网关故障 → 触发告警
- 网关重启 → 自动恢复连接

## 依赖服务

- [[MqttService]] - MQTT 服务
- [[MqttMessageBackgroundService]] - MQTT 消息后台服务
- [[DeviceService]] - 设备服务
- [[MonitoredPointService]] - 监测点位服务

## 相关实体

- [[Gateway]] - 网关实体
- [[Device]] - 设备
- [[MonitoredPoint]] - 监测点位

## 配置项

```json
{
  "Gateway": {
    "HeartbeatInterval": 30000,
    "ConnectionTimeout": 10000,
    "MaxRetryCount": 3,
    "AutoReconnect": true,
    "DataBufferMaxSize": 10000
  }
}
```

## 相关文档

- [[MqttService]] - MQTT 服务文档
- [[GatewaySyncJob]] - 网关同步任务文档
- [[StreamingGatewayService]] - 流媒体网关服务文档

## API 路径

- `POST /api/app/gateways` - 创建网关
- `PUT /api/app/gateways/{id}` - 更新网关
- `DELETE /api/app/gateways/{id}` - 删除网关
- `GET /api/app/gateways` - 获取网关列表
- `GET /api/app/gateways/{id}` - 获取网关详情
- `POST /api/app/gateways/{id}/test-connection` - 测试连接
- `POST /api/app/gateways/{id}/sync-config` - 同步配置
- `POST /api/app/gateways/{id}/restart` - 远程重启

## 注意事项

1. **网络安全**：网关通信需要加密
2. **认证授权**：网关需要认证后才能连接
3. **数据安全**：敏感数据需要加密传输
4. **并发控制**：同一网关的并发操作需要同步
5. **连接池**：需要维护连接池，避免频繁创建连接
