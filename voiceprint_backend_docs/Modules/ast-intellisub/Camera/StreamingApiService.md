# 流媒体 API 服务 (StreamingApiService)

## 概述

流媒体API服务是与流媒体网关（如WVP-PRO）通信的核心组件，提供设备管理、视频播放、录像回放、云台控制和截图等底层API接口。

**位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/StreamingApiService.cs`  
**层**：Application  
**模块**：ast-intellisub  
**依赖注入**：Transient

## 职责

- 与流媒体网关API通信
- 提供设备查询和通道管理
- 实现实时视频播放控制
- 实现录像查询和回放
- 实现云台PTZ控制
- 实现预置位管理
- 实现设备截图功能
- 自动认证和令牌刷新

## 主要接口

### 认证管理

```csharp
Task LoginAsync(string username, string password)
```

**功能**：首次登录并保存凭据用于自动刷新

**认证机制**：
- 使用用户名密码登录获取AccessToken
- AccessToken存储在内存中用于后续请求
- 请求失败时自动重新登录
- 使用SemaphoreSlim保证线程安全

### 设备管理

```csharp
Task<PagedResult<Device>> GetDevicesAsync(int page, int count, string? query = null, bool? status = null)
Task<Device> GetDeviceInfoAsync(string deviceId)
Task SyncDeviceChannelsAsync(string deviceId)
Task<SyncStatus> GetChannelSyncStatusAsync(string deviceId)
Task<PagedResult<Channel>> GetChannelsAsync(string deviceId, int page, int count, string? query = null, bool? online = null)
```

**功能**：查询设备列表、设备详情、同步通道、获取通道列表

**设备查询参数**：
- `page`：页码（从1开始）
- `count`：每页数量
- `query`：搜索关键词（可选）
- `status`：在线状态过滤（可选，true=在线）

**通道查询参数**：
- `deviceId`：设备ID
- `page`：页码
- `count`：每页数量
- `query`：通道名称搜索（可选）
- `online`：在线状态过滤（可选）

### 实时视频播放

```csharp
Task<ApiStreamContent> StartPlaybackAsync(string deviceId, string channelId)
Task StopPlaybackAsync(string deviceId, string channelId)
```

**功能**：启动/停止实时视频流

**返回值**：
```csharp
class ApiStreamContent
{
    string Stream      // 流ID
    string StreamUrl   // 播放URL
}
```

### 录像管理

```csharp
Task<MediaInfo> QueryRecordingsAsync(string deviceId, string channelId, DateTime startTime, DateTime endTime)
Task<ApiStreamContent> StartRecordingPlaybackAsync(string deviceId, string channelId, DateTime startTime, DateTime endTime)
Task StopRecordingPlaybackAsync(string deviceId, string channelId, string streamId)
Task<RecordingDownload> StartRecordingDownloadAsync(string deviceId, string channelId, DateTime startTime, DateTime endTime, string downloadSpeed)
Task StopRecordingDownloadAsync(string deviceId, string channelId, string streamId)
Task<DownloadTracker> GetRecordingDownloadProgressAsync(string deviceId, string channelId, string streamId)
```

**功能**：查询录像、回放录像、下载录像

**录像查询参数**：
- `deviceId`：设备ID
- `channelId`：通道ID
- `startTime`：开始时间
- `endTime`：结束时间

**下载参数**：
- `downloadSpeed`：下载速度（倍速，如：1、2、4）

### 云台控制 (PTZ)

```csharp
Task ControlPtzAsync(string deviceId, string channelId, string command, int horizonSpeed, int verticalSpeed, int zoomSpeed)
Task ControlIrisAsync(string deviceId, string channelId, string command, int speed)
Task ControlFocusAsync(string deviceId, string channelId, string command, int speed)
```

**功能**：云台方向控制、光圈控制、焦点控制

**PTZ控制参数**：
- `command`：控制指令（left、right、up、down、zoom、zoom-in、zoom-out等）
- `horizonSpeed`：水平速度（0-255）
- `verticalSpeed`：垂直速度（0-255）
- `zoomSpeed`：变倍速度（0-15）

**光圈/焦点控制参数**：
- `command`：控制指令（in、out、stop）
- `speed`：速度（0-255）

### 预置位管理

```csharp
Task<List<Preset>> GetPresetsAsync(string deviceId, string channelId)
Task CallPresetAsync(string deviceId, string channelId, int presetId)
Task AddPresetAsync(string deviceId, string channelId, int presetId)
Task DeletePresetAsync(string deviceId, string channelId, int presetId)
```

**功能**：查询预置位、调用预置位、添加预置位、删除预置位

### 设备截图

```csharp
Task<byte[]> RequestSnapshotAsync(string deviceId, string channelId)
Task<string> GetSnapshotAsync(string nvrId, string cameraId)
```

**功能**：请求设备截图（二进制数据）、截图并保存到本地

**GetSnapshotAsync 返回值**：Web可访问的相对路径（如：`/snapshot/2026-06-03/nvrId/cameraId/timestamp.jpg`）

## 流媒体协议

### 支持的流媒体协议

| 协议 | 格式 | 说明 |
|-----|------|------|
| HTTP-FLV | .flv | 通过HTTP传输的FLV流 |
| HLS | .m3u8 | HTTP Live Streaming |
| RTSP | rtsp:// | Real Time Streaming Protocol |
| HTTP-FLV | .flv | 实时视频流 |

### WVP-PRO API 端点

| 功能 | API端点 | 方法 |
|-----|---------|------|
| 用户登录 | `/api/user/login` | GET |
| 查询设备 | `/api/device/query/devices` | GET |
| 设备详情 | `/api/device/query/info` | GET |
| 同步通道 | `/api/device/query/devices/{id}/sync` | GET |
| 查询通道 | `/api/device/query/devices/{id}/channels` | GET |
| 开始播放 | `/api/play/start/{deviceId}/{channelId}` | GET |
| 停止播放 | `/api/play/stop/{deviceId}/{channelId}` | GET |
| 查询录像 | `/api/gb_record/query/{deviceId}/{channelId}` | GET |
| 开始回放 | `/api/playback/start/{deviceId}/{channelId}` | GET |
| 停止回放 | `/api/playback/stop/{deviceId}/{channelId}/{streamId}` | GET |
| PTZ控制 | `/api/front-end/ptz/{deviceId}/{channelId}` | GET |
| 预置位查询 | `/api/front-end/preset/query/{deviceId}/{channelId}` | GET |
| 调用预置位 | `/api/front-end/preset/call/{deviceId}/{channelId}` | GET |
| 添加预置位 | `/api/front-end/preset/add/{deviceId}/{channelId}` | GET |
| 删除预置位 | `/api/front-end/preset/delete/{deviceId}/{channelId}` | GET |
| 设备截图 | `/api/device/query/snap/{deviceId}/{channelId}` | GET |
| 录像下载 | `/api/gb_record/download/start/{deviceId}/{channelId}` | GET |
| 下载进度 | `/api/gb_record/download/progress/{deviceId}/{channelId}/{streamId}` | GET |

## 自动认证与令牌刷新

### 认证流程

```
首次调用 LoginAsync(username, password)
  ↓
