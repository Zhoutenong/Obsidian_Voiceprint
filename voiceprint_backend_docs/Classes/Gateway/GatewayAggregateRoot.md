# GatewayAggregateRoot

## 概述

`GatewayAggregateRoot` 是变电站监控系统的核心聚合根，代表边缘网关设备。网关是连接现场设备（传感器、摄像机）与后端系统的桥梁，负责数据采集、协议转换和设备管理。

## 表信息

- **表名**: `ast_gateway`
- **主键**: `Id` (Guid)
- **继承**: `AggregateRoot<Guid>` + `IAuditedObject`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `Guid` | `id` | 主键 | PK |
| `SubstationId` | `Guid` | `substation_id` | 所属站点ID | FK |
| `Key` | `string` | `key` | 网关唯一标识 | Unique |
| `Name` | `string` | `name` | 显示名称 | NOT NULL |
| `Url` | `string` | `url` | 网关地址 | |
| `Protocol` | `GatewayProtocolEnum` | `protocol` | 协议类型 | Enum |
| `Secret` | `string` | `secret` | 网关密钥 | 加密存储 |
| `Status` | `GatewayStatusEnum` | `status` | 在线状态 | Enum (0:离线,1:在线) |
| `Type` | `GatewayTypeEnum` | `type` | 网关类型 | Enum (0:状态监测,1:视觉) |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | |
| `LastHeartbeatTime` | `DateTime?` | `last_heartbeat_time` | 最后心跳时间 | Nullable |
| `CreatorId` | `Guid?` | `creator_id` | 创建者ID | FK (User) |
| `LastModificationTime` | `DateTime?` | `last_modification_time` | 最后修改时间 | Nullable |
| `LastModifierId` | `Guid?` | `last_modifier_id` | 最后修改者ID | FK (User) |

## 关联实体 (ER 关系)

### 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(SubstationId))]
public SubstationAggregateRoot Substation { get; set; }
```

### 关系图

```
SubstationAggregateRoot (1) ←──→ (N) GatewayAggregateRoot
                                         │
                                         ├── (1) SensorEntity (通过网关关联)
                                         ├── (1) NvrEntity
                                         └── (N) DeviceEntity
```

## 被哪些服务读写

### 写入服务

- `GatewayService` — 网关 CRUD、在线状态更新
- `StreamingGatewayService` — 流媒体网关管理

### 读取服务

- `GatewayService` — 网关查询、列表获取
- `RealtimeMonitoringPointService` — 实时监控点位数据
- `PatrolExecutionService` — 巡检任务执行时访问网关

## 业务规则约束

### 1. 唯一性约束

- `Key` 字段必须在全局范围内唯一，用于网关身份识别

### 2. 类型约束

- **Protocol**: 协议类型必须为 `GatewayProtocolEnum` 枚举值
- **Status**: 必须为 `GatewayStatusEnum` 枚举值
  - `Offline = 0` — 网关离线
  - `Online = 1` — 网关在线
- **Type**: 必须为 `GatewayTypeEnum` 枚举值
  - `StatusMonitoring = 0` — 状态监测网关（传感器数据采集）
  - `Vision = 1` — 视觉网关（视频/图像采集）

### 3. 心跳机制

- `LastHeartbeatTime` 通过 Hangfire 后台任务定期更新
- 网关离线判断：`LastHeartbeatTime` 超过配置的超时时间（默认 5 分钟）

### 4. 安全约束

- `Secret` 字段应加密存储，用于与网关设备通信认证
- 不得在日志或错误消息中暴露密钥

### 5. 级联关系

- 删除站点时需检查是否有关联网关
- 删除网关前需确保无关联的传感器、NVR 和设备

## 枚举定义

### GatewayProtocolEnum

```csharp
public enum GatewayProtocolEnum
{
    TCP = 0,
    HTTP = 1,
    HTTPS = 2,
    MQTT = 3
}
```

### GatewayStatusEnum

```csharp
public enum GatewayStatusEnum
{
    Offline = 0,
    Online = 1
}
```

### GatewayTypeEnum

```csharp
public enum GatewayTypeEnum
{
    StatusMonitoring = 0,  // 状态监测网关
    Vision = 1             // 视觉网关
}
```

## 索引建议

```sql
-- 建议索引
CREATE INDEX IX_GATEWAY_SUBSTATION_ID ON ast_gateway(substation_id);
CREATE UNIQUE INDEX UX_GATEWAY_KEY ON ast_gateway(key);
CREATE INDEX IX_GATEWAY_STATUS ON ast_gateway(status);
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Gateway/GatewayAggregateRoot.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/GatewayService.cs`
- **DTO**: `module/ast-intellisub/Ast.IntelliSub.Application.Contracts/Dtos/Gateway/`
