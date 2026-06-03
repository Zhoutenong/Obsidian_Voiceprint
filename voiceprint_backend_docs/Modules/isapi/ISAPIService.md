# ISAPI 服务 (ISAPIService)

## 概述

ISAPIService 是智能变电站监控系统中专门用于与热成像摄像机进行 ISAPI 协议通信的核心服务。该服务负责获取热成像数据、温度矩阵、红外图像和可见光图像，并提供温度分析功能。

**技术特点**：
- 支持热成像摄像机 ISAPI 协议（基于 ONVIF/PSIA 标准）
- Digest 认证机制确保通信安全
- Multipart 数据流解析（热成像图 + 可见光图 + 温度矩阵）
- 设备锁机制防止并发访问冲突
- 智能通道检测和故障自动重试

## ISAPI 协议

### 协议概述
ISAPI (Integrated Security API) 是基于 ONVIF/PSIA 标准的设备管理协议，广泛应用于安防和热成像设备。

### 主要功能
- **设备信息获取**：系统设备信息、通道信息
- **热成像能力查询**：支持测温的通道号检测
- **温度采集**：单点测温、区域温度范围、温度矩阵
- **图像获取**：热成像图、可见光图、带附加数据的 JPEG 图

### 应用场景
- 变电站设备温度监测
- 热成像温度分析
- 设备异常预警
- 温度可视化

## 核心职责

### 1. 设备通信管理
- **设备锁管理**：使用 `SemaphoreSlim` 实现设备级别的并发控制
- **认证处理**：Digest 认证机制的完整实现
- **故障重试**：请求失败时自动重试机制
- **资源清理**：定期清理过期的设备锁资源

### 2. 热成像数据获取
- **双光图像采集**：同时获取热成像图和可见光图
- **温度矩阵解析**：从 Multipart 流中提取温度数据
- **能力查询**：自动检测支持测温的通道号
- **区域温度分析**：计算指定区域的最高/最低温度

### 3. 图像处理
- **温度标记绘制**：在热成像图上绘制测温区域和温度值
- **多区域绘制**：支持同时绘制多个测温区域
- **图像存储**：将抓拍图像保存到本地文件系统

### 4. 坐标转换
- **像素到矩阵映射**：将图像像素坐标转换为温度矩阵坐标
- **边界校验**：防止坐标越界导致的数据异常

## 主要接口

### GetTemperatureCapabilitiesAsync
获取热成像能力信息，用于查询设备支持的测温通道。

```csharp
Task<HttpResponseMessage> GetTemperatureCapabilities(string deviceId)
```

### GetThermalCapabilitiesContent
获取支持测温的通道号（带缓存）。

```csharp
Task<string> GetThermalCapabilitiesContent(string deviceId)
```

**返回值**：支持测温的通道号（如 "2"），缓存 10 分钟

### GetJpegPicWithAppendData
获取带附加数据的 JPEG 图片（热成像图 + 可见光图 + 温度矩阵）。

```csharp
Task<HttpResponseMessage> GetJpegPicWithAppendData(string deviceId, string channelId = "2")
```

**特性**：
- 使用设备锁防止并发冲突
- 失败时自动重试一次
- 返回 Multipart 格式的 HTTP 响应

### GetJpegPicWithAppendDataParsed
解析带附加数据的 JPEG 图片，分离各部分数据。

```csharp
Task<ThermalResult> GetJpegPicWithAppendDataParsed(string deviceId, string channelId = "2")
```

**返回值**：`ThermalResult` 包含：
- `ThermalImage`：热成像 JPEG 图片
- `VisibleImage`：可见光 JPEG 图片
- `TempMatrixRaw`：温度矩阵原始数据
- `MetaJson`：元数据 JSON

### GetTemperatureMatrixAsync
获取完整的温度矩阵数据。

```csharp
Task<float[,]> GetTemperatureMatrixAsync(string deviceId, int channelId)
```

**返回值**：二维浮点数数组，表示每个像素点的温度值

### GetTemperatureRangeAsync
获取指定矩形区域内的最小和最大温度值。

```csharp
Task<ISAPIDto> GetTemperatureRangeAsync(GetTemperatureRangeInputDto input)
```

**参数**：
- `input.DeviceId`：设备 ID
- `input.Area`：矩形区域坐标（矩阵坐标系）

**异常**：区域越界时抛出 `ArgumentOutOfRangeException`

### GetThermalAndVisibleImagesAsync
获取热成像和可见光图像并保存到本地。

```csharp
Task<(string thermalWebPath, string visibleWebPath)> GetThermalAndVisibleImagesAsync(string deviceId)
```

**返回值**：
- `thermalWebPath`：热成像图的 Web 访问路径
- `visibleWebPath`：可见光图的 Web 访问路径（可能为 null）

### GetThermalImageWithTemperatureMarkAsync
获取带温度标记的热成像图。

```csharp
Task<string> GetThermalImageWithTemperatureMarkAsync(GetTemperatureRangeInputDto input)
```

**功能**：
- 在热成像图上绘制测温区域矩形框
- 显示区域最高温度值
- 支持多种字体回退机制

