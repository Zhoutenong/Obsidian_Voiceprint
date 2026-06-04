# StreamingUrlManager — 流媒体URL管理器

## 基本信息

- **Manager名称**：`StreamingUrlManager`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Managers/`
- **生命周期**：`ITransientDependency` - 瞬态依赖
- **命名空间**：`Ast.IntelliSub.Application.Managers`

## Manager概述

StreamingUrlManager 负责构建流媒体相关的URL，包括实时流URL和文件下载URL。它支持配置化的流媒体服务器地址，并提供回退机制使用当前请求服务器地址。

## 核心职责

1. **流媒体URL构建** - 构建WebSocket实时流URL
2. **下载URL构建** - 构建文件下载URL
3. **配置优先** - 优先使用配置的流媒体服务器地址
4. **回退机制** - 配置不可用时回退到请求服务器地址

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `IHttpContextAccessor` | HTTP上下文访问器 |
| `IOptionsMonitor<StreamingServerOptions>` | 流媒体服务器配置 |
| `ILogger<StreamingUrlManager>` | 日志记录器 |

## 核心方法

### 1. BuildStreamingUrl(string streamName) - 构建流媒体URL

**签名**：
```csharp
public string? BuildStreamingUrl(string streamName)
```

**功能**：
根据流名称构建WebSocket实时流URL。

**执行流程**：

```
1. 参数验证
   │
   ├─ 如果 streamName 为空
   │  └─ return null
   │
2. 获取服务器配置
   │
   ├─ 优先使用 StreamingServerOptions
   │  ├─ 如果配置有效（IsValid()）
   │  │  ├─ host = streamingOptions.Host
   │  │  ├─ port = streamingOptions.Port
   │  │  └─ useHttps = streamingOptions.UseHttps
   │  │
   │  └─ 否则使用当前请求服务器地址
   │     ├─ 获取当前HTTP请求
   │     ├─ 如果请求为空 → 记录警告，return null
   │     ├─ scheme = request.Scheme
   │     ├─ host = request.Host.Host
   │     ├─ port = request.Host.Port（默认443/80）
   │     └─ useHttps = (scheme == "https")
   │
3. 构建WebSocket URL
   │
   ├─ wsScheme = useHttps ? "wss" : "ws"
   ├─ streamUrl = "{wsScheme}://{host}:{port}/rtp/{streamName}.live.flv?originTypeStr=rtp_push"
   │
4. 返回streamUrl
```

**URL格式**：
```
ws://{host}:{port}/rtp/{streamName}.live.flv?originTypeStr=rtp_push
wss://{host}:{port}/rtp/{streamName}.live.flv?originTypeStr=rtp_push
```

**示例**：
```csharp
var url = BuildStreamingUrl("camera001_stream");
// 返回: "ws://192.168.1.100:1980/rtp/camera001_stream.live.flv?originTypeStr=rtp_push"
```

### 2. BuildDownloadUrl(string filePath) - 构建下载URL

**签名**：
```csharp
public string? BuildDownloadUrl(string filePath)
```

**功能**：
根据文件路径构建文件下载URL。

**执行流程**：

```
1. 参数验证
   │
   ├─ 如果 filePath 为空
   │  └─ return null
   │
2. 获取服务器配置（同BuildStreamingUrl）
   │
3. 构建HTTP URL
   │
   ├─ httpScheme = useHttps ? "https" : "http"
   ├─ downloadUrl = "{httpScheme}://{host}:{port}/index/api/downloadFile?file_path={escapedFilePath}"
   │
4. 返回downloadUrl
```

**URL格式**：
```
http://{host}:{port}/index/api/downloadFile?file_path={escapedFilePath}
https://{host}:{port}/index/api/downloadFile?file_path={escapedFilePath}
```

**示例**：
```csharp
var url = BuildDownloadUrl("/videos/recordings/camera001_20260604.mp4");
// 返回: "http://192.168.1.100:1980/index/api/downloadFile?file_path=%2Fvideos%2Frecordings%2Fcamera001_20260604.mp4"
```

### 3. BuildStreamingUrl(nvrId, deviceId) - 构建流媒体URL（使用设备ID）

**签名**：
```csharp
public string? BuildStreamingUrl(string nvrId, string deviceId)
```

**功能**：
根据NVR ID和设备ID构建流媒体URL。

**实现逻辑**：
```csharp
public string? BuildStreamingUrl(string nvrId, string deviceId)
{
    if (string.IsNullOrEmpty(nvrId) || string.IsNullOrEmpty(deviceId))
    {
        return null;
    }

    // 组合流名称：{nvrId}_{deviceId}
    var streamName = $"{nvrId}_{deviceId}";
    return BuildStreamingUrl(streamName);
}
```

**流名称格式**：
```
{nvrId}_{deviceId}
```

**示例**：
```csharp
var url = BuildStreamingUrl("nvr001", "camera003");
// 内部调用: BuildStreamingUrl("nvr001_camera003")
// 返回: "ws://192.168.1.100:1980/rtp/nvr001_camera003.live.flv?originTypeStr=rtp_push"
```

## 配置选项

### StreamingServerOptions

```csharp
public class StreamingServerOptions
{
    public string? Host { get; set; }        // 流媒体服务器地址
    public int? Port { get; set; }           // 流媒体服务器端口
    public bool UseHttps { get; set; }      // 是否使用HTTPS
    
