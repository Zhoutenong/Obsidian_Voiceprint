# 声纹音频上传接口

> **边缘设备音频采集核心入口** - 树莓派等边缘设备通过此接口上传原始声纹音频文件到后端服务

---

## 接口概览

| 属性 | 值 |
|------|-----|
| **HTTP 方法** | `POST` |
| **路由路径** | `/api/app/voiceprint/upload` |
| **Content-Type** | `multipart/form-data` |
| **认证方式** | `AllowAnonymous` + `X-Api-Key` 请求头 |
| **大小限制** | 100 MB |
| **源码位置** | `module/ast-voiceprint/Ast.Voiceprint.Application/Services/VoiceprintAudioAppService.cs:133` |
| **服务类** | `VoiceprintAudioAppService.UploadAsync` |

---

## 认证机制

### API Key 认证

接口支持两种认证模式：

```csharp
// VoiceprintStorageOptions 配置
public string ApiKey { get; set; }  // 为空则跳过认证
```

**请求头格式：**
```http
X-Api-Key: your-secret-api-key
```

**验证逻辑：**
```csharp
private void ValidateApiKey()
{
    if (string.IsNullOrWhiteSpace(_storageOptions.ApiKey))
    {
        return; // 未配置则跳过验证
    }

    var request = _httpContextAccessor.HttpContext?.Request;
    if (!request.Headers.TryGetValue("X-Api-Key", out var value))
    {
        throw new AbpAuthorizationException("Missing api key");
    }

    if (!string.Equals(_storageOptions.ApiKey, value, StringComparison.Ordinal))
    {
        throw new AbpAuthorizationException("Invalid api key");
    }
}
```

---

## 请求参数

### Input Model: `VoiceprintAudioUploadInput`

```csharp
public class VoiceprintAudioUploadInput
{
    [FromForm(Name = "file")]
    public IFormFile File { get; set; }        // 音频文件（必填）

    public Guid GroupId { get; set; }           // 采集批次ID（必填）
    public string DeviceId { get; set; }        // 设备ID（必填）
    public DateTime DateTime { get; set; }      // 采集时间（必填）
    public int DurationSeconds { get; set; }   // 音频时长（秒）
    public int RetryIndex { get; set; }         // 重试索引（用于幂等性）
}
```

### 请求示例（multipart/form-data）

