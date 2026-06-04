# SubstationUserEntity

## 概述

`SubstationUserEntity` 是用户与站点的多对多关系表，定义用户对特定站点的访问权限和默认站点设置。

## 表信息

- **表名**: `ast_substation_user`
- **主键**: `Id` (Guid)
- **继承**: `Entity<Guid>` + `IAuditedObject`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `Guid` | `id` | 主键 | PK |
| `SubstationId` | `Guid` | `substation_id` | 站点ID | FK, NOT NULL |
| `UserId` | `Guid` | `user_id` | 用户ID | FK, NOT NULL |
| `ExtraInfo` | `string?` | `extra_info` | 附加信息 | TEXT (JSON) |
| `IsDefault` | `bool` | `is_default` | 是否默认站点 | Default: false |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | |
| `CreatorId` | `Guid?` | `creator_id` | 创建者ID | FK (User) |
| `LastModificationTime` | `DateTime?` | `last_modification_time` | 最后修改时间 | |
| `LastModifierId` | `Guid?` | `last_modifier_id` | 最后修改者ID | FK (User) |

## 关联实体 (ER 关系)

### 导航属性

```csharp
// 关联站点
[Navigate(NavigateType.OneToOne, nameof(SubstationId))]
public SubstationAggregateRoot Substation { get; set; }

// 关联用户
[Navigate(NavigateType.OneToOne, nameof(UserId))]
public UserAggregateRoot User { get; set; }
```

### 关系图

```
UserAggregateRoot (N) ←──→ (M) SubstationUserEntity ←──→ (N) SubstationAggregateRoot
```

## 被哪些服务读写

### 写入服务

- `SubstationUserService` — 用户站点权限分配
- `UserService` — 用户创建时自动分配默认站点

### 读取服务

- `UserService` — 查询用户可访问的站点列表
- `SubstationService` — 查询站点下的授权用户
- `PermissionService` — 权限验证时检查用户站点访问权限

## 业务规则约束

### 1. 唯一性约束

```sql
-- 一个用户对一个站点只能有一条记录
CREATE UNIQUE INDEX UX_SUBSTATION_USER ON ast_substation_user(substation_id, user_id);
```

### 2. 默认站点约束

- 一个用户只能有一个默认站点 (`IsDefault = true`)
- 新分配默认站点时，需将旧默认站点的 `IsDefault` 设为 `false`

```sql
-- 确保每个用户只有一个默认站点
CREATE UNIQUE INDEX UX_SUBSTATION_USER_DEFAULT ON ast_substation_user(user_id) WHERE is_default = 1;
```

### 3. 级联删除

- 删除用户时，删除所有 `SubstationUserEntity` 记录
- 删除站点时，删除所有关联的 `SubstationUserEntity` 记录

### 4. 附加信息 (ExtraInfo)

`ExtraInfo` 存储用户站点相关配置：

```json
{
  "role": "operator",           // 用户角色
  "permissionLevel": 2,          // 权限等级
  "favoriteCameras": [           // 常用摄像机
    "camera_001",
    "camera_002"
  ],
  "notifications": {
    "alarmEnabled": true,        // 告警通知
    "reportEnabled": false       // 报告通知
  },
  "customView": {                // 自定义视图配置
    "dashboard": "compact",
    "theme": "dark"
  }
}
```

### 5. 审计要求

- 所有权限分配都必须记录审计信息
- `CreatorId` 和 `LastModifierId` 用于权限追溯

## 使用场景

### 1. 分配站点权限

```csharp
var substationUser = new SubstationUserEntity
{
    Id = Guid.NewGuid(),
    SubstationId = substationId,
    UserId = userId,
    IsDefault = false,
    ExtraInfo = "{\"role\":\"operator\",\"permissionLevel\":2}",
    CreationTime = DateTime.Now,
    CreatorId = adminUserId
};

await _substationUserRepo.InsertAsync(substationUser);
```

### 2. 设置默认站点

```csharp
// 1. 清除旧默认站点
var oldDefaults = await _substationUserRepo.GetListAsync(
    u => u.UserId == userId && u.IsDefault
);

foreach (var oldDefault in oldDefaults)
{
    oldDefault.IsDefault = false;
    oldDefault.LastModificationTime = DateTime.Now;
    oldDefault.LastModifierId = adminUserId;
}

await _substationUserRepo.UpdateAsync(oldDefaults);

// 2. 设置新默认站点
var newDefault = new SubstationUserEntity
{
    Id = Guid.NewGuid(),
    SubstationId = newSubstationId,
    UserId = userId,
    IsDefault = true,
    CreationTime = DateTime.Now,
    CreatorId = adminUserId
};

await _substationUserRepo.InsertAsync(newDefault);
```

### 3. 查询用户可访问的站点

```csharp
public async Task<List<SubstationAggregateRoot>> GetUserSubstationsAsync(Guid userId)
{
    var relations = await _substationUserRepo.GetListAsync(u => u.UserId == userId);
    var substationIds = relations.Select(r => r.SubstationId).ToList();

    return await _substationRepo.GetListAsync(s => substationIds.Contains(s.Id));
}
```

### 4. 权限验证

```csharp
public async Task<bool> HasAccessAsync(Guid userId, Guid substationId)
{
    return await _substationUserRepo.AnyAsync(
        u => u.UserId == userId && u.SubstationId == substationId
    );
}
```

## 索引建议

```sql
-- 唯一约束（用户 + 站点）
CREATE UNIQUE INDEX UX_SUBSTATION_USER ON ast_substation_user(substation_id, user_id);

-- 唯一约束（用户默认站点）
CREATE UNIQUE INDEX UX_SUBSTATION_USER_DEFAULT ON ast_substation_user(user_id) WHERE is_default = 1;

-- 查询索引（按用户查询）
CREATE INDEX IX_SUBSTATION_USER_USER ON ast_substation_user(user_id);

-- 查询索引（按站点查询）
CREATE INDEX IX_SUBSTATION_USER_SUBSTATION ON ast_substation_user(substation_id);
```

## 数据迁移示例

### 初始化管理员站点权限

```sql
INSERT INTO ast_substation_user (id, substation_id, user_id, is_default, creation_time, creator_id)
SELECT
    NEWID(),
    s.id,
    @AdminUserId,
    CASE WHEN ROW_NUMBER() OVER (ORDER BY s.id) = 1 THEN 1 ELSE 0 END,
    GETDATE(),
    @SystemUserId
FROM ast_substation s;
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Station/SubstationUserEntity.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/SubstationUserService.cs`
- **用户实体**: `Classes/Rbac/UserAggregateRoot.md`
- **站点实体**: `Classes/Substation/SubstationAggregateRoot.md`
