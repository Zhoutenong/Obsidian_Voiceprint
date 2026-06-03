# 摄像机服务 (CameraService)

## 概述

摄像机服务负责管理变电站监控系统中的摄像机设备，提供设备树形结构查询、设备信息更新等核心功能。

**位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/CameraService.cs`  
**层**：Application  
**模块**：ast-intellisub  
**依赖注入**：Scoped

## 职责

- 管理摄像机设备信息
- 构建巡视设备树形结构
- 提供摄像机设备更新功能
- 集成流媒体 URL 管理
- 关联预置点和监测对象信息

## 主要接口

### 获取巡视设备树

```csharp
Task<CameraTreeDto> GetTreeAsync(Guid substationId, Guid? gatewayId = null, bool? onlyEnabledPresets = null)
```

**功能**：获取指定变电站的完整巡视设备树形结构

**请求参数**：
- `substationId`：变电站ID（必填）
- `gatewayId`：网关ID（可选，用于过滤）
- `onlyEnabledPresets`：是否只返回启用的预置点位（可选）

**返回数据**：
```csharp
class CameraTreeDto
{
    List<NvrDto> Nvrs              // 录像机列表
    List<CameraDeviceDto> Cameras  // 摄像头列表（含流媒体URL）
    List<MonitoredObjectSimpleDto> MonitoredObjects  // 监测对象列表
    List<PresetDto> Presets        // 预置点列表
}
```

**业务逻辑**：
1. 查询录像机信息（通过网关关联）
2. 查询摄像头设备（type=1）
3. 查询监测对象信息（过滤分组类型）
4. 查询预置点信息（type=1）
5. 为每个摄像头构建流媒体URL

### 更新摄像机设备

```csharp
Task<DeviceDto> UpdateAsync(string id, CameraUpdateDto input)
```

**请求参数**：
- `id`：摄像机设备ID
- `Name`：设备名称
- `MonitoredObjectId`：关联监测对象ID
- `CameraDeviceType`：摄像机设备类型
- `CameraVideoType`：视频类型
- `CameraPresetType`：预置位类型
- `CameraHostAddress`：主机地址
- `CameraUsername`：用户名
- `CameraPassword`：密码
- `CameraManufacturer`：制造商
- `Remark`：备注
- `ExtInfo`：扩展信息

**验证规则**：
- 验证设备是否存在
- 验证设备类型是否为摄像机
- 验证监测对象是否存在（如果提供）

## 数据流

```
前端请求 → CameraService.GetTreeAsync()
  ↓
查询 NVR 录像机信息（通过网关关联）
  ↓
查询摄像机设备（type=1）
  ↓
查询监测对象（过滤分组类型）
  ↓
查询预置点信息（type=1）
  ↓
使用 StreamingUrlManager 构建流媒体 URL
  ↓
返回 CameraTreeDto 树形结构
```

## 摄像机设备类型

| 类型 | 说明 |
|-----|------|
| Camera | 摄像机设备 |
| PresetPoint | 预置点 |
| MonitoredObject | 监测对象 |

## 流媒体 URL 构建

每个摄像机设备都会自动构建流媒体 URL：

```csharp
StreamUrl = _streamingUrlManager.BuildStreamingUrl(nvrId, cameraId)
```

生成的 URL 格式通常为：
- 实时视频流：`http://streaming-server/live/{nvrId}/{cameraId}.flv`
- 回放流：`http://streaming-server/playback/{nvrId}/{cameraId}`

## 树形结构数据

CameraTreeDto 包含以下层级：

```
CameraTreeDto
├── Nvrs (录像机列表)
│   ├── Id
│   ├── Name
│   ├── HostAddress
│   └── GatewayId
├── Cameras (摄像机列表)
│   ├── Id
│   ├── Name
│   ├── NvrId
│   ├── StreamUrl (流媒体URL)
│   └── MonitoredObjectId
├── MonitoredObjects (监测对象列表)
│   ├── Id
│   └── Name
└── Presets (预置点列表)
    ├── Id
    ├── Name
    ├── PresetNo (预置位编号)
    └── MonitoredObjectId
```

## 依赖服务

- [[StreamingUrlManager]] - 流媒体URL管理器
- [[MediaService]] - 媒体服务
- [[NvrService]] - 录像机服务
- [[DeviceService]] - 设备服务

## 相关实体

- [[DeviceEntity]] - 设备实体
- [[NvrEntity]] - 录像机实体
- [[SensorEntity]] - 传感器实体（预置点）
- [[MonitoredObjectAggregateRoot]] - 监测对象实体
- [[MonitoredObjectPresetRelEntity]] - 预置点关联实体

## 权限控制

```csharp
[Authorize]  // 需要认证
public class CameraService : ApplicationService, ICameraService
```

所有接口都需要身份认证。

## 异常处理

| 异常类型 | 场景 |
|---------|------|
| UserFriendlyException "404" | 摄像头设备不存在 |
| UserFriendlyException "400" | 设备类型不是摄像机 |
| UserFriendlyException "500" | 系统内部错误 |

## API 路径

- `GET /api/app/camera/tree` - 获取巡视设备树
- `PUT /api/app/camera/{id}` - 更新摄像机设备

## 使用场景

1. **巡视任务配置**：获取可用的摄像机和预置点
2. **实时监控**：获取摄像机流媒体URL
3. **设备管理**：更新摄像机配置信息
4. **预置位管理**：获取摄像机下的预置点列表

## 注意事项

1. **性能优化**：树形结构查询使用了 Join 查询减少数据库访问
2. **流媒体URL**：每个摄像机都会自动生成流媒体URL用于视频播放
3. **关联查询**：预置点会自动关联监测对象信息
4. **过滤功能**：支持按网关和启用状态过滤数据
5. **日志记录**：所有操作都有详细的日志记录便于排查问题

## 配置项

该服务依赖以下配置：

```json
{
  "Streaming": {
    "BaseUrl": "http://streaming-server",
    "DefaultFormat": "flv"
  }
}
```