```http
POST /api/app/voiceprint/upload HTTP/1.1
Host: api.example.com
X-Api-Key: your-secret-key
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="device_001.wav"
Content-Type: audio/wav

<binary audio data>
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="GroupId"

a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="DeviceId"

device-001
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="DateTime"

2026-06-03T10:30:00
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="DurationSeconds"

60
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="RetryIndex"

0
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

### 参数验证规则

```csharp
private static void ValidateInput(VoiceprintAudioUploadInput input)
{
    if (input.File == null || input.File.Length == 0)
        throw new UserFriendlyException("音频文件为空");

    if (input.GroupId == Guid.Empty)
        throw new UserFriendlyException("groupId 无效");

    if (string.IsNullOrWhiteSpace(input.DeviceId))
        throw new UserFriendlyException("deviceId 不能为空");
}
```

---

## 响应格式

### Response Model: `VoiceprintAudioUploadResultDto`

```csharp
public class VoiceprintAudioUploadResultDto
{
    public string FileName { get; set; }        // 保存的文件名
    public long FileSize { get; set; }          // 文件大小（字节）
    public string FilePath { get; set; }        // 公共访问路径
    public DateTime UploadTime { get; set; }    // 上传时间
    public string ContentType { get; set; }     // Content-Type
    public Guid GroupId { get; set; }           // 采集批次ID
    public string DeviceId { get; set; }        // 设备ID
}
```

### 成功响应示例

```json
{
  "fileName": "device_001_20260603103000123.wav",
  "fileSize": 9600000,
  "filePath": "/audio/unprocessed/device_001_20260603103000123.wav",
  "uploadTime": "2026-06-03T10:30:05.123Z",
  "contentType": "audio/wav",
  "groupId": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "deviceId": "device-001"
}
```

---

## 幂等性保证

### RetryIndex 机制

通过 `GroupId + DeviceId + RetryIndex` 组合实现上传幂等性：

| 字段 | 作用 |
|------|------|
| `GroupId` | 标识同一次采集批次（多个设备） |
| `DeviceId` | 标识批次内的具体设备 |
| `RetryIndex` | 标识同一设备+批次的重复上传尝试 |

**幂等性场景：**
- 网络中断后自动重试上传
- 相同参数多次上传返回相同结果
- 服务端可通过 `RetryIndex` 去重处理

---

## 请求大小限制

### 双重限制保护

```csharp
[RequestSizeLimit(104_857_600)]                    // 100MB
[RequestFormLimits(MultipartBodyLengthLimit = 104_857_600)] // 100MB
```

| 限制类型 | 值 | 说明 |
|----------|-----|------|
| `RequestSizeLimit` | 104,857,600 字节 (100MB) | 整个请求体大小 |
| `MultipartBodyLengthLimit` | 104,857,600 字节 (100MB) | multipart 表单数据大小 |

**ARM32 兼容性：**
此限制同时保护堆栈溢出风险（见 [[../系统架构/ARM32兼容性]])

---

## 超时自动恢复机制

### VoiceprintCaptureRuntimeStateService 协作

接口与 `VoiceprintCaptureRuntimeStateService` 协作实现超时自动恢复：

```csharp
if (_runtimeState.TryRecoverTimeout(out var timeoutRecoveredGroupId, out var timeoutReason))
{
    _logger.LogWarning("Voiceprint 手动采集超时自动恢复：groupId={GroupId}, reason={Reason}",
        timeoutRecoveredGroupId, timeoutReason);
    await TrySwitchAnalysisToOriginalAsync($"upload-timeout-recover:{timeoutRecoveredGroupId}");
}
```

**超时检测逻辑：**
```csharp
// VoiceprintCaptureRuntimeStateService.TryRecoverTimeout
public bool TryRecoverTimeout(out Guid? recoveredGroupId, out string reason)
{
    lock (_sync)
    {
        if (!_isManualCaptureRunning)
            return false;

        var elapsed = DateTime.UtcNow - _startedAtUtc;
        if (elapsed < _timeout)
            return false;

        recoveredGroupId = _currentGroupId;
        reason = $"manual capture timeout after {elapsed.TotalSeconds:F0}s";
        ResetUnsafe();
        return true;
    }
}
```

**超时配置：**
- 默认超时：10 分钟
- 可通过 `VoiceprintCaptureJobOptions` 配置
- 超时后自动恢复定时任务模式

**相关服务：**
- [[../后台任务/VoiceprintCaptureJob]] - 定时采集任务
- [[../../架构/VoiceprintCaptureRuntimeStateService]] - 运行时状态管理

---

## 文件存储路径规则

### 存储目录结构

```
{BaseDirectory}/wwwroot/audio/
├── unprocessed/          # 待处理文件（pending）
│   └── device_001_20260603103000123.wav
├── processed/            # 已处理文件
│   └── device_001_20260603103000123_processed.wav
├── logs/                 # 音频日志
└── standard_library/     # 标准音频库
```

### 路径生成逻辑

**物理路径：**
```csharp
string pendingDir = Path.Combine(_storageOptions.Root, _storageOptions.PendingFolderName);
// 示例: /wwwroot/audio/unprocessed

