# 监测点位服务 (MonitoredPointService)

## 概述
监测点位服务管理变电站设备上的所有监测点位配置，是数据采集和告警的基础。

## 职责
- 管理监测点位信息
- 关联监测对象和传感器
- 配置告警规则
- 管理点位状态
- 提供点位查询服务

## 主要接口

### 创建监测点位
```csharp
Task<MonitoredPointDto> CreateAsync(CreateMonitoredPointDto input)
```

**请求参数**：
- `Name`: 点位名称
- `Code`: 点位编码（唯一）
- `DeviceId`: 所属设备ID
- `PointType`: 点位类型
- `SensorType`: 传感器类型
- `Unit`: 单位
- `MinValue`: 最小值
- `MaxValue`: 最大值
- `AlarmRuleIds`: 告警规则ID列表

### 更新监测点位
```csharp
Task<MonitoredPointDto> UpdateAsync(Guid id, UpdateMonitoredPointDto input)
```

### 删除监测点位
```csharp
Task DeleteAsync(Guid id)
```

### 获取监测点位列表
```csharp
Task<PagedResultDto<MonitoredPointDto>> GetListAsync(GetMonitoredPointListDto input)
```

**查询参数**：
- `DeviceId`: 设备ID
- `PointType`: 点位类型
- `Status`: 状态
- `Keyword`: 关键字（名称/编码）
- `PageIndex`: 页码
- `PageSize`: 每页大小

### 按设备获取点位
```csharp
Task<List<MonitoredPointDto>> GetByDeviceIdAsync(Guid deviceId)
```

### 获取点位详情
```csharp
Task<MonitoredPointDetailDto> GetDetailAsync(Guid id)
```

## 点位类型

| 类型 | 说明 | 数据类型 |
|-----|------|---------|
| Vibration | 振动监测 | Double |
| Temperature | 温度监测 | Double |
| Noise | 噪声监测 | Double |
| Infrared | 红外监测 | Double |
| Humidity | 湿度监测 | Double |
| Pressure | 压力监测 | Double |
| Current | 电流监测 | Double |
| Voltage | 电压监测 | Double |

## 传感器类型

| 类型 | 说明 | 型号示例 |
|-----|------|---------|
| Accelerometer | 加速度传感器 | PCB 352C33 |
| Thermocouple | 热电偶 | Type K |
| RTD | 热电阻 | PT100 |
| Microphone | 麦克风 | BSWA MPA201 |
| ThermalCamera | 红外热像仪 | FLIR A700 |
| Hygrometer | 温湿度传感器 | Sensirion SHT85 |

## 数据结构

### 监测点位 (MonitoredPoint)

| 字段 | 类型 | 说明 |
|-----|------|------|
| Id | Guid | 点位ID |
| Name | string | 点位名称 |
| Code | string | 点位编码（唯一） |
| DeviceId | Guid | 所属设备ID |
| PointType | PointType | 点位类型 |
| SensorType | SensorType | 传感器类型 |
| Unit | string | 单位 |
| MinValue | double? | 最小值 |
| MaxValue | double? | 最大值 |
| AlarmRuleIds | List<Guid> | 告警规则ID列表 |
| Status | PointStatus | 状态 |
| Sort | int | 排序 |
| Remark | string | 备注 |

### 点位状态

| 状态 | 说明 |
|-----|------|
| Active | 激活 |
| Inactive | 未激活 |
| Fault | 故障 |
| Maintenance | 维护中 |

## 告警配置

### 告警规则关联
- 一个点位可以关联多个告警规则
- 告警规则定义触发条件
- 支持多级告警（警告、严重、危急）

### 告警类型

| 告警类型 | 触发条件 |
|---------|---------|
| 上限告警 | 当前值 > 阈值上限 |
| 下限告警 | 当前值 < 阈值下限 |
| 变化率告警 | 变化率超过阈值 |
| 持续告警 | 值持续异常超过时间 |
| 设备告警 | 传感器故障 |

## 数据处理

### 点位数据采集
1. 接收传感器数据
2. 验证数据有效性
3. 转换数据单位
4. 保存到数据库
5. 检查告警条件
6. 触发告警（如需要）

### 数据验证
- 检查值范围（MinValue ~ MaxValue）
- 检查变化率合理性
- 检查时间戳有效性
- 检查传感器状态

### 数据转换
- ADC 值转物理值
- 单位转换
- 温度补偿
- 校正系数应用

## 异常处理

- `BusinessException`: 点位编码重复
- `BusinessException`: 设备不存在
- `BusinessException`: 点位有关联数据不能删除
- `BusinessException`: 告警规则不存在

## 依赖服务

- [[DeviceService]] - 设备服务
- [[AlarmCategoryService]] - 告警分类服务
- [[MonitoredPointAlarmCategoryRelService]] - 点位告警分类关联服务
- [[PointDataService]] - 点位数据服务

## 相关实体

- [[MonitoredPoint]] - 监测点位实体
- [[Device]] - 设备
- [[AlarmCategory]] - 告警分类
- [[PointData]] - 点位数据

## 配置项

```json
{
  "MonitoredPoint": {
    "MaxRetryCount": 3,
    "DataValidation": true,
    "AutoCalibration": false,
    "DefaultValue": 0
  }
}
```

## 相关文档

- [[RealtimeMonitoringPointService]] - 实时监测点位服务
- [[PointDataService]] - 点位数据服务
- [[AlarmCategoryService]] - 告警分类服务
- [[Device]] - 设备实体

## API 路径

- `POST /api/app/monitored-points` - 创建监测点位
- `PUT /api/app/monitored-points/{id}` - 更新监测点位
- `DELETE /api/app/monitored-points/{id}` - 删除监测点位
- `GET /api/app/monitored-points` - 获取监测点位列表
- `GET /api/app/monitored-points/{id}` - 获取点位详情
- `GET /api/app/monitored-points/by-device/{deviceId}` - 按设备获取点位
