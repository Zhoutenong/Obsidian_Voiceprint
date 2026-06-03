# 流媒体传输服务 (StreamingTransferService)

## 概述

流媒体传输服务提供基于HTTP Chunked Transfer-Encoding的数据流传输能力，支持大容量数据（如综合运行报告、设备报告）的分块传输和实时转发，具备自动重试、GZIP压缩、服务器支持检测等企业级特性。

**位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/StreamingTransferService.cs`  
**层**：Application  
**模块**：ast-intellisub  
**依赖注入**：Transient（瞬时，每次请求创建新实例）

## 职责

- 提供HTTP Chunked Transfer-Encoding数据传输
- 支持综合运行报告和设备报告的流式发送
- 自动分块和GZIP压缩
- 传输失败自动重试机制
- 目标服务器支持检测
- 可配置的传输参数（分块大小、超时、重试次数等）

## 主要接口

### 发送综合运行报告流

```csharp
Task<ReportResponse> SendComprehensiveReportStreamAsync(
    ComprehensiveOperationReportDto reportData,
    string targetUrl,
    int chunkSize = 65536,
    CancellationToken cancellationToken = default)
```

**功能说明**：使用HTTP Chunked Transfer-Encoding发送综合运行报告数据，自动分块传输并支持压缩

**请求参数**：
- `reportData`：综合运行报告数据（必填，包含设备、监测对象、告警等信息）
- `targetUrl`：目标API地址（必填，接收数据的URL）
- `chunkSize`：分块大小（字节，可选，默认65536=64KB）
- `cancellationToken`：取消令牌（可选，用于取消操作）

**返回数据**：
```csharp
class ReportResponse
{
    bool success                  // 是否成功
    int code                      // 状态码（200=成功）
    string message                // 响应消息
    ResponseReportData data       // 响应数据
}

class ResponseReportData
{
    string reportId              // 报告ID
    DateTime timestamp           // 时间戳
}
```

**业务流程**：
```
1. 记录开始传输日志
   ↓
2. 验证并调整分块大小（ValidateChunkSize）
   ↓
3. 检查服务器支持（可选，CheckChunkedTransferSupportAsync）
   ↓
4. 序列化报告数据为JSON
   ↓
5. 发送JSON数据流（SendJsonStreamAsync）
   ↓
6. 解析响应并返回ReportResponse
```

**异常处理**：
- 传输失败时返回`success=false`的ReportResponse
- 异常信息记录在日志中
- 不抛出异常，由调用者检查返回值

**调用链**：
```
ReportService → SendComprehensiveReportStreamAsync
                    ↓
             ValidateChunkSize（验证分块大小）
                    ↓
             CheckChunkedTransferSupportAsync（检查服务器支持）
                    ↓
             JsonSerializer.Serialize（序列化数据）
                    ↓
             SendJsonStreamAsync（发送JSON流）
                    ↓
             JsonSerializer.Deserialize（解析响应）
```

---

### 发送设备运行报告流

```csharp
Task<ReportResponse> SendDeviceReportStreamAsync(
    DeviceReportDataDto reportData,
    string targetUrl,
    int chunkSize = 65536,
    CancellationToken cancellationToken = default)
```

**功能说明**：使用HTTP Chunked Transfer-Encoding发送设备运行报告数据

**请求参数**：
- `reportData`：设备运行报告数据（必填）
- `targetUrl`：目标API地址（必填）
- `chunkSize`：分块大小（字节，可选，默认65536）
- `cancellationToken`：取消令牌（可选）

**业务流程**：与`SendComprehensiveReportStreamAsync`类似

---

### 发送通用JSON数据流

```csharp
Task<string> SendJsonStreamAsync(
    string jsonData,
    string targetUrl,
    int chunkSize = 65536,
    string contentType = "application/json",
    CancellationToken cancellationToken = default)