### GetThermalImageWithAreasAndMaxMarkAsync
获取带多个温度标记的热成像图。

```csharp
Task<string> GetThermalImageWithAreasAndMaxMarkAsync(string deviceId, IEnumerable<PixelAreaDto> areas)
```

**特性**：
- 支持同时绘制多个测温区域
- 自动进行像素到矩阵的坐标转换
- 每个区域显示对应的最高温度

### GetTemperatureFromDeviceAsync
根据识别区域坐标获取设备温度。

```csharp
Task<double> GetTemperatureFromDeviceAsync(string deviceId, string recognitionArea)
```

**参数**：
- `recognitionArea`：JSON 格式的像素区域坐标

**流程**：
1. 解析识别区域坐标
2. 获取温度矩阵
3. 进行像素到矩阵的坐标转换
4. 计算区域最高温度

### ClickToThermometryAsync
单点测温功能。

```csharp
Task<ClickToThermometryResult> ClickToThermometryAsync(string deviceId, int channelId, double positionX, double positionY)
```

**参数**：
- `positionX`：归一化 X 坐标（0~1）
- `positionY`：归一化 Y 坐标（0~1）

### SaveImageAsync
保存图片到本地文件系统。

```csharp
Task<string> SaveImageAsync(byte[] imageBytes, string fileExtension, string nvrId, string cameraId)
```

**存储路径**：`snapshot/{日期}/{NVR_ID}/{Camera_ID}/{时间戳}_{GUID}.jpg`

## 认证机制

### Digest 认证流程
1. 首次请求返回 401 Unauthorized
2. 解析 WWW-Authenticate 头获取认证参数
3. 计算 MD5 哈希值：
   - HA1 = MD5(username:realm:password)
   - HA2 = MD5(method:uri)
   - Response = MD5(HA1:nonce:nc:cnonce:qop:HA2)
4. 重新发送带 Authorization 头的请求

### 支持的认证参数
- `realm`：认证域
- `nonce`：服务器生成的随机数
- `qop`：保护质量（auth/auth-int）
- `algorithm`：算法（MD5）
- `cnonce`：客户端随机数
- `nc`：请求计数器

## 数据结构

### ThermalResult
带附加数据的热成像结果。

```csharp
public class ThermalResult
{
    public byte[] ThermalImage { get; set; }      // 热成像图
    public byte[] VisibleImage { get; set; }      // 可见光图
    public byte[] TempMatrixRaw { get; set; }     // 温度矩阵原始数据
    public string MetaJson { get; set; }          // 元数据 JSON
}
```

### ClickToThermometryResult
单点测温结果。

```csharp
public class ClickToThermometryResult
{
    public double PositionX { get; set; }    // X 坐标
    public double PositionY { get; set; }    // Y 坐标
    public double Temperature { get; set; }  // 温度值
    public string Unit { get; set; }        // 单位
}
```

### ISAPIDto
温度范围结果。

```csharp
public class ISAPIDto
{
    public float Min { get; set; }  // 最低温度
    public float Max { get; set; }  // 最高温度
}
```

### RectAreaDto
矩形区域参数（矩阵坐标系）。

```csharp
public class RectAreaDto
{
    public int TopLeftX { get; set; }   // 左上角 X 坐标
    public int TopLeftY { get; set; }   // 左上角 Y 坐标
    public int Width { get; set; }      // 宽度
    public int Height { get; set; }     // 高度
}
```

### PixelAreaDto
像素区域参数（图像坐标系）。

```csharp
public class PixelAreaDto
{
    public int x { get; set; }      // X 坐标
    public int y { get; set; }      // Y 坐标
    public int width { get; set; }  // 宽度
    public int height { get; set; } // 高度
}
```

## 并发控制

### 设备锁机制
使用 `SemaphoreSlim` 实现设备级别的并发控制：

```csharp
private static readonly ConcurrentDictionary<string, SemaphoreSlim> _deviceLocks = new();
private static readonly ConcurrentDictionary<string, DateTime> _lockLastUsedTime = new();
```

### 获取设备锁
```csharp
using (await AcquireDeviceLockAsync(deviceId))
{
    // 执行设备操作
}
```

### 清理过期锁
定期清理长时间未使用的设备锁：

```csharp
public static async Task CleanupExpiredLocksAsync(TimeSpan maxAge)
```

## Multipart 数据解析

### 响应格式
ISAPI 设备返回的 Multipart 响应包含多个部分：

1. **application/json**：元数据（图像尺寸、温度数据长度、缩放因子等）
2. **image/jpeg**：热成像 JPEG 图片
3. **image/jpeg**：可见光 JPEG 图片（可选）
4. **application/octet-stream**：温度矩阵原始数据

### 解析流程
1. 从 Content-Type 头获取 boundary
2. 按 boundary 分割响应体
3. 分离每个部分的 header 和 body
4. 根据 Content-Type 分发到对应的数据结构

### 温度矩阵还原
根据 `temperatureDataLength` 字段选择还原算法：

- **4 字节**：直接转换为 `float`
- **2 字节**：使用 `scale` 和 `offset` 转换为摄氏度

## 坐标转换

