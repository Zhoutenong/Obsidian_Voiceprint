# 巡检执行服务 (PatrolExecutionService)

## 概述
巡检执行服务负责执行巡检任务，协调摄像机、传感器等资源完成数据采集。

## 职责
- 执行巡检任务
- 协调摄像机云台控制
- 触发传感器数据采集
- 监控执行进度
- 处理执行异常

## 主要接口

### 执行巡检任务
```csharp
Task<PatrolExecutionResultDto> ExecuteAsync(Guid taskId)
```

**执行流程**：
1. 接收巡检任务
2. 遍历巡检路线点位
3. 控制摄像机云台到预置位
4. 触发图像采集
5. 触发传感器数据采集
6. 保存巡检记录
7. 更新任务状态

### 获取执行状态
```csharp
Task<PatrolExecutionStatusDto> GetStatusAsync(Guid executionId)
```

### 取消执行
```csharp
Task CancelAsync(Guid executionId)
```

## 执行流程

### 完整流程

```
接收任务 → 加载路线 → 遍历点位 → 云台控制 → 图像采集 → 传感器采集 → 保存记录 → 更新状态
```

### 详细步骤

1. **任务准备**
   - 加载巡检任务信息
   - 加载巡检路线配置
   - 检查设备状态
   - 初始化执行上下文

2. **点位遍历**
   - 获取路线上的所有点位
   - 按顺序处理每个点位
   - 记录点位执行状态

3. **云台控制**
   - 调用 [[CameraService]] 控制云台
   - 转动到指定预置位
   - 等待云台稳定

4. **图像采集**
   - 触发摄像机抓拍
   - 下载图像到服务器
   - 保存图像路径

5. **传感器采集**
   - 触发传感器数据读取
   - 记录点位数值
   - 保存到数据库

6. **异常处理**
   - 记录失败点位
   - 继续执行后续点位
   - 标记任务部分完成

7. **任务完成**
   - 统计执行结果
   - 更新任务状态
   - 生成执行报告

## 执行状态

| 状态 | 说明 |
|-----|------|
| Pending | 待执行 |
| Running | 执行中 |
| Completed | 已完成 |
| Failed | 失败 |
| Cancelled | 已取消 |
| PartialCompleted | 部分完成 |

## 数据处理

### 采集数据类型

- **图像数据**
  - 可见光图像
  - 红外图像
  - 图像元数据

- **传感器数据**
  - 振动数据
  - 温度数据
  - 噪声数据
  - 环境数据

### 数据保存

- 图像保存到文件存储
- 传感器数据保存到数据库
- 创建巡检记录关联
- 更新点位最新数据

## 异常处理

### 摄像机异常
- 连接失败 → 跳过该点位，记录错误
- 云台控制失败 → 跳过该点位，记录错误
- 图像下载失败 → 跳过该点位，记录错误

### 传感器异常
- 数据读取失败 → 跳过该点位，记录错误
- 数据超时 → 跳过该点位，记录错误

### 系统异常
- 任务中断 → 记录中断位置，支持断点续传
- 系统重启 → 恢复未完成任务

## 依赖服务

- [[PatrolTaskService]] - 巡检任务服务
- [[CameraService]] - 摄像机服务
- [[SensorService]] - 传感器服务
- [[PatrolRecordService]] - 巡检记录服务
- [[MonitoredPointService]] - 监测点位服务

## 相关实体

- [[PatrolTask]] - 巡检任务
- [[PatrolRoute]] - 巡检路线
- [[PatrolPoint]] - 巡检点位
- [[PatrolRecord]] - 巡检记录
- [[Device]] - 设备

## 配置项

```json
{
  "PatrolExecution": {
    "CameraStabilizationDelay": 2000,
    "ImageCaptureTimeout": 10000,
    "SensorReadTimeout": 5000,
    "MaxRetryCount": 3,
    "ContinueOnError": true
  }
}
```

## 相关文档

- [[PatrolTaskService]] - 巡检任务服务文档
- [[PatrolRecordService]] - 巡检记录服务文档
- [[CameraService]] - 摄像机服务文档
- [[SensorService]] - 传感器服务文档

## 注意事项

1. **云台控制延迟**：云台转动需要时间，需要等待稳定后再采集
2. **图像传输时间**：图像下载可能耗时较长，需要设置合理超时
3. **并发控制**：同一设备不支持并发巡检，需要排队执行
4. **错误恢复**：单个点位失败不影响其他点位执行
5. **断点续传**：支持中断后从失败点位继续执行