```

**功能说明**：发送任意JSON数据流，使用Chunked Transfer-Encoding

**请求参数**：
- `jsonData`：JSON字符串（必填）
- `targetUrl`：目标URL（必填）
- `chunkSize`：分块大小（可选，默认65536字节）
- `contentType`：内容类型（可选，默认"application/json"）
- `cancellationToken`：取消令牌（可选）

**业务流程**：
```
1. 验证分块大小
   ↓
2. 将JSON字符串转换为UTF-8字节数组
   ↓
3. 创建MemoryStream
   ↓
4. 调用SendStreamAsync发送流数据
```

---

### 发送通用数据流

```csharp
Task<string> SendStreamAsync(
    Stream dataStream,
    string targetUrl,
    string contentType = "application/octet-stream",
    CancellationToken cancellationToken = default)
```

**功能说明**：发送任意数据流，支持自动重试机制

**请求参数**：
- `dataStream`：数据流（必填，必须可读取）
- `targetUrl`：目标URL（必填）
- `contentType`：内容类型（可选，默认"application/octet-stream"）
- `cancellationToken`：取消令牌（可选）

**业务流程**（支持重试）：
```
1. 创建HttpClient实例
   ↓
2. 创建ChunkedStreamContent（包装数据流）
   ↓
3. 发送POST请求
   ↓
4. 检查响应状态
   ↓（成功时）
5. 返回响应内容
   ↓（失败时）
6. 重试（最多MaxRetryCount次）
   ↓（重试时）
7. 延迟RetryIntervalMs毫秒
   ↓
8. 重置流位置（如果可定位）
   ↓（所有重试失败时）
9. 抛出最后一个异常
```

**重试机制**：
- 最大重试次数：`_options.MaxRetryCount`（默认3次）
- 重试间隔：`_options.RetryIntervalMs`（默认1000毫秒）
- 每次重试前重置流位置（如果`dataStream.CanSeek == true`）
- 最后一次失败后抛出异常

**调用链**：
```
SendJsonStreamAsync → SendStreamAsync
                        ↓
                 CreateHttpClient（创建HTTP客户端）
                        ↓
                 new ChunkedStreamContent（创建分块内容）
                        ↓
                 HttpClient.PostAsync（发送请求）
                        ↓（失败时）
                 Task.Delay（延迟）
                 dataStream.Position = 0（重置流）
                        ↓
                 重试
```

---

### 检查服务器分块传输支持

```csharp
Task<bool> CheckChunkedTransferSupportAsync(
    string targetUrl,
    CancellationToken cancellationToken = default)
```

**功能说明**：检查目标服务器是否支持Chunked Transfer-Encoding

**请求参数**：
- `targetUrl`：目标URL（必填）
- `cancellationToken`：取消令牌（可选）

**检测方法**：
1. 发送OPTIONS请求到目标URL
2. 检查响应头中的`Transfer-Encoding`或`Transfer-EncodingChunked`
3. 返回是否支持分块传输

**业务规则**：
- 检测失败时默认返回`true`（假设支持）
- 不影响后续传输，仅用于日志记录

**调用链**：
```
SendComprehensiveReportStreamAsync → CheckChunkedTransferSupportAsync
                                        ↓
                                 CreateHttpClient
                                        ↓
                                 HttpRequestMessage(HttpMethod.Options)
                                        ↓
                                 HttpClient.SendAsync
                                        ↓
                                 检查响应头
```

---

### 验证分块大小

```csharp
private int ValidateChunkSize(int chunkSize)
```

**功能说明**：验证并调整分块大小，确保在合理范围内

**验证规则**：
```csharp
if (chunkSize < MinChunkSize)    // 小于4096字节
    return MinChunkSize;         // 调整为最小值

if (chunkSize > MaxChunkSize)    // 大于1048576字节（1MB）
    return MaxChunkSize;         // 调整为最大值