发送登录请求获取 AccessToken
  ↓
保存 AccessToken 和用户凭据
  ↓
后续请求使用 access-token 头部
  ↓
如果收到 401 Unauthorized
  ↓
自动重新登录获取新 Token
  ↓
重试原始请求
```

### 线程安全机制

```csharp
private readonly SemaphoreSlim _loginSemaphore = new SemaphoreSlim(1, 1);

private async Task ReLoginAsync(string? failedToken)
{
    await _loginSemaphore.WaitAsync();
    try
    {
        // 双重检查：只有当当前token匹配失败token时才重新登录
        if (_accessToken != failedToken)
            return;
            
        // 执行重新登录
        // ...
    }
    finally
    {
        _loginSemaphore.Release();
    }
}
```

**作用**：
- 确保同一时间只有一个线程在执行登录
- 避免多个线程同时检测到401后重复登录
- 使用双重检查优化性能

## 设备截图处理

### 截图请求流程

```
RequestSnapshotAsync(deviceId, channelId)
  ↓
发送 GET /api/device/query/snap/{deviceId}/{channelId}
  ↓
接收二进制图片数据
  ↓
返回图片字节数组
```

### 截图保存流程

```
GetSnapshotAsync(nvrId, cameraId)
  ↓
请求截图（RequestSnapshotAsync）
  ↓
验证图像数据（非空、大小合理）
  ↓
检测图像格式（JPEG/PNG/GIF/BMP/WebP）
  ↓
生成存储路径（按日期和设备组织）
  ↓
确保目录存在
  ↓
原子性写入文件（临时文件+重命名）
  ↓
