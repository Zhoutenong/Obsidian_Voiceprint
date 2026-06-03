# 媒体服务 (MediaService)

## 概述

媒体服务是视频监控系统的核心组件，提供视频流媒体、录像管理、云台控制、预置位管理和图像抓拍等完整功能。

**位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/MediaService.cs`  
**层**：Application  
**模块**：ast-intellisub  
**依赖注入**：Scoped

## 职责

- 视频流媒体管理（实时播放、录像回放）
- 云台控制（PTZ、光圈、焦点）
- 预置位管理（添加、更新、删除、调用）
- 图像抓拍与存储
- 录像查询与下载
- 摄像头资源占用管理
- 支持多厂商NVR设备

## 主要接口

### 实时视频播放

```csharp
Task<ApiStreamContent> StartPlaybackAsync(DeviceAndChannelInputDto input)
Task StopPlaybackAsync(DeviceAndChannelInputDto input)
```

**功能**：启动/停止实时视频流

**请求参数**：
- `NvrId`：录像机设备ID
- `CameraId`：摄像头通道ID

**返回值**：
```csharp
class ApiStreamContent
{
    string Stream      // 流ID
    string StreamUrl   // 流媒体播放URL
}
```

### 录像回放

```csharp
Task<MediaInfo> QueryRecordingsAsync(QueryRecordingsInputDto input)
Task<Playback> StartRecordingPlaybackAsync(StartRecordingPlaybackInputDto input)
Task StopRecordingPlaybackAsync(StopRecordingPlaybackInputDto input)
```

**功能**：查询历史录像、启动/停止录像回放

**录像查询参数**：
- `NvrId`：录像机设备ID
- `CameraId`：摄像头通道ID
- `Date`：查询日期

**回放参数**：
- `NvrId`：录像机设备ID
- `CameraId`：摄像头通道ID
- `StartTime`：开始时间
- `EndTime`：结束时间

### 云台控制

```csharp
Task UnifiedControlAsync(UnifiedControlInputDto input)
```

**功能**：统一的设备控制接口

**控制指令**：
- **方向控制**：Left, Right, Up, Down, UpLeft, UpRight, DownLeft, DownRight（速度：0-255）
- **变倍控制**：ZoomIn, ZoomOut（速度：0-15）
- **光圈控制**：IrisIn, IrisOut, IrisStop（速度：0-255）
- **焦点控制**：FocusNear, FocusFar, FocusStop（速度：0-255）
- **停止控制**：Stop, PtzStop, IrisStop, FocusStop

**占用检查**：执行前会检查摄像头是否被巡视任务占用

### 预置位管理

```csharp
Task<List<Preset>> GetPresetsAsync(DeviceAndChannelInputDto input)
Task CallPresetAsync(CallPresetInputDto input)
Task CallPresetAndCaptureAsync(CallPresetAndCaptureInputDto input)
Task AddOrUpdatePresetAsync(AddPresetInputDto input)
Task DeletePresetAsync(DeletePresetInputDto input)
```

**功能**：预置位查询、调用、添加、删除

**预置位调用流程**：
1. 发送预置位调用指令
2. 等待摄像头稳定（可配置，默认3秒）
3. 自动截图并保存

### 图像抓拍

```csharp
Task<string?> CaptureAndSaveSnapshotAsync(CaptureAndSaveSnapshotInputDto input)
```

**功能**：抓拍当前画面并保存到服务器

**返回值**：截图文件的Web访问路径（如：`/snapshot/2026-06-03/nvrId/cameraId/timestamp.jpg`）

**支持厂商**：
- **HIKVISION**：使用ISAPI接口截图
- **其他厂商**：使用流媒体网关截图

**存储路径**：
```
store/
└── snapshot/
    └── 2026-06-03/
        └── {nvrId}/
            └── {cameraId}/
                └── {timestamp}.jpg