return chunkSize;                // 在范围内，原样返回
```

**配置范围**：
- 最小分块：4096字节（4KB）
- 最大分块：1048576字节（1MB）
- 默认分块：65536字节（64KB）

---

### 创建HTTP客户端

```csharp
private HttpClient CreateHttpClient()
```

**功能说明**：创建配置好的HttpClient实例

**配置项**：
```csharp
httpClient.Timeout = TimeSpan.FromSeconds(ReadWriteTimeoutSeconds);  // 超时时间
httpClient.DefaultRequestHeaders.Add("User-Agent", "IntelliSubstation-StreamingClient/1.0");
httpClient.DefaultRequestHeaders.Add("Accept", "application/json");
httpClient.DefaultRequestHeaders.TransferEncodingChunked = true;    // 启用分块传输
```

**超时配置**：
- 读写超时：`_options.ReadWriteTimeoutSeconds`（默认300秒=5分钟）

---

## 依赖服务

### 基础设施
- `[[IHttpClientFactory]]` - HTTP客户端工厂
- `[[ILogger<StreamingTransferService>]]` - 日志记录
- `[[IConfiguration]]` - 配置读取

### 自定义组件
- `[[ChunkedStreamContent]]` - 分块流内容（自定义HttpContent）
- `[[StreamingTransferOptions]]` - 传输选项配置
- `[[JsonSerializerOptions]]` - JSON序列化选项

### DTO类型
- `[[ComprehensiveOperationReportDto]]` - 综合运行报告DTO
- `[[DeviceReportDataDto]]` - 设备报告DTO
- `[[ReportResponse]]` - 报告响应DTO

---

## 配置选项

### StreamingTransferOptions

**配置节**：`appsettings.json` → `StreamingTransfer`

```json
{
  "StreamingTransfer": {
    "DefaultChunkSize": 65536,              // 默认分块大小（64KB）
    "MaxChunkSize": 1048576,                 // 最大分块大小（1MB）
    "MinChunkSize": 4096,                    // 最小分块大小（4KB）
    "ConnectionTimeoutSeconds": 30,         // 连接超时（秒）
    "ReadWriteTimeoutSeconds": 300,          // 读写超时（秒，默认5分钟）
    "EnableGzipCompression": true,          // 启用GZIP压缩
    "EnableVerboseLogging": false,           // 启用详细日志
    "MaxRetryCount": 3,                      // 最大重试次数
    "RetryIntervalMs": 1000,                // 重试间隔（毫秒）
    "CheckServerSupport": false              // 传输前检查服务器支持
  }
}
```

**配置说明**：

| 配置项 | 类型 | 默认值 | 说明 |
|-------|------|--------|------|
| `DefaultChunkSize` | int | 65536 | 默认分块大小（64KB），平衡性能和内存占用 |
| `MaxChunkSize` | int | 1048576 | 最大分块大小（1MB），防止内存溢出 |
| `MinChunkSize` | int | 4096 | 最小分块大小（4KB），避免过多小分块 |
| `ConnectionTimeoutSeconds` | int | 30 | 连接超时时间（秒） |
| `ReadWriteTimeoutSeconds` | int | 300 | 读写超时时间（秒），用于大文件传输 |
| `EnableGzipCompression` | bool | true | 是否启用GZIP压缩，减少带宽占用 |
| `EnableVerboseLogging` | bool | false | 是否启用详细日志，调试时开启 |
| `MaxRetryCount` | int | 3 | 最大重试次数，应对网络波动 |
| `RetryIntervalMs` | int | 1000 | 重试间隔（毫秒），避免立即重试 |
| `CheckServerSupport` | bool | false | 是否检查服务器支持，可选功能 |

---

## ChunkedStreamContent 工作原理

### 功能说明
自定义的`HttpContent`实现，支持HTTP Chunked Transfer-Encoding和GZIP压缩

**位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/Http/ChunkedStreamContent.cs`

### 核心特性
- 自动分块读取源流
- 可选GZIP压缩
- 自动设置HTTP头：
  - `Transfer-Encoding: chunked`
  - `Content-Encoding: gzip`（启用压缩时）
  - `Content-Type: application/json`

### 工厂方法

```csharp
// 从JSON字符串创建
ChunkedStreamContent.FromJson(string jsonContent, int chunkSize, bool enableCompression)

// 从对象创建（自动序列化）
ChunkedStreamContent.FromObject(object data, int chunkSize, bool enableCompression)

// 直接构造
new ChunkedStreamContent(Stream sourceStream, int chunkSize, bool enableCompression, bool disposeSourceStream)
```

