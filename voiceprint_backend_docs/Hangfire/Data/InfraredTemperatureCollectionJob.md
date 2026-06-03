# 红外测温采集任务 (InfraredTemperatureCollectionJob)

## 概述

定期从红外热成像设备采集温度数据，通过事件总线上报到系统。支持多区域温度测量、热力图生成和温度矩阵分析。

**职责：**
- 从红外设备获取热成像图片和温度矩阵
- 按识别区域计算最大温度值
- 生成多区域标注热力图
- 通过事件总线上报温度数据
- 支持多设备并发采集

## 任务配置

### appsettings.json 配置

```json
{
  "InfraredTemperatureCollection": {
    "Enabled": true,
    "CronExpression": "*/3 * * * *"
  }
}
```

### 配置选项说明

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Enabled` | bool | true | 是否启用任务 |
| `CronExpression` | string | "*/3 * * * *" | Cron表达式（每3分钟） |

## Cron 表达式说明

| 表达式 | 说明 |
|--------|------|
| `*/3 * * * *` | 每3分钟执行（默认） |
| `*/5 * * * *` | 每5分钟执行 |
| `*/10 * * * *` | 每10分钟执行 |
| `0 */3 * * *` | 每3小时的整点执行 |
| `0 * * * *` | 每小时执行 |

## 执行流程

```
获取红外测温点位 → 按监测点位+设备+预置位分组 → 获取热成像和温度矩阵 → 计算区域温度 → 生成标注图 → 上报数据
```

### 详细步骤

1. **检查任务启用状态**
   - 读取 `Enabled` 配置
   - 禁用时跳过执行

2. **获取红外测温点位**
   ```csharp
   var input = new RtMonitoredPointBindingGetListInputDto
   {
       AlgorithmType = AlgorithmTypeEnum.InfraredTemperatureMeasurement,
       IncludeUnbound = false,
       DataSrcType = DataSrcTypeEnum.RealTimeMonitoring
   };
   ```

3. **分组处理点位**
   - 按 `监测点位ID + 设备ID + 预置位号` 组合分组
   - 同一分组的点位使用同一张热成像图

4. **获取温度数据**
   - 获取热成像图片
   - 获取温度矩阵
   - 计算识别区域在温度矩阵中的对应区域

5. **计算区域温度**
   - 将像素区域映射到温度矩阵
   - 计算区域内的最大温度值
   - 保留一位小数

6. **生成标注图**
   - 在热成像图上标注所有识别区域
   - 标记最高温度位置
   - 生成标注图片URL

7. **上报数据**
   - 通过事件总线发布
   - 包含温度值、图片URL、区域信息

## 数据采集逻辑

### 热成像图片获取

```csharp
// 获取热成像和可见光图片
var thermalResult = await _isapiService.GetThermalAndVisibleImagesAsync(deviceId);
var thermalWebPath = thermalResult.thermalWebPath;
```

### 温度矩阵获取

```csharp
// 获取支持测温的通道号
var channelId = await _isapiService.GetThermalCapabilitiesContent(deviceId);

// 获取温度矩阵
var tempMatrix = await _isapiService.GetTemperatureMatrixAsync(deviceId, int.Parse(channelId));
```

### 坐标映射

**图像尺寸链路：**
```
原始截图(预置位) → 热成像图片 → 温度矩阵
     ↓              ↓            ↓
  RecArea     ThermalArea   MatrixArea
 (像素坐标)    (像素坐标)    (矩阵索引)
```

**映射公式：**
```csharp
// 原始 → 热成像
thermalX = originalX * thermalWidth / originalWidth
thermalY = originalY * thermalHeight / originalHeight

// 热成像 → 矩阵
matrixX = thermalX * matrixWidth / thermalWidth
matrixY = thermalY * matrixHeight / thermalHeight
```

### 温度计算

```csharp
// 获取区域内最大温度
var maxTemp = ISAPIService.GetMaxInArea(tempMatrix, matrixArea);
```

## 设备连接

### ISAPI 服务接口

任务通过 `IISAPIService` 与红外设备通信：

| 方法 | 说明 |
|------|------|
| `GetThermalAndVisibleImagesAsync` | 获取热成像和可见光图片 |
| `GetThermalCapabilitiesContent` | 获取测温通道号 |
| `GetTemperatureMatrixAsync` | 获取温度矩阵数据 |
| `GetThermalImageWithAreasAndMaxMarkAsync` | 生成多区域标注热力图 |

### 设备要求

- 支持热成像功能的摄像头
- 支持 ISAPI 协议
- 具备温度矩阵输出能力
- 配置预置位和识别区域

## 数据上报

### 事件总线格式

```csharp
var eventArgs = new PointValueEventArgs
{
    Type = "infrared_temperature_collection",
    Data = new MqPointValueDto
    {
        PointVals = temperatureData
    }
};