string fileName = GetSafeFileName(input);
string filePath = Path.Combine(pendingDir, fileName);
// 示例: /wwwroot/audio/unprocessed/device_001_20260603103000123.wav
```

**公共路径（返回给客户端）：**
```csharp
private static string BuildPublicPath(string folderName, string fileName)
{
    var combined = Path.Combine("audio", folderName, fileName).Replace("\\", "/");
    return "/" + combined.TrimStart('/');
    // 示例: /audio/unprocessed/device_001_20260603103000123.wav
}
```

### 文件命名规则

```csharp
private static string GetSafeFileName(VoiceprintAudioUploadInput input)
{
    var original = input.File?.FileName;
    if (!string.IsNullOrWhiteSpace(original))
    {
        // 优先使用客户端提供的文件名
        return Path.GetFileName(original);
    }

    // 兜底生成: {deviceId}_{timestamp}.wav
    var timestamp = input.DateTime == default ? DateTime.Now : input.DateTime;
    var safeDeviceId = string.IsNullOrWhiteSpace(input.DeviceId) ? "device" : input.DeviceId.Trim();
    return $"{safeDeviceId}_{timestamp:yyyyMMddHHmmssfff}.wav";
}
```

---

## 完整工作流程

```
┌─────────────┐                    ┌──────────────┐
│  树莓派/    │ 1. POST /upload    │   后端API    │
│  边缘设备   │────────────────────▶│  (本接口)    │
└─────────────┘                    └──────────────┘
      │                                  │
      │  multipart/form-data             │
      │  - file (binary)                 │
      │  - GroupId                        │  2. 验证 ApiKey
      │  - DeviceId                       │
      │  - DateTime                      │  3. 验证输入参数
      │  - DurationSeconds               │
      │  - RetryIndex                     │  4. 检查超时恢复
      │                                  │
      │                                  │  5. 保存文件到 pending/
      │                                  │
      │                                  │  6. 记录上传完成
      │                                  │     (TryMarkUploadCompleted)
      │                                  │
      │                                  │  7. 返回公共路径
      │                                  │
      │◀─────────────────────────────────│
      │     VoiceprintAudioUploadResultDto
      │     - filePath: /audio/unprocessed/...
      │
      ▼
┌─────────────────┐
│  后续处理流程    │
│                 │
│ • VoiceprintCaptureJob 发起采集
│ • 边缘设备采集音频
│ • 通过本接口上传
│ • [[./VoiceprintResultUploadAPI]] 上传识别结果
│ • [[../../后台任务/VoiceprintProcessedCleanupJob]] 清理旧文件
└─────────────────┘
```

---

## 错误处理

### 常见错误响应

| HTTP 状态 | 错误类型 | 原因 | 解决方案 |
|-----------|----------|------|----------|
| `401 Unauthorized` | `AbpAuthorizationException` | ApiKey 缺失或无效 | 检查 `X-Api-Key` 请求头 |
| `400 Bad Request` | `UserFriendlyException` | 音频文件为空 | 确保 `file` 字段包含有效文件 |
| `400 Bad Request` | `UserFriendlyException` | groupId/deviceId 无效 | 检查必填字段格式 |
| `413 Payload Too Large` | `InvalidOperationException` | 超过 100MB 限制 | 压缩音频或分段上传 |

### 错误响应示例

```json
{
  "error": {
    "code": "AbpAuthorizationException",
    "message": "Invalid api key",
    "details": null
  }
}
```

---

## 树莓派调用示例

### Curl 示例

```bash
curl -X POST "http://your-server/api/app/voiceprint/upload" \
  -H "X-Api-Key: your-secret-api-key" \
  -F "file=@/tmp/audio_001.wav" \
  -F "GroupId=a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11" \
  -F "DeviceId=device-001" \
  -F "DateTime=2026-06-03T10:30:00" \
  -F "DurationSeconds=60" \
  -F "RetryIndex=0"
```

### C# HttpClient 示例

```csharp
using var multipartContent = new MultipartFormDataContent();

// 添加音频文件
var fileContent = new ByteArrayContent(File.ReadAllBytes("audio_001.wav"));
fileContent.Headers.ContentType = new MediaTypeHeaderValue("audio/wav");
multipartContent.Add(fileContent, "file", "audio_001.wav");