    public bool IsValid()
    {
        return !string.IsNullOrEmpty(Host) && Port.HasValue && Port.Value > 0;
    }
}
```

### appsettings.json 配置

```json
{
  "StreamingServer": {
    "Host": "192.168.1.100",
    "Port": 1980,
    "UseHttps": false
  }
}
```

## 日志记录

### 调试日志 (LogDebug)

```csharp
// 使用配置的流媒体服务器地址
"使用配置的流媒体服务器地址: {Host}:{Port}, UseHttps={UseHttps}"

// 使用当前请求服务器地址
"使用当前请求服务器地址: {Host}:{Port}, UseHttps={UseHttps}"

// 构建的URL
"构建流媒体URL: {StreamUrl}"
"构建下载URL: {DownloadUrl}"
```

### 警告日志 (LogWarning)

```csharp
// 无法获取HTTP上下文
"无法获取当前HTTP请求上下文，且未配置流媒体服务器地址"

// 构建失败
"构建流媒体URL失败: StreamName={StreamName}"
"构建下载URL失败: filePath={FilePath}"
```

## 错误处理

### 异常捕获

```csharp
try
{
    // 构建URL逻辑
    return streamUrl;
}
catch (Exception ex)
{
    _logger.LogWarning(ex, "构建流媒体URL失败: StreamName={StreamName}", streamName);
    return null;
}
```

**设计决策**：
- 捕获所有异常，不抛出
- 返回null表示构建失败
- 记录警告日志便于排查

### 容错机制

1. **参数为空** - 返回null
2. **HTTP上下文缺失** - 返回null
3. **配置无效** - 回退到请求服务器地址
4. **构建异常** - 记录日志并返回null

## URL构建规则

### 协议选择

| 配置 | WebSocket流 | HTTP下载 |
|------|-------------|----------|
| `UseHttps = true` | `wss://` | `https://` |
| `UseHttps = false` | `ws://` | `http://` |

### 端口默认值

当从HTTP请求推断端口时：
- HTTPS → 默认端口 443
- HTTP → 默认端口 80

### URL路径格式

**实时流**：
```
/rtp/{streamName}.live.flv?originTypeStr=rtp_push
```

**文件下载**：
```
/index/api/downloadFile?file_path={escapedFilePath}
```

## 使用场景

### 1. 实时视频播放

```csharp
var streamUrl = _streamingUrlManager.BuildStreamingUrl(nvrId, deviceId);

// 返回给前端用于视频播放
// 前端使用 flv.js 或 hls.js 播放实时流
```

### 2. 历史视频下载

```csharp
var downloadUrl = _streamingUrlManager.BuildDownloadUrl(filePath);

// 返回给前端用于视频下载
// 前端使用此URL发起下载请求
```

### 3. 多路视频切换

```csharp
var urls = new List<string>();
foreach (var camera in cameras)
{
    var url = _streamingUrlManager.BuildStreamingUrl(camera.NvrId, camera.DeviceId);
    if (url != null)
    {
        urls.Add(url);
    }
}

// 前端可快速切换不同摄像头的实时流
```

## 设计特点

1. **配置优先** - 优先使用配置的流媒体服务器地址
2. **智能回退** - 配置不可用时自动使用请求服务器地址
3. **协议自适应** - 根据UseHttps配置选择正确的协议
4. **容错设计** - 所有错误情况都返回null，不抛出异常
5. **URL编码** - 文件路径自动进行URI编码
6. **单例复用** - 使用IOptionsMonitor支持配置热更新

## 性能考虑

1. **瞬态依赖** - 每次请求创建新实例，避免状态共享
2. **IOptionsMonitor** - 支持配置热更新，无需重启应用
3. **字符串构建** - 使用字符串拼接而非复杂模板引擎

## 注意事项

1. **配置检查** - 使用前检查StreamingServerOptions是否配置完整
2. **网络可达性** - 确保构建的URL在网络中可访问
3. **流名称格式** - 确保流名称与流媒体服务器预期格式一致
4. **文件路径** - 下载URL的文件路径需要是服务器可访问的绝对路径
5. **HTTPS证书** - 使用HTTPS时确保证书配置正确

## 相关服务

- `CameraService` - 摄像机服务
- `MediaService` - 媒体服务
- `StreamingApiService` - 流媒体API服务
- `StreamingTransferService` - 流媒体转发服务

## 相关实体

- `NvrEntity` - NVR实体
- `DeviceEntity` - 设备实体

## 配置示例

### 完整配置

```json
{
  "StreamingServer": {
    "Host": "192.168.1.100",
    "Port": 1980,
    "UseHttps": false
  }
}
```

### 使用当前服务器（不配置）

```json
{
  "StreamingServer": {
    "Host": "",
    "Port": null,
    "UseHttps": false
  }
}
```

如果不配置或配置无效，系统会自动使用当前请求的服务器地址。

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Managers/StreamingUrlManager.cs`