### 分块传输流程
```
源流数据
    ↓
分块读取（chunkSize字节）
    ↓
GZIP压缩（可选）
    ↓
写入HTTP流
    ↓
重复直到结束
```

---

## 数据传输流程

### 完整流程图

```
报告数据（DTO）
    ↓
序列化为JSON
    ↓
转换为UTF-8字节数组
    ↓
创建MemoryStream
    ↓
包装为ChunkedStreamContent
    ↓
分块读取源流（每chunkSize字节）
    ↓（如果启用压缩）
GZIP压缩分块数据
    ↓
写入HTTP请求流（Transfer-Encoding: chunked）
    ↓
发送到目标服务器
    ↓
接收响应
    ↓
反序列化为ReportResponse
    ↓
返回结果
```

### Chunked Transfer-Encoding 优势

1. **无需预知内容大小**：不需要`Content-Length`头
2. **实时传输**：数据生成即可发送，减少延迟
3. **内存友好**：不需要将整个数据加载到内存
4. **大文件支持**：适合传输GB级数据
5. **断点续传**：理论上支持（客户端实现）

---

## 错误处理

### 异常类型

| 异常场景 | 处理方式 | 返回值 |
|---------|---------|--------|
| 序列化失败 | 捕获异常 | 返回`success=false` |
| 传输失败（重试内） | 自动重试 | 成功后返回正常响应 |
| 传输失败（超重试） | 捕获异常 | 返回`success=false` |
| 服务器返回错误 | 抛出异常 | 触发重试机制 |
| 取消操作 | 抛出OperationCanceledException | 上层处理 |

### 重试策略

```
第1次传输失败
    ↓
延迟 RetryIntervalMs 毫秒
    ↓
重置流位置（如果可定位）
    ↓
第2次传输
    ↓（失败）
延迟 RetryIntervalMs 毫秒
    ↓
...
    ↓
第 MaxRetryCount 次失败
    ↓
抛出最后一个异常
```

**重试条件**：
- 仅在`SendStreamAsync`中重试
- 其他方法（如`SendComprehensiveReportStreamAsync`）不重试，直接返回失败响应

---

## 日志记录

### 信息日志

```csharp
// 传输开始
"开始使用HTTP Chunked Transfer-Encoding发送综合运行报告到: {targetUrl}"

// 数据大小
"报表数据大小: {bytes} bytes"

// 传输成功
"综合运行报告流式传输完成"
"设备运行报告流式传输完成"

// 服务器支持检查
"服务器分块传输支持检查结果: {isSupported}"
```

### 警告日志

```csharp
// 分块大小调整
"分块大小 {chunkSize} 小于最小值 {MinChunkSize}，已调整"
"分块大小 {chunkSize} 大于最大值 {MaxChunkSize}，已调整"

// 服务器不支持
"目标服务器可能不支持Chunked Transfer-Encoding，继续尝试传输"

// 重试
"流式传输失败，第{retryCount}次重试: {ex.Message}"
```

### 错误日志

```csharp
// 传输失败
"综合运行报告流式传输失败: {ex.Message}"
"设备运行报告流式传输失败: {ex.Message}"

// 达到最大重试次数
"流式传输失败，已达到最大重试次数: {ex.Message}"
```

### 调试日志（需启用EnableVerboseLogging）

```csharp
// 发送分块流数据
"发送分块流数据到: {targetUrl}, 内容类型: {contentType}"

// 传输成功
"流式传输成功，响应状态: {statusCode}"
```

---

## 使用示例

### 发送综合运行报告

```csharp
// 构建报告数据
var reportData = new ComprehensiveOperationReportDto
{
    SubstationId = substationId,
    StartTime = startTime,
    EndTime = endTime,
    Devices = deviceList,
    // ... 其他字段
};

// 发送报告
var response = await _streamingTransferService.SendComprehensiveReportStreamAsync(
    reportData: reportData,
    targetUrl: "https://remote-server.com/api/reports/comprehensive",
    chunkSize: 131072  // 128KB分块
);

if (response.success)
{
    _logger.LogInformation($"报告发送成功，报告ID: {response.data?.reportId}");
}
else
{
    _logger.LogError($"报告发送失败: {response.message}");
}
```

