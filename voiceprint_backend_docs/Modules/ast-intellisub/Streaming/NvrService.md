# NVR服务 (NvrService)

## 概述

NVR服务（网络视频录像机服务）负责管理变电站监控系统中的硬盘录像机设备，提供NVR设备的增删改查、预置点管理和级联删除功能。

**位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/NvrService.cs`  
**层**：Application  
**模块**：ast-intellisub  
**依赖注入**：Scoped

## 职责

- 管理NVR设备信息（创建、更新、删除、查询）
- 级联删除NVR及其关联的摄像头、传感器、预置点等数据
- 通过ISAPI接口从NVR设备获取预置点列表
- 验证网关关联性
- 生成唯一8位设备ID
- 支持按名称、网关、状态、类型等多维度查询

## 主要接口

### 获取NVR列表

```csharp
Task<PagedResultDto<NvrDto>> GetListAsync(NvrGetListInputDto input)
```

**功能说明**：分页查询NVR设备列表，支持多条件过滤和排序

**请求参数**：
- `GatewayId`：网关ID（可选，用于过滤特定网关下的NVR）
- `Name`：设备名称（模糊查询，可选）
- `Status`：在线状态（可选，枚举值：Normal/Offline/Fault）
- `Type`：NVR类型（可选，枚举值：Dahua/Hikvision/Other）
- `HostAddress`：主机地址（模糊查询，可选）
- `Manufacturer`：设备厂家（模糊查询，可选）
- `IsVirtual`：是否虚拟设备（可选）
- `IncludeDeleted`：是否包含已删除记录（默认false）
- `Sorting`：排序规则（可选，默认按创建时间倒序）
- `SkipCount`：跳过记录数（分页用）
- `MaxResultCount`：最大返回数量（分页用）

**返回数据**：
```csharp
class PagedResultDto<NvrDto>
{
    int TotalCount              // 总记录数
    List<NvrDto> Items          // NVR列表
}

class NvrDto
{
    string Id                   // 8位设备ID
    Guid GatewayId              // 关联网关ID
    string Name                 // 设备名称
    DeviceStatusEnum Status     // 在线状态
    NvrTypeEnum Type           // NVR类型
    string HostAddress          // 主机地址
    int ChannelCount            // 通道数量
    string Manufacturer         // 设备厂家
    DateTime CreationTime       // 创建时间
    bool IsVirtual             // 是否虚拟设备
    bool IsDeleted             // 是否删除
    string Username             // 登录用户名
    string Password             // 登录密码
}
```

**业务逻辑**：
1. 根据过滤条件构建查询（默认排除已删除记录）
2. 应用排序规则（默认按创建时间倒序）
3. 执行分页查询
4. 返回分页结果

---

### 创建NVR设备

```csharp
[NonAction]
Task<NvrDto> CreateAsync(NvrCreateUpdateDto input)
```

**功能说明**：创建新的NVR设备记录，自动生成8位唯一ID

**请求参数**：
- `GatewayId`：网关ID（必填）
- `Name`：设备名称（可选，最大100字符）
- `Status`：在线状态（必填，枚举值）
- `Type`：NVR类型（必填，枚举值）
- `HostAddress`：主机地址（可选，最大200字符）
- `Username`：登录用户名（可选，最大100字符）
- `Password`：登录密码（可选，最大100字符）
- `ChannelCount`：通道数量（必填，不能为负数）
- `Manufacturer`：设备厂家（可选，最大100字符）
- `IsVirtual`：是否虚拟设备（可选）
- `IsDeleted`：是否删除（可选）

**验证规则**：
- 网关ID必须存在且有效
- 自动生成8位唯一ID（UUID前8位）
- 循环验证ID唯一性直到找到可用ID

**业务流程**：
```
1. 生成8位候选ID（GUID前8位）
   ↓
2. 验证ID唯一性
   ↓（如果ID已存在）
3. 重新生成ID并验证
   ↓
4. 验证网关存在性
   ↓
5. 创建NvrEntity实体
   ↓
6. 插入数据库
   ↓
