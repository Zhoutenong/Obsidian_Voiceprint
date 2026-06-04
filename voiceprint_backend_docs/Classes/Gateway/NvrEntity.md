# NvrEntity

## 概述

`NvrEntity` (Network Video Recorder Entity) 代表网络录像机设备。NVR 用于连接和管理多台摄像机，提供视频录制、存储和回放功能。

## 表信息

- **表名**: `ast_nvr`
- **主键**: `Id` (string)
- **继承**: `CreationAuditedEntity<string>`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `string` | `id` | 主键（设备唯一标识） | PK |
| `GatewayId` | `Guid` | `gateway_id` | 所属网关ID | FK |
| `Name` | `string?` | `name` | NVR名称 | Nullable |
| `Status` | `DeviceStatusEnum` | `status` | 在线状态 | Enum |
| `Type` | `NvrTypeEnum` | `type` | NVR类型 | Enum |
| `HostAddress` | `string?` | `host_address` | NVR IP地址或域名 | |
| `Username` | `string?` | `username` | 登录用户名 | 加密存储 |
| `Password` | `string?` | `password` | 登录密码 | 加密存储 |
| `ChannelCount` | `int?` | `channel_count` | 摄像头通道数 | Default: 0 |
| `Manufacturer` | `string?` | `manufacturer` | 设备厂家 | |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | 继承 |
| `IsVirtual` | `bool` | `is_virtual` | 是否虚拟设备 | Default: false |
| `IsDeleted` | `bool` | `is_deleted` | 是否删除 | 软删除标记 |

## 关联实体 (ER 关系)

### 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(GatewayId))]
public GatewayAggregateRoot Gateway { get; set; }
```

### 关系图

```
GatewayAggregateRoot (1) ←──→ (N) NvrEntity
                                      │
                                      └── (N) DeviceEntity (通过 NvrId 关联)
```

## 被哪些服务读写

### 写入服务

- `GatewayService` — NVR 设备 CRUD
- `DeviceService` — 摄像机设备管理（关联 NVR）

### 读取服务

- `MediaService` — 视频流、录像回放
- `StreamingApiService` — 实时视频流处理
- `PatrolExecutionService` — 巡检任务视频采集

## 业务规则约束

### 1. 索引约束

```csharp
[SugarIndex($"IX_{nameof(GatewayId)}", nameof(GatewayId), OrderByType.Asc)]
[SugarIndex($"IX_{nameof(Status)}", nameof(Status), OrderByType.Asc)]
[SugarIndex($"IX_{nameof(Type)}", nameof(Type), OrderByType.Asc)]
```

对应的 SQL 索引：

```sql
CREATE INDEX IX_NVR_GATEWAY_ID ON ast_nvr(gateway_id);
CREATE INDEX IX_NVR_STATUS ON ast_nvr(status);
CREATE INDEX IX_NVR_TYPE ON ast_nvr(type);
```

### 2. 枚举约束

#### DeviceStatusEnum

```csharp
public enum DeviceStatusEnum
{
    Offline = 0,      // 离线
    Online = 1,       // 在线
    Fault = 2         // 故障
}
```

#### NvrTypeEnum

```csharp
public enum NvrTypeEnum
{
    Standard = 0,     // 标准NVR
    Embedded = 1,     // 嵌入式NVR
    Hybrid = 2,       // 混合NVR
    Virtual = 3       // 虚拟NVR（模拟）
}
```

### 3. 通道数限制

- `ChannelCount` 表示 NVR 支持的最大摄像头数量
- 常见值：4、8、16、32、64
- 关联的 `DeviceEntity` 数量不应超过 `ChannelCount`

### 4. 虚拟设备

- `IsVirtual = true` 表示测试或演示用的虚拟 NVR
- 虚拟 NVR 不实际连接物理设备
- 用于系统测试和培训场景

### 5. 软删除

- `IsDeleted = true` 表示已删除（软删除）
- 查询时需过滤 `IsDeleted = false` 的记录
- 物理删除由后台任务定期执行

### 6. 安全约束

- `Username` 和 `Password` 应加密存储
- 不得在日志或错误消息中暴露密码
- 密码应符合 NVR 设备的复杂度要求

## 使用场景

### 1. 创建海康威视 NVR

```csharp
var nvr = new NvrEntity
{
    Id = "NVR_HIK_001",
    GatewayId = gatewayId,
    Name = "1号机房NVR",
    Status = DeviceStatusEnum.Online,
    Type = NvrTypeEnum.Standard,
    HostAddress = "192.168.1.100",
    Username = "admin",
    Password = "encrypted_password",
    ChannelCount = 16,
    Manufacturer = "HIKVISION",
    IsVirtual = false
};
```

### 2. 创建虚拟 NVR（测试）

```csharp
var virtualNvr = new NvrEntity
{
    Id = "NVR_VIRTUAL_001",
    GatewayId = testGatewayId,
    Name = "测试NVR",
    Status = DeviceStatusEnum.Online,
    Type = NvrTypeEnum.Virtual,
    ChannelCount = 4,
    IsVirtual = true
};
```

## 级联操作

### 删除规则

1. 删除 NVR 前需检查：
   - 是否有关联的 `DeviceEntity`（摄像机）
   - 是否有正在进行的录像任务

2. 建议流程：
   ```csharp
   // 1. 标记为删除（软删除）
   nvr.IsDeleted = true;
   
   // 2. 解绑所有摄像机
   var cameras = await _deviceRepo.GetListAsync(d => d.NvrId == nvr.Id);
   cameras.ForEach(c => c.NvrId = null);
   
   // 3. 更新状态
   await _deviceRepo.UpdateAsync(cameras);
   await _nvrRepo.UpdateAsync(nvr);
   ```

## 索引优化建议

```sql
-- 复合索引（网关 + 删除标记）
CREATE INDEX IX_NVR_GATEWAY_DELETED ON ast_nvr(gateway_id, is_deleted);

-- 厂商索引（用于按厂商筛选）
CREATE INDEX IX_NVR_MANUFACTURER ON ast_nvr(manufacturer);
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Gateway/NvrEntity.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/GatewayService.cs`
- **媒体服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/MediaService.cs`
- **流服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/StreamingApiService.cs`