// 添加表单字段
multipartContent.Add(new StringContent("a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11"), "GroupId");
multipartContent.Add(new StringContent("device-001"), "DeviceId");
multipartContent.Add(new StringContent("2026-06-03T10:30:00"), "DateTime");
multipartContent.Add(new StringContent("60"), "DurationSeconds");
multipartContent.Add(new StringContent("0"), "RetryIndex");

// 发送请求
using var client = new HttpClient();
client.DefaultRequestHeaders.Add("X-Api-Key", "your-secret-api-key");

var response = await client.PostAsync(
    "http://your-server/api/app/voiceprint/upload",
    multipartContent);

var result = await response.Content.ReadAsStringAsync();
Console.WriteLine(result);
```

### Python Requests 示例

```python
import requests
from datetime import datetime

url = "http://your-server/api/app/voiceprint/upload"
headers = {"X-Api-Key": "your-secret-api-key"}

files = {"file": open("audio_001.wav", "rb")}
data = {
    "GroupId": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
    "DeviceId": "device-001",
    "DateTime": "2026-06-03T10:30:00",
    "DurationSeconds": "60",
    "RetryIndex": "0"
}

response = requests.post(url, headers=headers, files=files, data=data)
print(response.json())
```

---

## 配置项

### appsettings.json

```json
{
  "VoiceprintStorage": {
    "Root": "wwwroot/audio",
    "PendingFolderName": "unprocessed",
    "ProcessedFolderName": "processed",
    "LogsFolderName": "logs",
    "StandardLibraryFolderName": "standard_library",
    "ApiKey": "your-secret-api-key-here"
  },
  "VoiceprintCaptureJob": {
    "Enabled": true,
    "CronExpression": "0 */10 * * * *",
    "DurationSeconds": 60,
    "Agents": [...]
  }
}
```

---

## 相关接口

| 接口 | 路由 | 功能 | 关系 |
|------|------|------|------|
| **本接口** | `POST /api/app/voiceprint/upload` | 上传原始音频 | 📥 **入口** |
| [[./VoiceprintResultUploadAPI]] | `POST /api/app/voiceprint/result` | 上传识别结果 | ⬇️ 下游 |
| [[./VoiceprintReportGenerateAPI]] | `POST /api/app/voiceprint/reports/generate-from-audio` | 生成分析报告 | ⬇️ 下游 |
| [[../后台任务/VoiceprintProcessedCleanupJob]] | (后台任务) | 清理旧文件 | 🗑️ 清理 |

**调用链：**
```
边缘设备采集
    ↓
本接口 (upload) - 保存音频到 pending/
    ↓
VoiceprintCaptureJob / 分析服务处理
    ↓
VoiceprintResultUploadAPI - 保存结果到 processed/
    ↓
VoiceprintProcessedCleanupJob - 定期清理
```

---

## 相关文档

- [[../系统架构/存储架构]] - 文件存储目录结构
- [[../后台任务/VoiceprintCaptureJob]] - 定时采集任务配置
- [[../../架构/VoiceprintCaptureRuntimeStateService]] - 运行时状态管理
- [[../../架构/ARM32兼容性]] - 边缘设备部署注意事项
- [[./VoiceprintResultUploadAPI]] - 识别结果上传接口
- [[../安全/API密钥管理]] - ApiKey 配置指南

---

## 元数据

| 属性 | 值 |
|------|-----|
| **文档版本** | v1.0.0 |
| **创建时间** | 2026-06-03 |
| **最后更新** | 2026-06-03 |
| **维护者** | Backend Team |
| **标签** | `#api` `#voiceprint` `#upload` `#edge-device` |

---

> **💡 提示：** 此接口是边缘设备与后端服务的核心集成点，上传的音频文件将触发后续的声纹分析和告警流程。