7. 返回NvrDto
```

**调用链**：
```
Controller → NvrService.CreateAsync → SqlSugarRepository.InsertAsync
                ↓
         ValidateGatewayExistsAsync → GatewayRepository.AnyAsync
```

---

### 更新NVR设备

```csharp
Task<NvrDto> UpdateAsync(string id, NvrCreateUpdateDto input)
```

**功能说明**：更新指定NVR设备信息

**请求参数**：
- `id`：NVR设备ID（必填，8位字符串）
- `input`：更新数据（同CreateAsync参数）

**验证规则**：
- NVR设备必须存在
- 网关ID必须存在且有效

**业务流程**：
```
1. 根据ID查询NVR实体
   ↓
2. 验证设备存在性
   ↓（不存在时抛出异常）
3. 验证网关存在性
   ↓
4. 使用Mapster映射更新字段
   ↓
5. 更新数据库
   ↓
6. 返回更新后的NvrDto
```

**异常处理**：
- 设备不存在：抛出`UserFriendlyException("录像机不存在")`
- 网关不存在：抛出`UserFriendlyException($"网关 {gatewayId} 不存在")`

---

### 删除NVR设备

```csharp
Task DeleteAsync(string id)
```

**功能说明**：删除NVR设备（物理删除），有关联摄像头时禁止删除

**请求参数**：
- `id`：NVR设备ID（必填）

**业务规则**：
- 检查是否有关联的摄像头设备（`DeviceEntity.NvrId == id`）
- 有关联设备时抛出异常，不允许删除
- 无关联设备时执行物理删除

**异常处理**：
```csharp
if (hasDevices)
{
    throw new UserFriendlyException("录像机下还有关联的摄像头设备，无法删除");
}
```

**调用链**：
```
Controller → NvrService.DeleteAsync → DeviceRepository.AnyAsync（检查关联设备）
                ↓
         SqlSugarRepository.DeleteAsync（物理删除）
```

---

### 级联删除NVR及子数据

```csharp
Task DeleteNvrWithChildrenAsync(string nvrId)
```

**功能说明**：删除NVR及其所有关联的子结构数据，包括摄像头、传感器、预置点、绑定关系、监测对象关联等

**请求参数**：
- `nvrId`：NVR设备ID（必填）

**业务流程**（在事务中执行）：
```
1. 查询该NVR下所有设备ID列表
   ↓
2. 查询这些设备的预置点位传感器（type=1）的sensor_key
   ↓
3. 精准筛选需要删除的ast_point（同设备 && type=1 && sensor_key匹配）
   ↓
4. 删除与ast_point的绑定关系（PointBindingRelEntity）
   ↓
5. 删除筛选后的ast_point
   ↓
6. 获取要删除的传感器ID列表
   ↓
7. 删除监测对象预置点关联记录（MonitoredObjectPresetRelEntity）
   ↓
8. 删除所有传感器（AstSensorEntity）
   ↓
9. 删除所有设备（DeviceEntity）
   ↓
10. 删除NVR本身（NvrEntity）
```

**删除顺序**（防止外键约束冲突）：
1. 先删除绑定关系（多对多关联表）
2. 再删除监测对象预置点关联
3. 删除预置点数据（ast_point，type=1）
4. 删除传感器数据（ast_sensor）
5. 删除设备数据（ast_device）
6. 最后删除NVR本身

**事务保证**：
- 所有删除操作在一个数据库事务中执行
- 任一步骤失败则全部回滚
- 使用`UseTranAsync`确保原子性

**调用链**：
```
Controller → NvrService.DeleteNvrWithChildrenAsync
                ↓
         DeviceRepository._DbQueryable（查询设备）
                ↓
         SensorRepository._DbQueryable（查询传感器）
                ↓
         AstPointRepository._DbQueryable（查询预置点）
                ↓
         PointBindingRelRepository.DeleteAsync（删除绑定关系）
                ↓
         AstPointRepository.DeleteAsync（删除预置点）
                ↓
         MonitoredObjectPresetRelRepository.DeleteAsync（删除监测对象关联）
                ↓
         SensorRepository.DeleteAsync（删除传感器）
                ↓
         DeviceRepository.DeleteAsync（删除设备）
                ↓
         NvrRepository.DeleteAsync（删除NVR）
