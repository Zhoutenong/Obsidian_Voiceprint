# IEC61850 API 服务 (Iec61850ApiService)

## 概述

**Iec61850ApiService** 是智能变电站系统中负责与 IEC61850 服务器通信的核心服务。它提供了完整的 HTTP 客户端管理、请求验证、数据上报和错误处理功能，支持多种数据类型（浮点、整型、字符串、布尔、数组）的实时上报。

**源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/Iec61850ApiService.cs`

**注册方式**：`ITransientDependency` (瞬态依赖注入)

## IEC61850 协议

### 协议概述

IEC 61850 是**国际电工委员会（IEC）**制定的电力行业标准，用于变电站自动化系统内部的通信。它是现代智能变电站的核心通信协议。

### 核心特点

- **面向对象建模**：使用统一的建模语言描述变电站设备
- **抽象通信服务接口（ACSI）**：提供标准化的服务接口
- **自描述能力**：通过 ICD/SCD/SCL 模型文件描述设备配置
- **互操作性**：不同厂商设备可无缝通信

### 数据模型

```
IED (智能电子设备)
  ├── Logical Device (逻辑设备)
  │   ├── Logical Node (逻辑节点)
  │   │   ├── Data (数据对象)
  │   │   │   └── Data Attribute (数据属性)
```

### 应用场景

- **遥测（Measurement）**：模拟量数据（电压、电流、功率等）
- **遥信（Status）**：状态量数据（开关位置、告警状态等）
- **遥控（Control）**：远程控制命令
- **波形数据**：故障录波、谐波分析等数组型数据

## 核心职责

### 1. HTTP 客户端管理

- 基于 **IHttpClientFactory** 创建和管理 HttpClient 实例
- 利用工厂内部的连接池管理，无需手动缓存 HttpClient
- 支持动态配置多个 61850 服务器地址
- 自动处理协议前缀（http://）

### 2. 请求验证

- **Host 验证**：确保服务器地址有效
- **数据验证**：检查请求数据非空，点位引用（Reference）必填
- **时间戳处理**：自动填充默认时间戳（UTC 毫秒）

### 3. 数据上报

支持 5 种数据类型上报：
| 数据类型 | API 端点 | 用途 |
|---------|---------|------|
| Float | `/iedServer/report/float` | 遥测值（温度、湿度等） |
| Int | `/iedServer/report/int` | 遥信值（整数状态） |
| String | `/iedServer/report/string` | 遥信值（字符串状态） |
| Boolean | `/iedServer/report/boolean` | 遥信值（开关状态） |
| Array | `/iedServer/report/array` | 波形数据 |

### 4. 错误处理与重试

- **自动重试机制**：可配置重试次数和延迟时间
- **指数退避**：重试延迟逐步增加（delay * attempt）
- **HTTP 状态码处理**：
  - `400 Bad Request` → 参数错误
  - `404 Not Found` → 服务端点不存在
  - `504 Gateway Timeout` → 响应超时
  - `500 Internal Server Error` → 服务内部错误

### 5. 健康检查

提供 `IsServiceAvailableAsync` 方法检查 61850 服务可用性。

## 主要接口

### 批量上报接口

#### ReportFloatAsync
```csharp
Task<Iec61850ApiResponseDto<Iec61850FloatResponseDto>> ReportFloatAsync(
    string host,
    List<Iec61850FloatRequestDto> requests)
```
上报浮点型遥测值（如温度、湿度、电压等）。

#### ReportIntAsync
```csharp
Task<Iec61850ApiResponseDto<Iec61850IntResponseDto>> ReportIntAsync(
    string host,
    List<Iec61850IntRequestDto> requests)
```
上报整型遥信值，支持描述字段。

#### ReportStringAsync
```csharp
Task<Iec61850ApiResponseDto<Iec61850StringResponseDto>> ReportStringAsync(
    string host,
    List<Iec61850StringRequestDto> requests)
```
上报字符串遥信值（如设备状态描述）。

#### ReportBooleanAsync
```csharp
Task<Iec61850ApiResponseDto<Iec61850BooleanResponseDto>> ReportBooleanAsync(
    string host,
    List<Iec61850BooleanRequestDto> requests)
```
上报布尔型遥信值（如开关位置）。

#### ReportArrayAsync
```csharp
Task<Iec61850ApiResponseDto<Iec61850ArrayResponseDto>> ReportArrayAsync(
    string host,
    List<Iec61850ArrayRequestDto> requests)