```

### 录像下载

```csharp
Task<RecordingDownload> StartRecordingDownloadAsync(StartRecordingDownloadInputDto input)
Task StopRecordingDownloadAsync(StopRecordingDownloadInputDto input)
Task<DownloadProgressDto> GetRecordingDownloadProgressAsync(GetRecordingDownloadProgressInputDto input)
```

**功能**：启动/停止历史录像下载，查询下载进度

**下载参数**：
- `TimeRange`：时间范围（格式：`yyyy-MM-dd HH:mm:ss~HH:mm:ss`）
- `DownloadSpeed`：下载速度（倍速，如：1、2、4）

### 摄像头资源管理

```csharp
CameraStatus GetCameraStatus(string cameraId)
Task<TestCameraOccupyResultDto> CreateTestCameraOccupyAsync(TestCameraOccupyInputDto input)
```

**功能**：查询摄像头占用状态、测试摄像头占用

## 视频流协议

### 支持的流媒体协议

| 协议 | 格式 | 用途 |
|-----|------|------|
| HTTP-FLV | .flv | 实时视频流 |
| HLS | .m3u8 | 实时视频流（移动端） |
| RTSP | rtsp:// | 实时视频流（原生） |

### 流媒体URL格式

```
实时视频：http://streaming-server/live/{nvrId}/{cameraId}.flv
录像回放：http://streaming-server/playback/{streamId}
HLS：     http://streaming-server/hls/{nvrId}/{cameraId}/index.m3u8
```

## 云台控制协议 (PTZ)

### PTZ控制指令

| 指令类型 | 命令 | 速度范围 | 说明 |
|---------|------|---------|------|
| 方向控制 | left, right, up, down | 0-255 | 水平垂直移动 |
| 变倍控制 | zoom, zoom-in, zoom-out | 0-15 | 镜头缩放 |
| 光圈控制 | iris-in, iris-out | 0-255 | 光圈大小 |
| 焦点控制 | focus-near, focus-far | 0-255 | 焦距调整 |
| 停止控制 | stop | - | 停止所有动作 |

### 控制流程

```
前端请求 → MediaService.UnifiedControlAsync()
  ↓
检查摄像头占用状态（CameraResourceManager）
  ↓
获取流媒体服务（StreamingGatewayService）
  ↓
发送PTZ控制指令（StreamingApiService）
  ↓
返回执行结果
```

## 录像管理

### 录像查询流程

```
QueryRecordingsAsync()
  ↓
构造查询时间范围（当天00:00:00 - 23:59:59）
  ↓
调用流媒体网关查询录像
  ↓
处理录像文件列表
  ↓
格式化时间显示（同日只显示时间，跨日显示完整日期）
  ↓
返回 MediaInfo 对象
```

### 录像信息结构

```csharp
class MediaInfo
{
    string DeviceId              // 设备ID
    string ChannelId             // 通道ID
    List<MediaItem> RecordList   // 录像文件列表
}

class MediaItem
{
    string ChannelId    // 通道ID
    long FileSize       // 文件大小（字节）
    string StartTime    // 开始时间
    string EndTime      // 结束时间
    string TimeRange    // 时间范围（格式化显示）
}
```

## 图像抓拍流程

### 海康NVR抓拍流程

```
CaptureAndSaveSnapshotAsync()
  ↓
判断NVR厂商为HIKVISION
  ↓
查询摄像头设备IP地址
  ↓
查询NVR通道列表（ISAPI/ContentMgmt/InputProxy/channels）
  ↓
按IP匹配找到对应通道ID
  ↓
生成存储路径（按日期和设备组织）
  ↓
调用ISAPI截图接口保存到本地
  ↓
返回Web访问路径
```

### 其他厂商抓拍流程

```
CaptureAndSaveSnapshotAsync()
  ↓
获取流媒体服务
  ↓
请求设备截图（StreamingApiService.GetSnapshotAsync）
  ↓
检测图像格式（JPEG/PNG/GIF/BMP/WebP）
  ↓
生成唯一文件名
  ↓
原子性写入文件（先写临时文件再重命名）
  ↓
