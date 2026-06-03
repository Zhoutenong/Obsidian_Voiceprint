# 变电站用户服务 (SubstationUserService)

## 概述
变电站用户服务负责管理用户与变电站之间的多对多关联关系，支持用户的默认站点配置和站点访问权限管理。

## 职责
- 创建用户与站点的关联关系
- 更新用户站点关系
- 删除用户站点关系
- 查询用户的站点列表
- 管理用户的默认站点
- 验证用户站点关系唯一性
- 处理默认站点切换逻辑

## 主要接口

### 获取用户站点关系列表
```csharp
Task<PagedResultDto<SubstationUserDto>> GetListAsync(SubstationUserGetListInputDto input)
```

**请求参数**：
- `UserId`: 用户ID（可选）
- `SubstationId`: 站点ID（可选）
- `IsDefault`: 是否默认站点（可选）
- `SkipCount`: 跳过记录数
- `MaxResultCount`: 最大返回数量

### 获取单个用户站点关系
```csharp
Task<SubstationUserDto> GetAsync(Guid id)
```

### 创建用户站点关系
```csharp
Task<SubstationUserDto> CreateAsync(SubstationUserCreateUpdateDto input)
```

**请求参数**：
- `UserId`: 用户ID（必填）
- `SubstationId`: 站点ID（必填）
- `IsDefault`: 是否默认站点（必填）

**验证规则**：
- 站点必须存在
- 用户-站点关系必须唯一
- 如果设置为默认站点，会自动取消其他默认站点

### 更新用户站点关系
```csharp
Task<SubstationUserDto> UpdateAsync(Guid id, SubstationUserCreateUpdateDto input)
```

**特殊处理**：
- 如果 `IsDefault` 从 `false` 改为 `true`，会自动取消其他默认站点
- 如果 `IsDefault` 从 `true` 改为 `false`，需要处理默认站点缺失情况

### 删除用户站点关系
```csharp
Task DeleteAsync(Guid id)
```

**特殊处理**：
- 删除默认站点时，会自动将其他站点设为默认
- 如果没有其他站点，则允许删除

## 默认站点处理机制

### 创建时的默认站点处理
```csharp
// 如果已有默认站点且当前设置也为默认，则处理默认站点
if (input.IsDefault && hasDefaultStation)
{
    // 将其他默认站点改为非默认
    await _repository._DbQueryable
        .Where(x => x.UserId == input.UserId && x.IsDefault)
        .ToStorageAsync()
        .SetUpdate(x => x.IsDefault, false)
        .ExecuteAsync();
}
```

### 更新时的默认站点处理
```csharp
// 如果设置为默认站点
if (input.IsDefault)
{
    // 将当前用户的其他默认站点设为非默认
    await HandleDefaultStationAsync(input.UserId, id);
}
```

### 删除时的默认站点处理
```csharp
// 如果删除的是默认站点
if (existingRelation.IsDefault)
{
    // 查找该用户的其他站点
    var otherStation = await _repository._DbQueryable
        .Where(x => x.UserId == existingRelation.UserId && x.Id != id)
        .FirstAsync();

    // 将其他站点设为默认
    if (otherStation != null)
    {
        otherStation.IsDefault = true;
        await _repository.UpdateAsync(otherStation);
    }
}
```

## 数据处理流程

```
创建关系 → 验证站点存在 → 验证唯一性 → 处理默认站点 → 保存关系
```

### 创建流程详细步骤
1. 验证站点是否存在
2. 检查用户-站点关系是否已存在
3. 查询用户是否已有默认站点
4. 如果设置为默认站点，取消其他默认站点
5. 创建用户站点关系
6. 记录操作日志

### 删除流程详细步骤
1. 查询现有的用户站点关系
2. 如果删除的是默认站点，需要处理默认站点切换
3. 查找用户的其他站点关系
4. 将其他站点设为默认（如果有）
5. 删除指定的用户站点关系
6. 记录操作日志

## 依赖服务

- [[SubstationService]] - 变电站服务
- [[UserService]] - 用户服务（来自 RBAC 模块）
- [[SubstationPermissionChecker]] - 站点权限检查器

## 相关实体

- **SubstationUserEntity** - 用户站点关系实体
  - `Id`: 关系ID
  - `UserId`: 用户ID
  - `SubstationId`: 站点ID
  - `IsDefault`: 是否默认站点
  - `CreationTime`: 创建时间

## 配置项

无特定配置项，使用默认的数据库配置。

## 注意事项

- **唯一性约束**：同一用户与站点的关联关系必须唯一
- **默认站点规则**：
  - 一个用户可以有多个站点关联
  - 每个用户有且只有一个默认站点
  - 删除默认站点时会自动切换到其他站点
  - 最后一个站点被删除时允许无默认站点
- **权限要求**：所有接口需要授权访问 `[Authorize]`
- **操作日志**：创建和更新操作会记录操作日志 `[OperLog]`
- **级联操作**：删除站点时会同步清理用户站点关系

## 业务规则

### 默认站点切换规则
1. 创建新的默认站点时，自动取消旧的默认站点
2. 更新关系为默认站点时，自动取消其他默认站点
3. 删除默认站点时，自动将其他站点设为默认
4. 删除唯一的站点时，允许无默认站点状态

### 用户访问控制
- 用户只能访问已关联的变电站
- 默认站点用于用户登录时的默认视图
- 权限检查基于用户的站点关联关系

## API 路径

- `POST /api/app/substation-users` - 创建用户站点关系
- `PUT /api/app/substation-users/{id}` - 更新用户站点关系
- `DELETE /api/app/substation-users/{id}` - 删除用户站点关系
- `GET /api/app/substation-users/{id}` - 获取单个关系
- `GET /api/app/substation-users` - 获取用户站点关系列表

## 相关文档链接

- [[SubstationService]] - 变电站服务
- [[SubstationPermissionChecker]] - 站点权限检查器
- [[权限与访问控制]] - 完整的权限体系说明
