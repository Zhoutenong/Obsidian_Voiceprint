# 多租户模块概述

## 概述

多租户模块（tenant-management）提供完整的 SaaS 多租户支持，允许单个应用实例为多个租户（组织/公司）提供服务，每个租户拥有独立的用户、数据和配置。

**模块路径**：`module/tenant-management/`

## 功能说明

### 租户隔离
- 数据库级别隔离（可选）
- 应用级别隔离（默认）
- 租户上下文自动切换
- 跨租户操作支持

### 数据隔离策略
系统支持两种数据隔离模式：

1. **独立数据库模式**（高隔离）
   - 每个租户拥有独立数据库
   - 最高安全性
   - 适合大型企业客户

2. **共享数据库模式**（默认）
   - 所有租户共享数据库
   - 通过 `TenantId` 字段隔离
   - 成本更低，易于维护

### 连接字符串管理
- 支持租户自定义连接字符串
- 自动解析租户连接字符串
- 支持多数据库类型（MySQL、SQL Server、PostgreSQL、Oracle）

## 主要服务

### ITenantStore
租户存储接口，提供租户查询功能。

**实现类**：`SqlSugarAndConfigurationTenantStore`

**关键方法**：
- `FindAsync(Guid id)` - 按 ID 查询租户
- `FindAsync(string name)` - 按名称查询租户
- `GetCacheItemAsync(...)` - 缓存租户信息

### TenantService
租户管理应用服务，继承自 `YiCrudAppService`，提供标准的 CRUD 操作。

**关键方法**：
- `GetListAsync(TenantGetListInput input)` - 获取租户列表
- `GetSelectAsync()` - 获取租户选项（用于下拉框）
- `CreateAsync(...)` - 创建租户
- `InitAsync(Guid id)` - 初始化租户数据

### IConnectionStringResolver
连接字符串解析器接口。

**实现类**：`YiMultiTenantConnectionStringResolver`

**解析逻辑**：
1. 检查当前租户上下文
2. 查询租户自定义连接字符串
3. 如果租户未定义，回退到默认连接字符串

## 租户配置

### 数据结构

#### TenantAggregateRoot
租户主实体。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| Name | string | 租户名称 |
| TenantConnectionString | string | 租户连接字符串 |
| DbType | DbType | 数据库类型 |
| EntityVersion | int | 实体版本 |

**审计字段**：
- `CreationTime` - 创建时间
- `CreatorId` - 创建者 ID
- `LastModificationTime` - 最后修改时间
- `LastModifierId` - 最后修改者 ID
- `IsDeleted` - 是否删除

### 租户连接字符串
在 `appsettings.json` 中配置租户连接字符串：

```json
{
  "ConnectionStrings": {
    "Default": "Data Source=db/ast_intellisub.db"
  },
  "AbpMultiTenancy": {
    "Tenants": [
      {
        "Id": " tenant-guid-1",
        "Name": "Tenant A",
        "ConnectionStrings": {
          "Default": "Server=localhost;Database=tenant_a;User=root;Password=pass;"
        }
      }
    ]
  }
}
```

### 数据库配置
```json
{
  "DbConnOptions": {
    "DeploymentMode": "HighPerformance",
    "DbType": "Sqlite",
    "Url": "Data Source=db/ast_intellisub.db"
  }
}
```

## 数据隔离策略

### 行级隔离（默认）
所有需要隔离的实体实现 `IMultiTenant` 接口：

```csharp
[SugarTable("Device")]
public class DeviceAggregateRoot : AggregateRoot<Guid>, IMultiTenant
{
    public Guid? TenantId { get; set; }
    // ... 其他字段
}
```

SqlSugar 会自动添加 `TenantId` 过滤条件：

```sql
SELECT * FROM Device WHERE TenantId = @CurrentTenantId
```

### 跨租户操作
使用 `ICurrentTenant` 进行跨租户操作：

```csharp
// 切换到租户上下文
using (_currentTenant.Change(tenantId))
{
    // 在此代码块中，所有操作都在指定租户下执行
    var devices = await _deviceRepository.GetListAsync();
}

// 切换到宿主（无租户）
using (_currentTenant.Change(null))
{
    // 在此代码块中，操作在宿主级别执行
    var allTenants = await _tenantRepository.GetListAsync();
}
```

### 全局查询过滤器
在数据访问层自动应用租户过滤器：

```csharp
// 在 SqlSugar AOP 中自动添加
dataExecutor.Queryable<T>()
    .WhereIF(CurrentTenant.Id != null, x => x.TenantId == CurrentTenant.Id)
```

## 租户管理流程

### 创建租户
```csharp
// 1. 创建租户实体
var tenant = new TenantAggregateRoot(
    id: GuidGenerator.Create(),
    name: "新租户"
);

// 2. 设置连接字符串（可选）
tenant.SetConnectionString(
    DbType.MySql,
    "Server=localhost;Database=new_tenant;..."
);

// 3. 保存租户
await _tenantRepository.InsertAsync(tenant);

// 4. 初始化租户数据
await _dataSeeder.SeedAsync(tenant.Id);
```

### 租户数据初始化
每个租户创建后需要初始化基础数据：
- 默认角色和权限
- 默认管理员用户
- 系统配置和设置
- 业务基础数据（可选）

## 相关文档

- [[ABP 框架集成]] - ABP 多租户框架说明
- [[数据访问层]] - SqlSugar 多租户支持
- [[审计日志模块]] - 审计日志的租户隔离

## 注意事项

1. **宿主管理**：宿主用户（超级管理员）可以管理所有租户
2. **性能影响**：行级隔离会自动添加过滤条件，注意索引优化
3. **数据迁移**：独立数据库模式需要分别处理每个租户的迁移
4. **缓存策略**：租户信息会缓存，修改后需要清除缓存