返回Web访问路径
```

### 图像格式检测

```csharp
private static string DetectImageExtension(byte[] imageBytes)
{
    // JPEG: FF D8 FF
    if (imageBytes[0] == 0xFF && imageBytes[1] == 0xD8 && imageBytes[2] == 0xFF)
        return ".jpg";
        
    // PNG: 89 50 4E 47
    if (imageBytes[0] == 0x89 && imageBytes[1] == 0x50 && 
        imageBytes[2] == 0x4E && imageBytes[3] == 0x47)
        return ".png";
        
    // GIF: 47 49 46
    if (imageBytes[0] == 0x47 && imageBytes[1] == 0x49 && 
        imageBytes[2] == 0x46)
        return ".gif";
        
    // BMP: 42 4D
    if (imageBytes[0] == 0x42 && imageBytes[1] == 0x4D)
        return ".bmp";
        
    // WebP: RIFF...WEBP
    if (imageBytes.Length >= 12 && 
        imageBytes[0] == 0x52 && imageBytes[1] == 0x49 && 
        imageBytes[2] == 0x46 && imageBytes[3] == 0x46 &&
        imageBytes[8] == 0x57 && imageBytes[9] == 0x45 && 
        imageBytes[10] == 0x42 && imageBytes[11] == 0x50)
        return ".webp";
        
    return ".jpg";  // 默认JPEG
}
```

### 存储路径规则

```
store/
└── snapshot/
    └── {yyyy-MM-dd}/           # 按日期分组
        └── {sanitized(nvrId)}/  # NVR设备ID（清理特殊字符）
            └── {sanitized(cameraId)}/  # 摄像头ID
                └── {timestamp}.{ext}    # 时间戳+格式扩展名
```

**路径清理**：
- 移除路径非法字符
- 替换 `..` 为 `_`
- 限制组件长度不超过50字符
- 确保路径组件不为空

## NVR 连接管理

### 从网关配置创建服务

```csharp
public static StreamingApiService CreateFromGateway(
    HttpClient httpClient,
    ILogger<StreamingApiService> logger,
    string baseAddress,
    string username,
    string password,
    int timeoutSeconds = 30)
{
    var options = new StreamingApiOptions
    {
        BaseAddress = baseAddress,
        DefaultUsername = username,
        DefaultPassword = password,
        TimeoutSeconds = timeoutSeconds,
        AutoLogin = true
    };

    var service = new StreamingApiService(httpClient, logger, Options.Create(options));
    service._username = username;
    service._password = password;
    
    return service;
}
```

**使用场景**：
- 为每个NVR设备创建独立的StreamingApiService实例
- 使用NVR的连接信息和认证凭据
- 支持多个流媒体网关

## 配置项

```json
{
  "StreamingApi": {
    "BaseUrl": "http://streaming-server",
    "DefaultUsername": "admin",
    "DefaultPassword": "***",
    "TimeoutSeconds": 30,
    "AutoLogin": true,
    "SnapshotRootPath": "/store/snapshot"
  }
}
```

## 错误处理

### HTTP 状态码处理

| 状态码 | 处理方式 |
|-------|---------|
| 401 Unauthorized | 自动重新登录并重试请求 |
| 其他错误 | 抛出 HttpRequestException |

### API 业务错误

```csharp
class WVPResult<T>
{
    int Code       // 0表示成功，非0表示业务错误
    string Msg     // 错误消息
    T? Data       // 返回数据
}
```

**错误处理**：
- Code != 0 时抛出异常
- Data 为空时抛出异常
- 记录详细错误日志

## 依赖服务

- `HttpClient` - HTTP客户端
- `ILogger<StreamingApiService>` - 日志记录器
- `IOptions<StreamingApiOptions>` - 配置选项

## 相关实体

```csharp
class Device
{
    string DeviceId
    string Name
    bool OnLine
    // ... 其他设备属性
}

class Channel
{
    string ChannelId
    string Name
    bool OnLine
    // ... 其他通道属性
}

class Preset
{
    string PresetId
    string PresetName
}

class ApiStreamContent
{
    string Stream
    string StreamUrl
}

class MediaInfo
{
    string DeviceId
    string ChannelId
    List<MediaItem> RecordList
}

class Playback
{
    string Stream
    string StreamUrl
}
```

## 使用场景

1. **实时监控**：启动实时视频流
2. **录像回放**：查询和播放历史录像
3. **远程控制**：云台PTZ控制
4. **预置位管理**：查询、调用、添加、删除预置位
5. **图像采集**：抓拍当前画面
6. **录像下载**：下载历史录像文件
7. **设备管理**：查询设备状态和通道信息

## 注意事项

1. **自动认证**：首次使用需要调用LoginAsync，之后会自动处理令牌刷新
2. **线程安全**：重新登录机制使用SemaphoreSlim保证线程安全
3. **失败重试**：401错误会自动重新登录并重试一次
4. **资源释放**：视频流和录像回放使用后需要调用停止接口
5. **路径安全**：文件路径会清理非法字符防止路径遍历攻击
6. **原子操作**：文件写入使用临时文件+重命名保证原子性
7. **格式检测**：自动检测图像格式选择正确的文件扩展名
8. **日志记录**：所有操作都有详细日志便于排查问题

## 安全考虑

1. **令牌管理**：AccessToken存储在内存中，重启后需要重新登录
2. **凭据保护**：用户名和密码只在内存中保存，不持久化
3. **HTTPS支持**：生产环境建议使用HTTPS加密通信
4. **路径安全**：文件路径会清理防止路径遍历攻击
5. **超时保护**：请求超时限制为30秒避免长时间阻塞
