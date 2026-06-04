# TenantAggregateRoot — 租户聚合根

## 基本信息

- **实体名称**：`TenantAggregateRoot`
- **数据库表**：`YiTenant`
- **模块位置**：`module/tenant-management/Yi.Framework.TenantManagement.Domain/`
- **继承关系**：`FullAuditedAggregateRoot<Guid>` → `IHasEntityVersion`

## 实体说明

租户聚合根是多租户系统的核心实体，每个租户代表一个独立的业务实例。支持租户名称、连接字符串、数据库类型等配置，实现多租户数据隔离。

## 字段说明

### 主键与基本信息
- `Id` (Guid) - 主键
- `Name` (string) - 租户名称（不可为空，最大长度限制）
- `EntityVersion` (int) - 实体版本号（用于并发控制）

### 数据库配置
- `TenantConnectionString` (string) - 租户连接字符串
- `DbType` (DbType) - 数据库类型枚举

### 扩展属性
- `ExtraProperties` (ExtraPropertyDictionary) - 扩展属性字典

## 枚举类型
### DbType — 数据库类型
- SqlServer
- MySQL
- PostgreSQL
- Oracle
- Sqlite
- 其他数据库类型

## 业务规则
1. **名称唯一**：租户名称在系统中必须唯一
2. **连接配置**：每个租户有独立的数据库连接字符串
3. **版本控制**：EntityVersion用于乐观并发控制
4. **多租户隔离**：所有业务数据按租户隔离存储

## 数据隔离机制
- **数据库隔离**：每个租户使用独立的数据库或Schema
- **连接字符串**：TenantConnectionString指向租户专属数据库
- **查询过滤**：所有数据查询自动过滤当前租户的数据

## 相关实体
- 所有业务实体都通过租户ID进行数据隔离

## 相关服务
- TenantService - 租户管理服务
- TenantConnectionService - 租户连接服务

## 数据查询示例
```sql
-- 查询所有租户
SELECT id, name, db_type, entity_version FROM YiTenant;

-- 查询租户连接信息
SELECT id, name, tenant_connection_string, db_type 
FROM YiTenant 
WHERE is_deleted = 0;
```

## 业务价值
- 实现SaaS多租户架构
- 提供租户级别的数据隔离
- 支持租户的创建、配置和管理
- 便于系统扩展和商业化部署

## 设计模式
- **多租户模式**：每个租户是独立的业务实例
- **数据隔离**：通过租户ID确保数据安全隔离
- **配置独立**：每个租户可以有独立的系统配置

---

> **最后更新**：2026-06-04
> **源码位置**：`module/tenant-management/Yi.Framework.TenantManagement.Domain/TenantAggregateRoot.cs`