await _localEventBus.PublishAsync(eventArgs);
```

### 点位值数据结构

```csharp
public class MqPointValueItemDto
{
    public string PointId { get; set; }        // 点位ID
    public DateTime Ts { get; set; }           // 采集时间
    public string DeviceId { get; set; }      // 设备ID
    public string SensorKey { get; set; }     // 传感器Key
    public string Property { get; set; }       // 属性
    public string GroupId { get; set; }        // 分组ID（同一次采集的点位相同）
    public double Value { get; set; }          // 温度值（保留一位小数）
    public int ValueType { get; set; }         // 值类型
    public string ExtInfo { get; set; }        // 扩展信息（JSON）
    public AlarmLevelEnum AlarmLevel { get; set; } // 告警级别
}
```

### ExtInfo 内容

```json
{
  "url": "标注热力图URL",
  "thermalArea": {
    "x": 100,
    "y": 200,
    "width": 300,
    "height": 400
  },
  "matrixArea": {
    "topLeftX": 10,
    "topLeftY": 20,
    "width": 30,
    "height": 40
  }
}
```

## 依赖服务

- [[IRealtimeMonitoringPointService]] - 实时监测点位服务
- [[IISAPIService]] - ISAPI 设备通信服务
- [[ILocalEventBus]] - 本地事件总线
- [[ISqlSugarRepository<SensorEntity>]] - 传感器数据仓储

## 相关任务

- [[EnvironmentDetectionCollectionJob]] - 环境检测采集任务
- [[PointDataCleanupJob]] - 点位数据清理任务

## 监控和日志

### 日志级别

```csharp
Logger.LogInformation("开始执行 [InfraredTemperatureCollectionJob] 红外测温数据采集任务...");
Logger.LogInformation("找到 {Count} 个红外测温点位，开始采集数据", count);
Logger.LogInformation("监测点位组 {Name} 采集并上报了 {Count} 个温度数据", name, count);
Logger.LogWarning("未获取到热成像图片: DeviceId={DeviceId}", deviceId);
Logger.LogError(ex, "红外测温数据采集任务执行失败");
```

### Hangfire Dashboard

访问路径：`/hangfire`

查看任务执行历史、状态和性能指标。

### 监控指标

- 采集点位数量
- 成功采集数量
- 失败采集数量
- 平均执行时间
- 温度异常告警

## 异常处理

### 图片获取失败

```csharp
if (string.IsNullOrEmpty(thermalWebPath))
{
    Logger.LogWarning("未获取到热成像图片: DeviceId={DeviceId}", deviceId);
    continue; // 跳过该设备
}
```

### 温度矩阵获取失败

```csharp
if (string.IsNullOrEmpty(channelId))
{
    Logger.LogWarning("未获取到支持测温的通道号: DeviceId={DeviceId}", deviceId);
    continue;
}
```

### 区域映射失败

```csharp
try
{
    // 计算区域映射
}
catch (Exception ex)
{
    Logger.LogWarning(ex, "处理点位区域失败: DeviceId={DeviceId}, PointId={PointId}");
    continue; // 跳过该点位
}
```

### 标注图生成失败

```csharp
try
{
    annotatedUrl = await _isapiService.GetThermalImageWithAreasAndMaxMarkAsync(deviceId, thermalAreas);
}
catch (Exception ex)
{
    Logger.LogWarning(ex, "生成多区域标注热力图失败: DeviceId={DeviceId}", deviceId);
    // 继续处理，使用原始热成像图
}
```

## 注意事项

### 并发控制

```csharp
[DisableConcurrentExecution(timeoutInSeconds: 10 * 60)]
```
- 防止任务重叠执行
- 超时时间：10分钟
- 前一个任务运行时，后续调度将被跳过

### 分组策略

- 按 `监测点位ID + 设备ID + 预置位号` 分组
- 同一分组的点位共享同一张热成像图
- 减少设备调用次数，提高效率

### 图片尺寸处理

```csharp
// 确保坐标在有效范围内
if (thermalArea.x < 0) thermalArea.x = 0;
if (thermalArea.y < 0) thermalArea.y = 0;
if (thermalArea.width <= 0) thermalArea.width = 1;
if (thermalArea.height <= 0) thermalArea.height = 1;
```

### 性能优化

- 按设备聚合：同一设备一次生成一张标注图
- 本地计算：避免重复调用相机接口
- 批量处理：一组点位一次性处理

## 温度异常处理

### 告警级别

```csharp
AlarmLevelEnum AlarmLevel = AlarmLevelEnum.Normal; // 默认正常级别
```

**注意：** 具体告警级别由告警模块根据温度阈值判断，采集任务统一设置为 `Normal`。

### 温度范围

- 有效温度范围：-40°C ~ 500°C
- 超出范围可能是设备故障或环境异常
- 建议配置温度阈值告警

## 相关文档

- [[IISAPIService]] - ISAPI 设备服务文档
- [[IRealtimeMonitoringPointService]] - 实时监测点位服务文档
- [[EnvironmentDetectionCollectionJob]] - 环境检测采集任务文档
- [[SensorEntity]] - 传感器实体文档

## 示例配置

### 完整配置示例

```json
{
  "InfraredTemperatureCollection": {
    "Enabled": true,
    "CronExpression": "*/3 * * * *"
  },
  "ISAPI": {
    "ConnectionTimeout": 10000,
    "ReadTimeout": 30000,
    "MaxRetries": 3
  }
}
```

### 高频采集配置

```json
{
  "InfraredTemperatureCollection": {
    "Enabled": true,
    "CronExpression": "*/1 * * * *"
  }
}
```

### 低频采集配置

```json
{
  "InfraredTemperatureCollection": {
    "Enabled": true,
    "CronExpression": "*/10 * * * *"
  }
}
```