```
上报数组型遥测值（波形数据）。

### 单点上报接口

#### ReportSingleFloatAsync
```csharp
Task<Iec61850OperationResultDto> ReportSingleFloatAsync(
    string host,
    string reference,
    float value,
    int quality = 0,
    long? timestamp = null)
```
便捷方法：上报单个浮点值，返回简化的操作结果。

#### ReportSingleIntAsync / ReportSingleStringAsync / ReportSingleBooleanAsync / ReportSingleArrayAsync
类似的单点上报方法，提供便捷的单点数据上报。

### 健康检查接口

#### IsServiceAvailableAsync
```csharp
Task<bool> IsServiceAvailableAsync(string host)
```
检查 61850 服务是否可用。通过发送最小测试请求验证连通性。

### 辅助接口

#### CreateHttpClient
```csharp
HttpClient CreateHttpClient(string host)
```
创建配置好的 HttpClient 实例：
- 设置 BaseAddress
- 配置超时时间
- 命名客户端：`Iec61850-{host}`

#### ValidateRequests
```csharp
void ValidateRequests<T>(List<T> requests, string requestType)
    where T : Iec61850BaseRequestDto
```
验证请求数据：
- 非空检查
- Reference 必填检查

## 依赖服务

### 注入的服务

```csharp
public Iec61850ApiService(
    IHttpClientFactory httpClientFactory,
    ILogger<Iec61850ApiService> logger,
    IOptions<Iec61850ApiOptions> options)
```

| 服务 | 用途 |
|-----|------|
| `IHttpClientFactory` | 创建和管理 HttpClient 实例 |
| `ILogger<Iec61850ApiService>` | 日志记录 |
| `IOptions<Iec61850ApiOptions>` | API 配置选项 |

### 被依赖的服务

- [[Iec61850DataReportHandler]] - 事件处理器，调用此服务上报数据
- [[IIec61850Processor]] - 数据处理器实现，通过此服务与 61850 服务器通信

## 配置选项

### Iec61850ApiOptions

配置于 `appsettings.json`：

```json
{
  "Iec61850Api": {
    "DefaultBaseAddress": "",
    "TimeoutSeconds": 30,
    "RetryCount": 2,
    "RetryDelayMs": 1000,
    "EnableLogging": true,
    "EnableConnectionPooling": true,
    "MaxConnectionsPerServer": 10,
    "ConnectionIdleTimeoutMinutes": 5
  }
}
```

| 配置项 | 默认值 | 说明 |
|-------|--------|------|
| `TimeoutSeconds` | 30 | HTTP 请求超时时间（秒） |
| `RetryCount` | 2 | 失败重试次数 |
| `RetryDelayMs` | 1000 | 重试基础延迟（毫秒） |
| `EnableLogging` | true | 是否启用详细日志 |
| `EnableConnectionPooling` | true | 是否启用连接池 |
| `MaxConnectionsPerServer` | 10 | 每服务器最大连接数 |
| `ConnectionIdleTimeoutMinutes` | 5 | 连接空闲超时（分钟） |

## 数据传输对象

### 请求 DTO

#### Iec61850BaseRequestDto
所有请求的基类：
- `Reference` (string, required) - 点位引用，如 `LD0/LN0.DO.identifier`
- `Quality` (int, default: 0) - 质量码（0=GOOD, 1=INVALID, 2=RESERVED, 3=QUESTIONABLE）
- `Timestamp` (long?, optional) - 时标（毫秒）
- `Desc` (string?, optional) - 数据描述（仅 Int/String）

#### 类型特定请求
- `Iec61850FloatRequestDto` - 添加 `Value` (float)
- `Iec61850IntRequestDto` - 添加 `Value` (int)
- `Iec61850StringRequestDto` - 添加 `Value` (string)
- `Iec61850BooleanRequestDto` - 添加 `Value` (bool)
- `Iec61850ArrayRequestDto` - 添加 `Value` (float[])

### 响应 DTO

#### Iec61850ApiResponseDto\<T\>
批量上报的响应：
```csharp
{
    "code": 0,
    "message": "success",
    "data": [ /* 具体数据响应 */ ]
}
```

#### Iec61850OperationResultDto
单点上报的简化响应：
```csharp
{
    "code": 0,  // 0=成功, 1=点位不存在, 2=类型错误, 3=长度不匹配
    "msg": "success"
}
```

## 调用流程

### 数据上报流程

```
传感器数据更新
    ↓