```

**关键SQL逻辑**：
```csharp
// 精准筛选预置点（type=1 且 sensor_key匹配）
var astPointIdsToDelete = await _astPointRepository._DbQueryable
    .Where(p => deviceIds.Contains(p.DeviceId)
                && (int)p.Type == 1
                && presetSensorKeys.Contains(p.SensorKey))
    .Select(p => p.Id)
    .ToListAsync();
```

---

### 获取默认NVR

```csharp
private Task<NvrEntity> GetDefaultNvrAsync()
```

**功能说明**：获取系统默认的NVR设备（最新创建且状态正常的NVR）

**业务规则**：
- 状态必须为`Normal`（正常）
- 未被删除（`IsDeleted == false`）
- 按创建时间倒序排列，取第一条（最新的）

**查询条件**：
```csharp
.Where(x => !x.IsDeleted && x.Status == DeviceStatusEnum.Normal)
.OrderBy(x => x.CreationTime, OrderByType.Desc)
.FirstAsync()
```

**异常处理**：
- 未找到可用NVR时抛出：`UserFriendlyException("未找到可用的NVR设备")`

---

### 从NVR获取预置点列表

```csharp
Task<PTZPresetList> GetPresetListFromNvrAsync(int channelId)
```

**功能说明**：通过NVR设备的ISAPI接口获取指定通道的PTZ预置点列表

**请求参数**：
- `channelId`：摄像机通道ID（必填）

**业务流程**：
```
1. 获取默认NVR设备
   ↓
2. 创建带认证的HttpClient
   ↓
3. 构建ISAPI请求URL
   ↓
4. 发送GET请求到NVR
   ↓
5. 反序列化XML响应
   ↓
6. 返回PTZPresetList对象
```

**ISAPI接口**：
```
URL格式: http://{NvrHostAddress}/ISAPI/PTZCtrl/channels/{channelId}/presets
认证方式: HTTP Basic Authentication (Username + Password)
超时时间: 30秒
```

**异常处理**：
- 请求失败时返回`null`（静默失败）
- 使用`EnsureSuccessStatusCode()`确保HTTP成功状态

**调用链**：
```
Controller → NvrService.GetPresetListFromNvrAsync
                ↓
         GetDefaultNvrAsync（获取默认NVR）
                ↓
         HttpClient.GetAsync（发送ISAPI请求）
                ↓
         XmlSerializer.Deserialize（解析XML响应）
```

**依赖类型**：
```csharp
class PTZPresetList
{
    // 预置点列表（从ISAPI响应反序列化）
}
```

---

## 依赖服务

### 仓储层
- `[[ISqlSugarRepository<NvrEntity, string>]]` - NVR实体仓储
- `[[ISqlSugarRepository<DeviceEntity, string>]]` - 设备实体仓储
- `[[ISqlSugarRepository<GatewayAggregateRoot, Guid>]]` - 网关实体仓储
- `[[ISqlSugarRepository<SensorEntity, Guid>]]` - 传感器实体仓储
- `[[ISqlSugarRepository<AstPointEntity, Guid>]]` - 预置点实体仓储
- `[[ISqlSugarRepository<PointBindingRelEntity, Guid>]]` - 点位绑定关系仓储
- `[[ISqlSugarRepository<MonitoredObjectPresetRelEntity, Guid>]]` - 监测对象预置点关联仓储

### 基础设施
- `[[ILogger<NvrService>]]` - 日志记录
- `[[YiCrudAppService<NvrEntity, NvrDto, string, NvrGetListInputDto, NvrCreateUpdateDto>]]` - CRUD基类

### HTTP客户端
- `[[HttpClient]]` - 用于调用NVR的ISAPI接口

---

## 数据实体

### NvrEntity

**表名**：`ast_nvr`  
**主键**：`Id`（string，8位）

**主要字段**：
```csharp
class NvrEntity : Entity<string>
{
    string Id                   // 主键（8位）
    Guid GatewayId              // 网关ID（外键）
    string Name                 // 设备名称
    DeviceStatusEnum Status     // 在线状态
    NvrTypeEnum Type            // NVR类型
    string HostAddress          // 主机地址
    int ChannelCount            // 通道数量
    string Manufacturer         // 设备厂家
    DateTime CreationTime       // 创建时间
    bool IsVirtual             // 是否虚拟设备
    bool IsDeleted             // 是否删除
    string Username             // 登录用户名
    string Password             // 登录密码
}
```

**枚举类型**：
```csharp
enum DeviceStatusEnum
{
    Normal,     // 正常
    Offline,    // 离线
    Fault       // 故障
}