返回Web访问路径
```

## 摄像头资源占用管理

### 占用检查机制

```csharp
private bool CheckCameraAvailability(string cameraId, bool throwOnBusy = true)
{
    var status = _cameraResourceManager.GetCameraStatus(cameraId);
    
    if (status.IsInUse)
    {
        // 摄像头被占用
        if (throwOnBusy)
            throw new UserFriendlyException("摄像头被占用，请稍后再试", "409");
        return false;
    }
    
    return true;
}
```

### 占用状态信息

```csharp
class CameraStatus
{
    bool IsInUse                 // 是否被占用
    string CurrentOperationId    // 当前操作ID
    DateTime? AcquiredAt          // 占用开始时间
}
```

### 使用场景

- **巡视任务期间**：禁止云台和预置位操作
- **手动控制**：执行前检查占用状态
- **测试占用**：提供测试接口模拟占用

## 依赖服务

- [[StreamingUrlManager]] - 流媒体URL管理器
- [[StreamingGatewayService]] - 流媒体网关服务
- [[StreamingApiService]] - 流媒体API服务
- [[CameraResourceManager]] - 摄像头资源管理器
- [[PTZPresetService]] - PTZ预置位服务（海康专用）
- [[NvrService]] - 录像机服务

## 相关实体

- [[NvrEntity]] - 录像机实体
- [[DeviceEntity]] - 设备实体
- [[SensorEntity]] - 传感器实体（预置点）

## 配置项

### 巡视配置

```json
{
  "Patrol": {
    "PresetStabilizeDelayMs": 3000  // 预置位稳定等待时间（毫秒）
  }
}
```

### 流媒体配置

```json
{
  "StreamingApi": {
    "BaseUrl": "http://streaming-server",
    "TimeoutSeconds": 30,
    "SnapshotRootPath": "/store/snapshot"
  }
}
```

## API 路径

| 功能 | 路径 | 方法 |
|-----|------|------|
| 实时播放 | `/api/app/media/play/start` | POST |
| 停止播放 | `/api/app/media/play/stop` | POST |
| 录像查询 | `/api/app/media/recordings/query` | POST |
| 录像回放 | `/api/app/media/playback/start` | POST |
| 停止回放 | `/api/app/media/playback/stop` | POST |
| 云台控制 | `/api/app/media/control` | POST |
| 预置位列表 | `/api/app/media/presets` | POST |
| 调用预置位 | `/api/app/media/preset/call` | POST |
| 预置位截图 | `/api/app/media/preset/capture` | POST |
| 添加预置位 | `/api/app/media/preset/add` | POST |
| 删除预置位 | `/api/app/media/preset/delete` | POST |
| 抓拍截图 | `/api/app/media/snapshot` | POST |
| 摄像头状态 | `/api/app/media/camera/status` | GET |
| 测试占用 | `/api/app/media/camera/test-occupy` | POST |

## 异常处理

| 异常类型 | 场景 |
|---------|------|
| UserFriendlyException "409" | 摄像头被占用 |
| UserFriendlyException "404" | 设备不存在 |
| HttpRequestException | 流媒体服务通信失败 |
| InvalidOperationException | 操作参数无效 |

## 使用场景

1. **实时监控**：启动实时视频流进行监控
2. **录像回放**：查询和播放历史录像
3. **远程控制**：云台方向、变倍、光圈、焦点控制
4. **预置位管理**：调用预置位、添加/删除预置位
5. **图像采集**：抓拍当前画面用于分析
6. **巡视任务**：自动调用预置位并截图
7. **录像下载**：下载历史录像文件

## 注意事项

1. **资源管理**：播放结束后必须调用停止接口释放资源
2. **占用检查**：云台操作前会检查摄像头占用状态
3. **厂商兼容**：支持多种NVR厂商，使用不同的接口
4. **原子性操作**：文件写入使用临时文件+重命名保证原子性
5. **格式检测**：自动检测图像格式（JPEG/PNG/GIF/BMP/WebP）
6. **路径清理**：文件路径组件会清理不安全字符
7. **日志记录**：所有操作都有详细日志便于排查问题
8. **异常容忍**：某些非关键操作失败不会中断流程（如预置位截图失败）

## 安全考虑

1. **认证授权**：所有接口都需要身份认证
2. **操作日志**：云台控制和预置位操作会记录操作日志
3. **路径安全**：文件路径会清理防止路径遍历攻击
4. **资源限制**：测试占用限制在1-300秒防止滥用
