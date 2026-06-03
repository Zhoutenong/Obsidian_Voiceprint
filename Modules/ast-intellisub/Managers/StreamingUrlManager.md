# StreamingUrlManager

**路径**: `module/ast-intellisub/Ast.IntelliSub.Application/Managers/StreamingUrlManager.cs`

**依赖**: `ITransientDependency`

## 概述

流媒体 URL 管理器，负责构建实时视频流和录像文件的访问 URL。支持多种协议（RTSP/FLV/HLS/WS-FLV）和动态端口检测，是视频监控和巡检系统的核心组件。

## 核心功能

### 1. 实时流 URL 构建

```csharp
public string? BuildStreamingUrl(string streamName)
```

**职责**：
- 根据流名称构建实时流 URL
- 优先使用配置的流媒体服务器地址
- 回退到当前 HTTP 请求的服务器地址
- 支持 HTTPS 和 HTTP 动态选择

**返回示例**：
```
ws://192.168.1.100:8080/flv?stream=camera_01
http://192.168.1.100:8080/flv?stream=camera_01.flv
```

### 2. 录像下载 URL 构建

```csharp
public string? BuildDownloadUrl(string filePath)
```

**职责**：
- 构建静态文件下载 URL
- 支持 URL 路径编码
- 可选添加时间戳防缓存

**返回示例**：
```
http://192.168.1.100:8080/files/recordings/2023-10-01/camera_01.mp4
http://192.168.1.100:8080/files/recordings/2023-10-01/camera_01.mp4?t=1696112345
```

### 3. 协议转换

```csharp
public string? BuildStreamingUrl(string streamName, string protocol)
```

**支持的协议**：
- `ws-flv` - WebSocket FLV (低延迟)
- `flv` - HTTP FLV
- `hls` - HTTP Live Streaming (兼容性好)
- `rtsp` - RTSP 原始流

## 配置选项

### StreamingServerOptions

```json
{
  "StreamingServer": {
    "Host": "192.168.1.100",
    "Port": 8080,
    "UseHttps": false,
    "FlvPort": 8080,
    "HlsPort": 8080,
    "WsFlvPort": 8080,
    "RtspPort": 554
  }
}
```

**字段说明**：
- `Host` - 流媒体服务器地址（留空则自动检测）
- `Port` - 默认端口
- `UseHttps` - 是否使用 HTTPS
- 协议特定端口（可选）

## 协议转换矩阵

| 输入协议 | URL 协议 | 默认端口 | 路径格式 |
|----------|----------|----------|----------|
| `ws-flv` | `ws://` | `WsFlvPort` | `/flv?stream={name}` |
| `flv` | `http://` | `FlvPort` | `/flv?stream={name}.flv` |
| `hls` | `http://` | `HlsPort` | `/hls/{name}/index.m3u8` |
| `rtsp` | `rtsp://` | `RtspPort` | `/stream/{name}` |

## 使用示例

### 构建实时流 URL

```csharp
public class PresetService
{
    private readonly StreamingUrlManager _streamingUrlManager;
    
    public async Task<PresetDto> GetPresetStreamingUrl(Guid presetId)
    {
        var preset = await _presetRepository.GetAsync(presetId);
        var streamName = $"camera_{preset.CameraId}_preset_{preset.Index}";
        
        var url = _streamingUrlManager.BuildStreamingUrl(streamName);
        
        return new PresetDto
        {
            Id = preset.Id,
            StreamingUrl = url,
            Protocol = "ws-flv"
        };
    }
}
```

### 构建录像下载 URL

```csharp
public class PatrolRecordService
{
    private readonly StreamingUrlManager _streamingUrlManager;
    
    public async Task<string> GetRecordingDownloadUrl(Guid recordId)
    {
        var record = await _recordRepository.GetAsync(recordId);
        var filePath = $"/recordings/{record.Date}/{record.CameraId}.mp4";
        
        return _streamingUrlManager.BuildDownloadUrl(filePath);
    }
}
```

### 多协议支持

```csharp
public async Task<Dictionary<string, string>> GetStreamingUrls(Guid cameraId)
{
    var streamName = $"camera_{cameraId}";
    var urls = new Dictionary<string, string>();
    
    urls["ws-flv"] = _streamingUrlManager.BuildStreamingUrl(streamName, "ws-flv");
    urls["hls"] = _streamingUrlManager.BuildStreamingUrl(streamName, "hls");
    urls["flv"] = _streamingUrlManager.BuildStreamingUrl(streamName, "flv");
    
    return urls;
}
```

## 动态地址检测

### 回退机制

当未配置流媒体服务器地址时，自动从 HTTP 请求上下文检测：

```csharp
if (streamingOptions.IsValid())
{
    host = streamingOptions.Host!;
    port = streamingOptions.Port!.Value;
}
else
{
    // 回退到当前请求地址
    var request = _httpContextAccessor.HttpContext?.Request;
    host = request.Host.Host;
    port = request.Host.Port ?? (scheme == "https" ? 443 : 80);
    useHttps = request.Scheme == "https";
}
```

### 验证配置有效性

```csharp
public bool IsValid()
{
    return !string.IsNullOrWhiteSpace(Host) && Port.HasValue && Port > 0;
}
```

## 错误处理

### 无效输入

```csharp
if (string.IsNullOrEmpty(streamName))
{
    return null;  // 静默失败
}
```

### 缺少上下文

```csharp
if (request == null)
{
    _logger.LogWarning("无法获取当前HTTP请求上下文，且未配置流媒体服务器地址");
    return null;
}
```

### 协议不支持

```csharp
if (!Enum.TryParse<StreamingProtocol>(protocol, true, out var parsedProtocol))
{
    _logger.LogWarning("不支持的流媒体协议: {Protocol}", protocol);
    return null;
}
```

## 性能特性

### 缓存

无内部缓存，每次调用重新构建（轻量操作）。

### 典型延迟

- 配置模式：< 10μs
- 回退模式：< 50μs

## 安全性

### HTTPS 支持

```csharp
string scheme = useHttps ? "https" : "http";
string wsScheme = useHttps ? "wss" : "ws";
```

### URL 编码

下载 URL 自动处理特殊字符：

```csharp
var encodedPath = System.Web.HttpUtility.UrlEncode(filePath);
```

## 监控日志

### 调试日志

```
使用配置的流媒体服务器地址: 192.168.1.100:8080, UseHttps=False
构建实时流URL: ws://192.168.1.100:8080/flv?stream=camera_01_preset_5
```

### 警告日志

```
无法获取当前HTTP请求上下文，且未配置流媒体服务器地址
```

## 相关文档

- [流媒体配置](../../Configuration/StreamingConfiguration.md)
- [预置点管理](../../Services/PresetService.md)
- [巡检记录](../../Services/PatrolRecordService.md)
