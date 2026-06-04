# ConfigAggregateRoot — 系统配置聚合根

## 基本信息

- **实体名称**：`ConfigAggregateRoot`
- **数据库表**：`Config`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Entities/`
- **继承关系**：`AggregateRoot<Guid>` → `IAuditedObject` → `IOrderNum` → `ISoftDelete`

## 实体说明

系统配置聚合根用于存储系统的键值对配置信息。支持配置分类、排序、软删除等功能，提供灵活的系统配置管理。

## 字段说明

### 主键与状态
- `Id` (Guid) - 主键
- `IsDeleted` (bool) - 软删除标记

### 配置信息
- `ConfigName` (string) - 配置名称，用于显示
- `ConfigKey` (string) - 配置键，用于代码引用
- `ConfigValue` (string) - 配置值
- `ConfigType` (string?) - 配置类别

### 排序与说明
- `OrderNum` (int) - 排序字段，控制显示顺序
- `Remark` (string?) - 配置说明

### 审计信息
- `CreationTime` (DateTime) - 创建时间
- `CreatorId` (Guid?) - 创建者ID
- `LastModificationTime` (DateTime?) - 最后修改时间
- `LastModifierId` (Guid?) - 最后修改者ID

## 业务规则
1. **键值唯一**：ConfigKey在整个系统中必须唯一
2. **分类管理**：通过ConfigType对配置进行分类
3. **排序支持**：OrderNum控制配置显示顺序
4. **软删除**：删除配置仅标记IsDeleted

## 相关服务
- ConfigService - 配置管理服务

## 数据查询示例
```sql
-- 查询所有启用的配置
SELECT * FROM Config WHERE is_deleted = 0 ORDER BY order_num;

-- 按类型查询配置
SELECT * FROM Config WHERE config_type = '系统' AND is_deleted = 0;
```

## 业务价值
- 提供灵活的系统配置管理
- 支持配置的动态修改和热更新
- 配置分类便于管理和查找

---
> **最后更新**：2026-06-04
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Entities/ConfigAggregateRoot.cs`