enum NvrTypeEnum
{
    Dahua,      // 大华
    Hikvision,  // 海康
    Other       // 其他
}
```

---

## 业务规则

### ID生成规则
- 使用GUID的前8位作为设备ID
- 循环验证直到找到唯一ID
- 确保不与现有设备冲突

### 级联删除规则
- `DeleteAsync`：有关联设备时禁止删除
- `DeleteNvrWithChildrenAsync`：强制删除所有关联数据
- 删除顺序：绑定关系 → 监测对象关联 → 预置点 → 传感器 → 设备 → NVR

### 网关关联规则
- 所有NVR必须关联到有效网关
- 创建和更新时验证网关存在性
- 不允许关联到不存在的网关

### 预置点筛选规则
- 只删除`type=1`的预置点（PTZ预置点位）
- 通过`sensor_key`精准匹配
- 避免误删除其他类型的点位数据

---

## 配置说明

### ISAPI接口配置
NVR设备需要支持ISAPI接口：
- 协议：HTTP
- 认证：Basic Authentication
- 超时：30秒
- 路径：`/ISAPI/PTZCtrl/channels/{channelId}/presets`

---

## 相关文档

- [[Modules/ast-intellisub/Camera/CameraService]] - 摄像机设备管理
- [[Modules/ast-intellisub/DeviceService]] - 设备管理服务
- [[Modules/ast-intellisub/Gateway/GatewayService]] - 网关管理服务
- [[Modules/ast-intellisub/Camera/PresetService]] - 预置点管理服务
- [[Modules/ast-intellisub/Monitoring/MonitoredObjectService]] - 监测对象管理
- [[Modules/ast-intellisub/Gateway/StreamingGatewayService]] - 流媒体网关服务

---

## API路由

```
GET    /api/app/ast-intellisub/nvr                 # 获取列表
GET    /api/app/ast-intellisub/nvr/{id}             # 获取详情
POST   /api/app/ast-intellisub/nvr                 # 创建NVR
PUT    /api/app/ast-intellisub/nvr/{id}             # 更新NVR
DELETE /api/app/ast-intellisub/nvr/{id}             # 删除NVR
POST   /api/app/ast-intellisub/nvr/delete-with-children  # 级联删除
GET    /api/app/ast-intellisub/nvr/preset-list/{channelId}  # 获取预置点列表
```

---

## 权限要求

- 所有接口需要认证：`[Authorize]`
- 建议配置细粒度权限控制：
  - `ast-intellisub.nvr.create` - 创建NVR
  - `ast-intellisub.nvr.update` - 更新NVR
  - `ast-intellisub.nvr.delete` - 删除NVR
  - `ast-intellisub.nvr.read` - 查询NVR

---

## 日志记录

### 信息日志
- 创建成功：`"创建录像机成功: {Id} - {Name}"`
- 更新成功：`"更新录像机成功: {Id} - {Name}"`
- 删除成功：`"删除录像机成功: {Id}"`
- 级联删除成功：`"已删除 NVR 及下属数据(含筛选 ast_point 和监测对象预置点关联): {NvrId}"`
- 网关验证：`"验证网关存在性: {GatewayId}"`

### 异常日志
- 所有异常由ABP框架自动记录
- 业务异常使用`UserFriendlyException`

---

## 注意事项

⚠️ **重要**：
1. 级联删除操作不可逆，谨慎使用
2. 预置点查询依赖NVR设备的ISAPI接口支持
3. 删除前确保不再需要相关数据
4. ID生成使用GUID前8位，理论上可能有碰撞，通过循环验证解决
5. 事务中操作多个表，注意性能影响

---

**最后更新**：2026-06-03  
**版本**：v1.0.0
