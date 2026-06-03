# 识别 API 服务 (RecognitionApiService)

## 概述
识别 API 服务负责与外部智能识别 API 进行交互，提供图像识别功能，支持通过 URL 或 Base64 格式提交识别请求。

## 职责
- 执行智能识别任务
- 支持多种图像输入格式（URL、Base64）
- 验证识别请求参数
- 管理 API 连接和超时
- 处理识别结果
- 提供服务可用性检查
- 支持重试机制

## 主要接口

### 执行智能识别任务
```csharp
Task<RecognitionResponseDto> RecognizeAsync(RecognitionRequestDto request)
```

**请求参数**：
- `RequestId`: 请求ID（必填）
- `ImageUrl`: 图片URL（与ImageBase64二选一）
- `ImageBase64`: Base64图片数据（与ImageUrl二选一）
- `Tasks`: 识别任务列表（必填，至少一个）

**验证规则**：
- 请求ID不能为空
- ImageUrl和ImageBase64必须提供其中一个
- ImageUrl和ImageBase64不能同时提供
- 至少需要提供一个识别任务

### 通过 URL 执行识别
```csharp
Task<RecognitionResponseDto> RecognizeByUrlAsync(
    string requestId,
    string imageUrl,
    List<RecognitionTaskDto> tasks)
```

### 通过 Base64 执行识别
```csharp
Task<RecognitionResponseDto> RecognizeByBase64Async(
    string requestId,
    string imageBase64,
    List<RecognitionTaskDto> tasks)
```

### 执行单个识别任务（URL）
```csharp
Task<RecognitionResultDto?> RecognizeSingleByUrlAsync(
    string requestId,
    string imageUrl,
    string taskId,
    string taskName,
    string algorithmKey,
    RoiDto roi,
    Dictionary<string, object>? parameters = null)
```

### 执行单个识别任务（Base64）
```csharp
Task<RecognitionResultDto?> RecognizeSingleByBase64Async(
    string requestId,
    string imageBase64,
    string taskId,
    string taskName,
    string algorithmKey,
    RoiDto roi,
    Dictionary<string, object>? parameters = null)
```

### 检查服务可用性
```csharp
Task<bool> IsServiceAvailableAsync()
```

## 数据处理流程

```
请求验证 → 参数校验 → 构建 HTTP 请求 → 发送到 API → 处理响应 → 返回结果
```

### 识别流程详细步骤
1. **参数验证**：验证请求参数的完整性和有效性
2. **任务验证**：验证识别任务配置
3. **ROI验证**：验证感兴趣区域参数
4. **序列化请求**：将请求对象序列化为JSON
5. **发送请求**：通过HttpClient发送到识别API
6. **处理响应**：解析响应并返回识别结果
7. **错误处理**：记录错误并抛出用户友好异常

## 请求验证

### 基础参数验证
```csharp
private void ValidateRequest(RecognitionRequestDto request)
{
    // 请求对象不能为空
    // 请求ID不能为空
    // ImageUrl和ImageBase64必须提供其中一个
    // ImageUrl和ImageBase64不能同时提供
    // 至少需要提供一个识别任务
}
```

### 任务验证
```csharp
private void ValidateTask(RecognitionTaskDto task)
{
    // 任务ID不能为空
    // 任务名称不能为空
    // 算法密钥不能为空
    // ROI参数必须有效
}
```

### ROI验证
```csharp
private void ValidateRoi(RoiDto roi)
{
    // ROI坐标必须在有效范围内
    // 宽度和高度必须大于0
    // X和Y坐标必须非负
}
```

## 依赖服务

- `HttpClient` - HTTP客户端（来自HttpClientFactory）
- `IOptions<RecognitionApiOptions>` - 识别API配置
- `ILogger<RecognitionApiService>` - 日志记录器

## 相关实体

- **RecognitionRequestDto** - 识别请求DTO
- **RecognitionResponseDto** - 识别响应DTO
- **RecognitionTaskDto** - 识别任务DTO
- **RecognitionResultDto** - 识别结果DTO
- **RoiDto** - 感兴趣区域DTO

## 配置项

### RecognitionApiOptions
```json
{
  "RecognitionApi": {
    "BaseAddress": "http://recognition-api-server",
    "ApiKey": "your-api-key-here",
    "TimeoutSeconds": 30,
    "MaxRetryCount": 3,
    "RetryDelayMilliseconds": 1000
  }
}
```

**配置说明**：
- `BaseAddress`: 识别API的基础地址
- `ApiKey`: API密钥（用于请求头 `X-API-Key`）
- `TimeoutSeconds`: 请求超时时间（秒）
- `MaxRetryCount`: 最大重试次数
- `RetryDelayMilliseconds`: 重试延迟时间（毫秒）

## 注意事项

- **超时处理**：所有请求都有超时限制，默认30秒
- **错误处理**：API调用失败时会抛出 `HttpRequestException`
- **参数验证**：严格的参数验证，确保请求的有效性
- **重试机制**：支持自动重试，提高请求成功率
- **日志记录**：详细的日志记录，便于问题排查
- **线程安全**：服务实现为线程安全，可并发调用

## 错误处理

### 常见错误类型
1. **参数验证错误** - `ArgumentException`
2. **API调用失败** - `HttpRequestException`
3. **超时错误** - `TimeoutException`
4. **服务不可用** - `ServiceUnavailableException`

### 错误响应示例
```json
{
  "success": false,
  "error": "识别请求不能为空",
  "errorCode": "INVALID_REQUEST"
}
```

## 服务监控

### 可用性检查
```csharp
public async Task<bool> IsServiceAvailableAsync()
{
    try
    {
        // 发送心跳请求检查服务可用性
        // 返回服务状态
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "识别API服务不可用");
        return false;
    }
}
```

### 性能指标
- 请求响应时间
- 成功率统计
- 错误类型分布

## API 集成示例

### 表计识别示例
```csharp
var request = new RecognitionRequestDto
{
    RequestId = Guid.NewGuid().ToString(),
    ImageUrl = "http://example.com/meter.jpg",
    Tasks = new List<RecognitionTaskDto>
    {
        new RecognitionTaskDto
        {
            TaskId = "meter_ocr",
            TaskName = "表计OCR识别",
            AlgorithmKey = "meter_ocr_v1",
            Roi = new RoiDto { X = 100, Y = 100, Width = 200, Height = 150 }
        }
    }
};

var result = await _recognitionApiService.RecognizeAsync(request);
```

## 相关文档链接

- [[AI识别服务]] - AI识别业务服务
- [[巡视执行服务]] - 巡视中的AI识别集成
- [[图像处理流程]] - 完整的图像处理和识别流程
