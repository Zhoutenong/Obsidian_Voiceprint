# 变电站服务 (SubstationService)

## 概述
变电站服务负责管理变电站的基本信息和层级结构，实现站点的组织管理，支持分组和实际变电站的两级层次结构。

## 职责
- 创建变电站站点
- 更新变电站信息
- 删除变电站站点
- 查询变电站列表和详情
- 管理站点层级关系（父子关系）
- 验证站点类型和分组规则
- 权限检查和访问控制

## 主要接口

### 获取变电站列表
```csharp
Task<PagedResultDto<SubstationDto>> GetListAsync(SubstationGetListInputDto input)
```

**请求参数**：
- `Name`: 站点名称（模糊查询，可选）
- `SubstationTypeId`: 站点类型ID（可选）
- `ParentId`: 父站点ID（可选）
- `SkipCount`: 跳过记录数
- `MaxResultCount`: 最大返回数量

### 获取单个变电站
```csharp
Task<SubstationDto> GetAsync(Guid id)
```

### 创建变电站
```csharp
Task<SubstationDto> CreateAsync(SubstationCreateUpdateDto input)
```

**请求参数**：
- `Name`: 站点名称（必填，唯一）
- `SubstationTypeId`: 站点类型ID（必填）
- `ParentId`: 父站点ID（可选）
- `Location`: 位置信息
- `Description`: 描述信息
- `Longitude`: 经度
- `Latitude`: 纬度

**验证规则**：
- 站点名称必须唯一
- 站点类型必须存在且有效
- 如果指定父站点，父站点类型必须为分组类型（category=0）
- 分组类型的站点不能作为实际站点

### 更新变电站
```csharp
Task<SubstationDto> UpdateAsync(Guid id, SubstationCreateUpdateDto input)
```

**验证规则**：
- 站点名称唯一性（排除自身）
- 站点类型必须存在且有效
- 不能将站点设为自己的父级
- 父站点类型必须为分组类型
- 执行权限检查

### 删除变电站
```csharp
Task DeleteAsync(Guid id)
```

**级联操作**：
- 删除站点时会检查是否有子站点
- 删除站点时会检查是否有关联设备
- 删除站点时会清理用户站点关系

## 站点类型分类

| Category | 类型 | 说明 | 能否作为父站点 |
|----------|------|------|----------------|
| 0 | 分组 | 用于组织多个站点的逻辑分组 | 是 |
| 1 | 变电站 | 实际的变电站站点 | 否 |

## 数据处理流程

```
创建/更新站点 → 验证名称唯一性 → 验证站点类型 → 验证父站点关系 → 权限检查 → 保存数据
```

### 站点创建流程
1. 验证站点名称唯一性
2. 验证站点类型存在且有效
3. 如果有父站点，验证父站点存在且类型为分组类型
4. 创建站点记录
5. 记录操作日志

### 站点删除流程
1. 检查站点是否存在
2. 检查是否有子站点（不允许删除）
3. 检查是否有关联资源（设备、巡视任务等）
4. 执行删除操作
5. 清理关联数据

## 依赖服务

- [[SubstationTypeService]] - 站点类型服务
- [[SubstationUserService]] - 站点用户关系服务
- [[SubstationPermissionChecker]] - 站点权限检查器
- [[GatewayService]] - 网关服务（关联设备）
- [[PatrolTaskService]] - 巡视任务服务（关联任务）

## 相关实体

- **SubstationAggregateRoot** - 变电站聚合根
  - `Id`: 站点ID
  - `Name`: 站点名称
  - `SubstationTypeId`: 站点类型ID
  - `ParentId`: 父站点ID
  - `Location`: 位置信息
  - `Description`: 描述
  - `Longitude`: 经度
  - `Latitude`: 纬度
  - `CreationTime`: 创建时间

## 配置项

无特定配置项，使用默认的数据库配置。

## 注意事项

- **名称唯一性**：站点名称在全局范围内必须唯一
- **层级关系**：只有分组类型的站点才能作为父站点
- **权限控制**：更新和删除操作会执行权限检查
- **级联删除**：删除站点前需要检查和清理关联数据
- **操作日志**：所有修改操作都会记录操作日志 `[OperLog]`
- **软删除**：站点使用软删除机制，不会从数据库中物理删除

## API 路径

- `POST /api/app/substations` - 创建变电站
- `PUT /api/app/substations/{id}` - 更新变电站
- `DELETE /api/app/substations/{id}` - 删除变电站
- `GET /api/app/substations/{id}` - 获取单个变电站
- `GET /api/app/substations` - 获取变电站列表

## 权限控制

- **读取权限**：用户只能访问已授权的变电站
- **修改权限**：需要站点管理权限
- **删除权限**：需要站点管理员权限

## 相关文档链接

- [[SubstationUserService]] - 站点用户关系服务
- [[SubstationTypeService]] - 站点类型服务
- [[SubstationPermissionChecker]] - 站点权限检查器
- [[权限与访问控制]] - 完整的权限体系说明
