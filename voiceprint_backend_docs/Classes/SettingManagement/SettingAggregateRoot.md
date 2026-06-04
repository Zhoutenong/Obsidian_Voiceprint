# SettingAggregateRoot — 系统设置聚合根

## 基本信息

- **实体名称**：`SettingAggregateRoot`
- **数据库表**：`Setting`
- **模块位置**：`module/setting-management/Yi.Framework.SettingManagement.Domain/`
- **继承关系**：`Entity<Guid>` → `IAggregateRoot<Guid>`

## 实体说明

系统设置聚合根用于存储应用程序的键值对配置信息。支持提供者（Provider）模式，允许不同提供者管理相同的设置项。

## 字段说明

### 主键与设置信息
- `Id` (Guid) - 主键
- `Name` (string) - 设置名称（不可为空）
- `Value` (string) - 设置值（不可为空）

### 提供者信息
- `ProviderName` (string?) - 提供者名称（可选）
- `ProviderKey` (string?) - 提供者键值（可选）

## 业务规则
1. **键值唯一**：Name字段在整个系统中必须唯一
2. **值不可空**：Value字段必须有值
3. **提供者模式**：支持通过ProviderName和ProviderKey实现不同的设置提供者
4. **聚合根**：虽然继承Entity而非AggregateRoot，但在业务逻辑中作为聚合根使用

## 提供者模式
- 默认提供者：直接使用Value字段
- 自定义提供者：通过ProviderName和ProviderKey从其他来源获取值

## 相关服务
- SettingService - 系统设置服务
- SettingGroupService - 设置组服务

## 数据查询示例
```sql
-- 查询所有设置
SELECT name, value, provider_name FROM Setting;

-- 查询特定提供者的设置
SELECT * FROM Setting WHERE provider_name = 'Database' AND provider_key = 'default';

-- 按名称前缀查询
SELECT * FROM Setting WHERE name LIKE 'email.%';
```

## 业务价值
- 提供灵活的系统配置管理
- 支持配置的分层组织
- 允许运行时修改配置而无需重启
- 支持多种配置来源（数据库、环境变量、配置文件等）

---

> **最后更新**：2026-06-04
> **源码位置**：`module/setting-management/Yi.Framework.SettingManagement.Domain/SettingAggregateRoot.cs`
