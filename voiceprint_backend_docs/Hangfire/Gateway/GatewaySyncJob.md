# 网关同步任务 (GatewaySyncJob)

## 概述
网关同步任务定期同步网关配置和状态信息，确保系统与网关数据一致。

## 配置

### appsettings.json

```json
{
  "GatewaySyncJob": {
    "Enabled": true,
    "CronExpression": "0 */10 * * * *",
    "SyncConfig": true,
    "SyncStatus": true,
    "SyncDevices": true,
    "SyncPoints": true
  }
}
```

## 执行流程

```
查询网关列表 → 同步配置 → 同步状态 → 同步设备 → 同步点位 → 记录结果
```

## 同步内容

### 1. 配置同步
- 点位配置
- 告警规则
- 采集策略
- 时间配置

### 2. 状态同步
- 网关在线状态
- 设备在线状态
- 通信状态
- 电池电量

### 3. 设备同步
- 设备信息
- 设备类型
- 设备位置
- 设备状态

### 4. 点位同步
- 点位列表
- 点位配置
- 点位状态
- 点位数据

## 依赖服务

- [[GatewayService]] - 网关服务
- [[DeviceService]] - 设备服务
- [[MonitoredPointService]] - 监测点位服务
- [[MqttService]] - MQTT 服务

## 相关任务

- [[VisualGatewayHealthCheckJob]] - 可视化网关健康检查

## 配置项

```json
{
  "GatewaySyncJob": {
    "Enabled": true,
    "CronExpression": "0 */10 * * * *",
    "SyncConfig": true,
    "SyncStatus": true,
    "SyncDevices": true,
    "SyncPoints": true,
    "TimeoutSeconds": 60
  }
}
```

## 注意事项

1. **同步频率**：不宜过高，避免影响网关性能
2. **冲突处理**：本地配置优先还是网关配置优先
3. **错误处理**：单个网关同步失败不影响其他网关
4. **日志记录**：记录同步变化的详细内容
5. **性能监控**：监控同步任务的执行时间

## 相关文档

- [[GatewayService]] - 网关服务文档
- [[MqttService]] - MQTT 服务文档