[[ProcessedPointValueEventArgs]] 事件发布
    ↓
[[Iec61850DataReportHandler]] 处理事件
    ↓
查找 [[IIec61850MappingManager]] 映射关系
    ↓
按 ICD 文件分组
    ↓
[[IIec61850Processor]] 处理数据
    ↓
调用 [[Iec61850ApiService]] 上报接口
    ↓
发送 HTTP POST 到 61850 服务器
    ↓
返回操作结果
```

### 批量处理流程

1. **事件触发**：`ProcessedPointValueEventArgs` 包含多个点位值
2. **映射查找**：根据 `deviceId|sensorKey|property` 查找 61850 映射
3. **分组处理**：按 ICD 文件名分组，不同 ICD 使用不同处理器
4. **并发上报**：多个 ICD 分组并发处理
5. **结果聚合**：收集所有处理结果

## 错误处理

### 重试机制

```csharp
for (int attempt = 0; attempt <= _options.RetryCount; attempt++)
{
    try {
        // 发送请求
        return response;
    }
    catch (HttpRequestException / TaskCanceledException) {
        // 记录警告日志
        // 延迟：RetryDelayMs * (attempt + 1)
    }
}
// 所有重试失败后抛出 UserFriendlyException
```

### HTTP 状态码处理

| 状态码 | 处理方式 | 用户提示 |
|-------|---------|----------|
| 400 | 抛出 `UserFriendlyException` | "请求参数错误" |
| 404 | 抛出 `UserFriendlyException` | "IEC61850 服务端点不存在" |
| 504 | 抛出 `UserFriendlyException` | "IEC61850 服务响应超时" |
| 500 | 抛出 `UserFriendlyException` | "IEC61850 服务内部错误" |

### 日志记录

| 级别 | 场景 |
|-----|------|
| `LogDebug` | 创建 HttpClient、发送请求、收到响应（`EnableLogging=true`） |
| `LogWarning` | 单次重试失败、超时 |
| `LogError` | 最终失败、HTTP 错误响应 |
| `LogInformation` | 请求处理成功 |

## 优化要点

### HttpClient 生命周期管理

**❌ 错误做法**（已移除）：
```csharp
private readonly ConcurrentDictionary<string, HttpClient> _httpClients;
```

**✅ 正确做法**（当前实现）：
```csharp
private HttpClient GetHttpClient(string host)
{
    return CreateHttpClient(host);  // 每次从工厂获取
}
```

**原因**：
- `IHttpClientFactory` 内部已有完善的连接池管理
- 避免 HttpClient 耗尽问题
- 自动处理 DNS 刷新和连接复用

### JSON 序列化优化

```csharp
new JsonSerializerSettings
{
    ContractResolver = new CamelCasePropertyNamesContractResolver(),
    Formatting = Formatting.None  // 紧凑格式，减少传输大小
}
```

### 自动时间戳

```csharp
if (!baseRequest.Timestamp.HasValue)
{
    baseRequest.Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
}
```

## 使用示例

### 批量上报遥测值

```csharp
var requests = new List<Iec61850FloatRequestDto>
{
    new() { Reference = "LD0/LLN0.Temp.phsA", Value = 25.5f, Quality = 0 },
    new() { Reference = "LD0/LLN0.Temp.phsB", Value = 26.1f, Quality = 0 }
};

var response = await _iec61850ApiService.ReportFloatAsync("192.168.1.100:8080", requests);
```

### 单点上报状态

```csharp
var result = await _iec61850ApiService.ReportSingleBooleanAsync(
    "192.168.1.100:8080",
    "LD0/LLN0.Switch.stVal",
    true,
    quality: 0
);
```

### 健康检查

```csharp
bool isAvailable = await _iec61850ApiService.IsServiceAvailableAsync("192.168.1.100:8080");
```

## 相关文档

- [[Iec61850DataReportHandler]] - 数据上报事件处理器
- [[IIec61850MappingManager]] - 映射关系管理器
- [[IIec61850Processor]] - 数据处理器接口
- [[IEC61850 数据上报架构]] - 整体架构说明
- [[IEC61850 数据上报服务]] - 服务端配置
- [[Iec61850Options]] - 全局配置选项