### 像素坐标 → 矩阵坐标
```csharp
double sx = (double)matrixWidth / imageWidth;
double sy = (double)matrixHeight / imageHeight;

int matrixX = (int)Math.Round(pixelX * sx);
int matrixY = (int)Math.Round(pixelY * sy);
```

### 边界保护
所有坐标转换都包含边界校验，防止越界访问。

## 缓存机制

### 热成像通道缓存
使用 `IMemoryCache` 缓存支持测温的通道号，缓存时间 10 分钟：

```csharp
var key = $"therm_ch:{deviceId}";
if (_cache.TryGetValue(key, out string ch)) return ch;
```

## 错误处理

### 重试机制
对于失败的请求，自动重试一次：

```csharp
if (!response.IsSuccessStatusCode)
{
    Logger.LogError("请求失败1: ...");
    await Task.Delay(300);
    response = await SendAuthenticatedRequest(...);
    response.EnsureSuccessStatusCode();
}
```

### 异常情况
- 设备不存在：抛出异常
- 区域越界：抛出 `ArgumentOutOfRangeException`
- 认证失败：返回 401/403 状态码
- 数据解析失败：抛出异常并记录日志

## 依赖服务

### 数据访问层
- `ISqlSugarRepository<DeviceEntity, string>`：设备信息仓储

### 基础设施
- `HttpClient`：HTTP 通信客户端
- `IMemoryCache`：内存缓存
- `ILogger`：日志记录
- `IOptions<StreamingApiOptions>`：配置选项

### 外部依赖
- `SixLabors.ImageSharp`：图像处理库
- `Newtonsoft.Json`：JSON 序列化

## 配置选项

### StreamingApiOptions
```csharp
public class StreamingApiOptions
{
    public string SnapshotRootPath { get; set; }  // 截图存储根路径
}
```

## 使用示例

### 获取设备温度范围
```csharp
var input = new GetTemperatureRangeInputDto
{
    DeviceId = "device123",
    Area = new RectAreaDto
    {
        TopLeftX = 100,
        TopLeftY = 100,
        Width = 200,
        Height = 150
    }
};

var result = await _isapiService.GetTemperatureRangeAsync(input);
Console.WriteLine($"最高温度: {result.Max}°C, 最低温度: {result.Min}°C");
```

### 获取带温度标记的热成像图
```csharp
var markedImagePath = await _isapiService.GetThermalImageWithTemperatureMarkAsync(input);
```

### 多区域温度分析
```csharp
var areas = new List<PixelAreaDto>
{
    new PixelAreaDto { x = 100, y = 100, width = 200, height = 150 },
    new PixelAreaDto { x = 400, y = 300, width = 150, height = 100 }
};

var markedImagePath = await _isapiService.GetThermalImageWithAreasAndMaxMarkAsync(deviceId, areas);
```

## 注意事项

### 性能优化
- 使用设备锁避免并发冲突
- 缓存热成像通道号减少查询
- 定期清理过期的设备锁资源

### 安全考虑
- Digest 认证避免密码明文传输
- 路径清理防止路径遍历攻击
- 原子性文件写入防止数据损坏

### 兼容性
- 支持多种字体回退机制
- 兼容不同厂商的 ISAPI 实现
- 处理边界情况和异常数据

### 调试支持
- 详细的错误日志记录
- 请求失败时记录响应内容
- 支持 Multipart 各部分的 headers 输出

## 相关文档

### 内部模块
- [[IEC61850数据上报服务]]
- [[数据上报流程]]

### 相关服务
- [[AIRecognitionService]]：AI 识别服务
- [[InfraredTemperatureCollectionJob]]：红外温度采集后台任务

### 数据实体
- [[DeviceEntity]]：设备实体
- [[ISAPIDto]]：ISAPI 数据传输对象

### 技术文档
- [[热成像摄像机ISAPI协议]]
- [[Digest认证机制]]
- [[Multipart数据解析]]

## 附录

### ISAPI 端点列表

| 功能 | 端点 | 方法 |
|------|------|------|
| 设备信息 | `/ISAPI/System/deviceInfo` | GET |
| 通道信息 | `/ISAPI/AUXInfo/attributes/Channels` | GET |
| 热成像能力 | `/ISAPI/Thermal/capabilities` | GET |
| 温度采集 | `/ISAPI/Thermal/temperature/collection` | POST |
| 双光图像 | `/ISAPI/Thermal/channels/{id}/thermometry/jpegPicWithAppendData` | GET |
| 单点测温 | `/ISAPI/Thermal/channels/{id}/clickToThermometry/rules/1` | PUT |
| 可见光快照 | `/ISAPI/Streaming/channels/1/picture` | GET |

### 温度数据格式

#### 4 字节格式（float）
```csharp
float temperature = BitConverter.ToSingle(data, index);
```

#### 2 字节格式（int16 + scale/offset）
```csharp
short rawValue = BitConverter.ToInt16(data, index);
float temperature = rawValue / scale + offset - 273.15f;
```

### 字体回退顺序
1. DejaVu Sans
2. Arial
3. Liberation Sans
4. sans-serif（通用）

---

**最后更新**：2024-06-03
**维护者**：智能变电站监控团队
