# AuthAggregateRoot — 第三方授权聚合根

## 基本信息

- **实体名称**：`AuthAggregateRoot`
- **数据库表**：`Auth`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Authorization/`
- **继承关系**：`AggregateRoot<Guid>` → `ISoftDelete` → `IHasCreationTime`

## 实体说明

第三方授权聚合根用于存储用户的第三方登录授权信息，如OAuth登录（QQ、Gitee等）。记录授权类型、OpenID、用户关联等，支持多种第三方登录方式。

## 字段说明

### 主键与状态
- `Id` (Guid) - 主键
- `IsDeleted` (bool) - 软删除标记

### 授权信息
- `AuthType` (string) - 授权类型，如"qq"、"gitee"等
- `OpenId` (string) - 第三方平台返回的用户唯一标识
- `Name` (string) - 第三方用户昵称

### 用户关联
- `UserId` (Guid) - 关联的系统用户ID

### 审计信息
- `CreationTime` (DateTime) - 创建时间

## 业务规则
1. **唯一性**：同一AuthType + OpenId 只能有一个授权记录
2. **关联用户**：通过UserId关联到系统用户
3. **软删除**：删除授权仅标记IsDeleted，不物理删除
4. **多平台支持**：支持用户通过多个平台登录

## 相关实体
- UserAggregateRoot - 用户实体

## 相关服务
- AuthService - 第三方登录授权服务
- AccountService - 账号服务

## 数据查询示例
```sql
-- 查询用户的所有第三方授权
SELECT * FROM Auth WHERE user_id = 'xxx' AND is_deleted = 0;

-- 查询某类型的授权
SELECT * FROM Auth WHERE auth_type = 'qq' AND is_deleted = 0;
```

## 业务价值
- 支持多种第三方登录方式
- 管化用户注册和登录流程
- 提供用户账号绑定功能

---
> **最后更新**：2026-06-04
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Authorization/AuthAggregateRoot.cs`