### 发送设备报告

```csharp
var deviceReport = new DeviceReportDataDto
{
    DeviceId = deviceId,
    ReportData = data
};

var response = await _streamingTransferService.SendDeviceReportStreamAsync(
    deviceReport,
    "https://remote-server.com/api/reports/device"
);
```

### 自定义数据流传输

```csharp
// 创建自定义数据流
using var stream = new MemoryStream();
// 写入数据到stream...

var responseContent = await _streamingTransferService.SendStreamAsync(
    dataStream: stream,
    targetUrl: "https://remote-server.com/api/upload",
    contentType: "application/octet-stream"
);
```

---

## 性能优化建议

### 分块大小选择

| 数据大小 | 推荐分块大小 | 说明 |
|---------|-------------|------|
| < 1MB | 4096 - 16384 | 小分块，低延迟 |
| 1MB - 100MB | 65536（默认） | 平衡性能和内存 |
| 100MB - 1GB | 131072 - 262144 | 较大分块，提高吞吐 |
| > 1GB | 524288 - 1048576 | 最大分块，减少请求次数 |

### 压缩策略

- **文本数据（JSON/XML）**：启用GZIP压缩，可减少70-80%带宽
- **已压缩数据**：禁用GZIP，避免二次压缩浪费CPU
- **二进制数据**：根据数据特性决定，通常不需要压缩

### 超时设置

- **局域网**：`ReadWriteTimeoutSeconds = 60`
- **互联网**：`ReadWriteTimeoutSeconds = 300`（默认）
- **不稳定网络**：`ReadWriteTimeoutSeconds = 600` + 增加重试次数

---

## 相关文档

- [[ChunkedStreamContent]] - 分块流内容实现
- [[ComprehensiveOperationReportDto]] - 综合运行报告DTO
- [[ReportResponse]] - 报告响应DTO
- [[StreamingGatewayService]] - 流媒体网关服务
- [[NvrService]] - NVR设备管理

---

## HTTP协议细节

### Chunked Transfer-Encoding 格式

```
POST /api/reports HTTP/1.1
Host: remote-server.com
Transfer-Encoding: chunked
Content-Type: application/json
Content-Encoding: gzip

[chunk-size in hex]\r\n
[chunk-data]\r\n
[chunk-size in hex]\r\n
[chunk-data]\r\n
...
0\r\n
\r\n
```

**示例**：
```
4000\r\n
[16KB data]\r\n
4000\r\n
[16KB data]\r\n
0\r\n
\r\n
```

### 优势对比

| 特性 | Content-Length | Chunked Transfer-Encoding |
|------|---------------|---------------------------|
| 需预知大小 | ✅ 是 | ❌ 否 |
| 实时传输 | ❌ 否 | ✅ 是 |
| 内存占用 | 高（需缓存全部数据） | 低（分块处理） |
| 大文件支持 | 受限 | 良好 |

---

## 注意事项

⚠️ **重要**：
1. 流数据一旦开始传输，无法回退或修改
2. 源流需要支持定位（`CanSeek = true`）才能重试
3. 分块大小影响性能，需根据实际场景调整
4. GZIP压缩消耗CPU，在服务器性能不足时考虑禁用
5. 超时时间需根据数据大小和网络状况合理设置
6. 取消令牌用于长时间传输的中断控制
7. 默认不检查服务器支持，可在配置中启用

✅ **最佳实践**：
1. 大数据传输使用较大分块（128KB-512KB）
2. 不稳定网络增加重试次数和超时时间
3. 启用详细日志进行问题诊断
4. 监控传输成功率，调整重试策略
5. 考虑断点续传机制（需客户端配合）

---

**最后更新**：2026-06-03  
**版本**：v1.0.0
