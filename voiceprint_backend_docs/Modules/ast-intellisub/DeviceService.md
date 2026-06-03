# 设备服务 (DeviceService)

## 概述
设备服务负责管理变电站中的设备信息，包括采集设备、传感器等，提供设备的增删改查、状态查询和统计功能。

## 职责
- 创建设备信息
- 更新设备配置
- 删除设备
- 查询设备列表和详情
- 获取设备状态
- 统计设备信息
- 管理设备显示状态
- 设备与网关的关联管理

## 主要接口

### 获取设备列表
```csharp
Task<PagedResultDto<DeviceDto>> GetListAsync(DeviceGetListInputDto input)
```

**请求参数**：
- `GatewayId`: 网关ID（可选）
- `DeviceId`: 设备ID（模糊查询，可选）
- `Name`: 设备名称（模糊查询，可选）
- `Manufacturer`: 制造商（可选）
- `Model`: 型号（可选）
- `SkipCount`: 跳过记录数
- `MaxResultCount`: 最大返回数量

### 获取单个设备
```csharp
Task<DeviceDto> GetAsync(string id)
```

### 创建设备
```csharp
Task<DeviceDto> CreateAsync(DeviceCreateUpdateDto input)
```

**请求参数**：
- `Id`: 设备ID（必填，手动指定）
- `GatewayId`: 网关ID（必填）
- `Name`: 设备名称（必填）
- `Manufacturer`: 制造商
- `Model`: 型号
- `Ip`: IP地址
- `Port`: 端口号
- `ChannelCount`: 通道数量
- `Password`: 密码

**验证规则**：
- 网关必须存在
- 设备ID必须唯一
- 通道数量必须大于0

### 更新设备
```csharp
Task<DeviceDto> UpdateAsync(string id, DeviceCreateUpdateDto input)
```

**验证规则**：
- 网关必须存在
- 设备ID保持不变

### 删除设备
```csharp
Task DeleteAsync(string id)
```

**级联操作**：
- 删除设备时会检查是否有关联传感器
- 删除设备时会清理关联数据

### 获取设备状态列表
```csharp
Task<List<DeviceStatusDto>> GetStatusListAsync(DeviceStatusListRequestDto input)
```

**请求参数**：
- `GatewayId`: 网关ID（可选）
- `SubstationId`: 变电站ID（可选）

**响应数据**：
- 设备在线状态
- 传感器状态信息
- 最后通信时间

### 获取网关下的设备列表
```csharp
Task<List<DeviceListDto>> GetDevicesByGatewayAsync(DeviceByGatewayRequestDto input)
```

**请求参数**：
- `GatewayId`: 网关ID（必填）

### 获取设备统计
```csharp
Task<DeviceStatisticsDto> GetStatisticsAsync(DeviceStatisticsRequestDto input)
```

**请求参数**：
- `SubstationId`: 变电站ID（必填）
- `GatewayId`: 网关ID（可选）

**统计数据**：
- 设备总数
- 在线设备数
- 离线设备数
- 设备类型分布

### 更新设备显示状态
```csharp
Task<DeviceDto> UpdateDisplayStatusAsync(string id, bool isDisplay)
```

**功能**：控制设备是否在界面上显示

## 设备状态

### 在线状态
- **在线**：设备正常通信
- **离线**：设备无法通信或超时

### 状态判断依据
- 最后通信时间
- 心跳检测
- 数据上报频率

## 数据处理流程

```
创建设备 → 验证网关存在 → 验证设备ID唯一 → 保存设备信息 → 关联传感器
```

### 设备创建流程
1. 验证网关是否存在
2. 验证设备ID唯一性
3. 验证通道数量有效性
4. 创建设备记录
5. 记录操作日志

### 设备状态查询流程
1. 根据网关或变电站查询设备
2. 查询设备关联的传感器
3. 获取设备最后通信时间
4. 判断设备在线状态
5. 返回状态列表

## 依赖服务

- [[GatewayService]] - 网关服务
- [[SensorService]] - 传感器服务
- [[SubstationService]] - 变电站服务
- [[SubstationPermissionChecker]] - 站点权限检查器

## 相关实体

- **DeviceEntity** - 设备实体
  - `Id`: 设备ID
  - `GatewayId`: 网关ID
  - `Name`: 设备名称
  - `Manufacturer`: 制造商
  - `Model`: 型号
  - `Ip`: IP地址
  - `Port`: 端口号
  - `ChannelCount`: 通道数量
  - `Password`: 密码
  - `IsDisplay`: 是否在界面显示
  - `CreationTime`: 创建时间

- **SensorEntity** - 传感器实体
  - `Id`: 传感器ID
  - `DeviceId`: 所属设备ID
  - `Name`: 传感器名称
  - `Type`: 传感器类型
  - `Status`: 传感器状态

## 配置项

无特定配置项，使用默认的数据库配置。

## 注意事项

- **设备ID唯一性**：设备ID在全局范围内必须唯一
- **网关关联**：设备必须关联到有效的网关
- **级联删除**：删除设备前需要检查和清理关联传感器
- **权限控制**：查询操作会执行站点权限检查
- **操作日志**：所有修改操作都会记录操作日志 `[OperLog]`
- **软删除**：设备使用软删除机制，不会从数据库中物理删除

## 设备类型

### 采集设备
- 音频采集设备
- 视频采集设备
- 温度采集设备
- 其他传感器设备

### 通道数量
- 单通道设备
- 多通道设备
- 通道数量影响并发采集能力

## 设备状态监控

### 在线检测
```csharp
// 基于最后通信时间判断在线状态
var isOnline = (DateTime.Now - device.LastCommunicationTime) < timeout;
```

### 心跳机制
- 定时发送心跳请求
- 检测设备响应
- 更新在线状态

## API 路径

- `POST /api/app/devices` - 创建设备
- `PUT /api/app/devices/{id}` - 更新设备
- `DELETE /api/app/devices/{id}` - 删除设备
- `GET /api/app/devices/{id}` - 获取单个设备
- `GET /api/app/devices` - 获取设备列表
- `GET /api/app/devices/status-list` - 获取设备状态列表
- `GET /api/app/devices/by-gateway` - 获取网关下的设备
- `GET /api/app/devices/statistics` - 获取设备统计
- `PUT /api/app/devices/{id}/display-status` - 更新显示状态

## 权限控制

- **读取权限**：用户只能访问已授权变电站的设备
- **修改权限**：需要设备管理权限
- **删除权限**：需要设备管理员权限

## 相关文档链接

- [[GatewayService]] - 网关服务
- [[SensorService]] - 传感器服务
- [[设备管理流程]] - 完整的设备管理流程
- [[数据采集架构]] - 设备数据采集架构说明
