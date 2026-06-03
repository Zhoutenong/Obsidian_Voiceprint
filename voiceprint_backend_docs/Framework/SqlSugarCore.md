# Yi.Framework.SqlSugarCore

SqlSugarCore 模块提供基于 SqlSugar ORM 的完整数据访问层实现，支持多种数据库类型和部署模式。

## 核心组件

### 1. 仓储基类

#### SqlSugarRepository<TEntity>

泛型仓储基类，提供完整的 CRUD 操作：

**位置：** `framework/Yi.Framework.SqlSugarCore/Repositories/SqlSugarRepository.cs`

```csharp
public class SqlSugarRepository<TEntity> : ISqlSugarRepository<TEntity>
    where TEntity : class, IEntity, new()
{
    // 数据库客户端访问
    public ISqlSugarClient _Db { get; }
    public ISugarQueryable<TEntity> _DbQueryable { get; }

    // 单表查询
    Task<TEntity> GetByIdAsync(dynamic id);
    Task<TEntity> GetFirstAsync(Expression<Func<TEntity, bool>> whereExpression);
    Task<bool> IsAnyAsync(Expression<Func<TEntity, bool>> whereExpression);
    Task<int> CountAsync(Expression<Func<TEntity, bool>> whereExpression);

    // 多表查询
    Task<List<TEntity>> GetListAsync();
    Task<List<TEntity>> GetListAsync(Expression<Func<TEntity, bool>> whereExpression);

    // 分页查询
    Task<List<TEntity>> GetPageListAsync(
        Expression<Func<TEntity, bool>> whereExpression,
        int pageNum, int pageSize,
        Expression<Func<TEntity, object>>? orderByExpression = null,
        OrderByType orderByType = OrderByType.Asc
    );

    // 插入操作
    Task<bool> InsertAsync(TEntity insertObj);
    Task<bool> InsertRangeAsync(List<TEntity> insertObjs);
    Task<int> InsertReturnIdentityAsync(TEntity insertObj);
    Task<long> InsertReturnSnowflakeIdAsync(TEntity insertObj);

    // 更新操作
    Task<bool> UpdateAsync(TEntity updateObj);
    Task<bool> UpdateRangeAsync(List<TEntity> updateObjs);

    // 删除操作
    Task<bool> DeleteAsync(dynamic id);
    Task<bool> DeleteAsync(TEntity deleteObj);
    Task<int> DeleteAsync(Expression<Func<TEntity, bool>> whereExpression);

    // 高级查询
    Task<IInsertable<TEntity>> AsInsertable(TEntity insertObj);
    Task<IUpdateable<TEntity>> AsUpdateable(TEntity updateObj);
    Task<IDeleteable<TEntity>> AsDeleteable();
    Task<ISugarQueryable<TEntity>> AsQueryable();
}
```

#### SqlSugarRepository<TEntity, TKey>

带主键类型的泛型仓储，实现 ABP `IRepository<TEntity, TKey>` 接口：

```csharp
public class SqlSugarRepository<TEntity, TKey> : SqlSugarRepository<TEntity>,
    ISqlSugarRepository<TEntity, TKey>, IRepository<TEntity, TKey>
    where TEntity : class, IEntity<TKey>, new()
{
    Task<TEntity?> FindAsync(TKey id, bool includeDetails = true);
    Task<TEntity> GetAsync(TKey id, bool includeDetails = true);
    Task DeleteAsync(TKey id, bool autoSave = false);
    Task DeleteManyAsync(IEnumerable<TKey> ids, bool autoSave = false);
}
```

### 2. 数据库上下文

#### DefaultSqlSugarDbContext

默认数据库上下文实现，提供数据过滤和审计功能：

**位置：** `framework/Yi.Framework.SqlSugarCore/DefaultSqlSugarDbContext.cs`

**核心功能：**

1. **软删除过滤**
```csharp
protected override void CustomDataFilter(ISqlSugarClient sqlSugarClient)
{
    if (IsSoftDeleteFilterEnabled)
    {
        sqlSugarClient.QueryFilter.AddTableFilter<ISoftDelete>(
            u => u.IsDeleted == false
        );
    }

    if (IsMultiTenantFilterEnabled)
    {
        var expressionCurrentTenant = CurrentTenant.Id ?? null;
        sqlSugarClient.QueryFilter.AddTableFilter<IMultiTenant>(
            u => u.TenantId == expressionCurrentTenant
        );
    }
}
```

2. **审计日志**
```csharp
public override void DataExecuting(object oldValue, DataFilterModel entityInfo)
{
    switch (entityInfo.OperationType)
    {
        case DataFilterType.InsertByObject:
            entityInfo.SetValue(CurrentTenant.Id);
            entityInfo.SetValue(DateTime.Now);
            entityInfo.SetValue(CurrentUser.Id);
            break;

        case DataFilterType.UpdateByObject:
            entityInfo.SetValue(DateTime.Now);
            entityInfo.SetValue(CurrentUser.Id);
            break;
    }
}
```

3. **SQL 日志记录**
```csharp
public override void OnLogExecuting(string sql, SugarParameter[] pars)
{
    if (_options.EnabledSqlLog)
    {
        _logger.LogInformation($"Sql：{sql} ，参数：{JsonHelper.SerializeObject(pars)}");
    }
}
```

### 3. DbConnOptions 配置

**位置：** `framework/Yi.Framework.SqlSugarCore.Abstractions/DbConnOptions.cs`

#### 配置属性

| 属性 | 类型 | 默认值 | 说明 |
|-----|------|--------|------|
| `Url` | string? | - | 连接字符串（必填） |
| `DbType` | DbType? | - | 数据库类型 |
| `EnabledCodeFirst` | bool | false | 开启 CodeFirst 表结构自动创建 |
| `EnabledSqlLog` | bool | true | 开启 SQL 日志 |
| `EnableUnderLine` | bool | false | 开启驼峰转下划线 |
| `EnabledDbSeed` | bool | false | 开启种子数据 |
| `EnabledReadWrite` | bool | false | 开启读写分离 |
| `ReadUrl` | List<string>? | - | 读库连接列表 |
| `EnabledSaasMultiTenancy` | bool | false | 开启 SaaS 多租户 |
| `TDengineUrl` | string | - | TDengine 时序数据库连接 |
| `DeploymentMode` | DeploymentMode | HighPerformance | 部署模式 |

#### 部署模式

```csharp
public enum DeploymentMode
{
    /// 高性能模式：PostgreSQL + TDengine
    HighPerformance = 0,

    /// 低资源模式：单一 SQLite（ARM32/1GB 内存）
    LowResource = 1
}
```

#### 配置示例

**低资源模式（工控机）：**
```json
{
  "DbConnOptions": {
    "DeploymentMode": "LowResource",
    "Url": "Data Source=db/ast_intellisub.db",
    "DbType": "Sqlite",
    "EnabledCodeFirst": true,
    "EnabledSqlLog": true,
    "EnableUnderLine": false
  }
}
```

**高性能模式（数据中心）：**
```json
{
  "DbConnOptions": {
    "DeploymentMode": "HighPerformance",
    "Url": "Host=localhost;Port=5432;Database=ast_intellisub;Username=postgres;Password=123456",
    "DbType": "PostgreSQL",
    "EnabledCodeFirst": true,
    "EnabledSqlLog": true,
    "EnabledReadWrite": true,
    "ReadUrl": [
      "Host=slave1;Port=5432;Database=ast_intellisub",
      "Host=slave2;Port=5432;Database=ast_intellisub"
    ],
    "TDengineUrl": "Host=localhost;Port=6030;Database=ast_tsdb"
  }
}
```

### 4. 工作单元集成

#### UnitOfWorkSqlsugarDbContextProvider

**位置：** `framework/Yi.Framework.SqlSugarCore/Uow/UnitOfWorkSqlsugarDbContextProvider.cs`

工作单元提供器，确保数据库上下文在事务边界内正确管理：

```csharp
public class UnitOfWorkSqlsugarDbContextProvider<TDbContext>
    : ISugarDbContextProvider<TDbContext>
    where TDbContext : ISqlSugarDbContext
{
    public virtual async Task<TDbContext> GetDbContextAsync()
    {
        var unitOfWork = UnitOfWorkManager.Current;
        if (unitOfWork == null)
        {
            throw new AbpException(
                "DbContext 只能在工作单元内工作"
            );
        }

        var databaseApi = unitOfWork.FindDatabaseApi(dbContextKey);
        if (databaseApi == null)
        {
            databaseApi = new SqlSugarDatabaseApi(
                await CreateDbContextAsync(unitOfWork, connectionStringName, connectionString)
            );
            unitOfWork.AddDatabaseApi(dbContextKey, databaseApi);
        }

        return (TDbContext)((SqlSugarDatabaseApi)databaseApi).DbContext;
    }
}
```

**关键特性：**
- 强制工作单元模式（避免数据一致性风险）
- 事务复用（同一工作单元内共享数据库连接）
- 多租户连接字符串解析
- AsyncLocal 上下文传递

### 5. 多租户支持

#### DefaultTenantTableAttribute

标记默认租户表的特性：

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class DefaultTenantTableAttribute : Attribute
{
}
```

**使用示例：**
```csharp
[DefaultTenantTable]
public class UserAggregateRoot : Entity<Guid>, IMultiTenant
{
    public Guid? TenantId { get; set; }
    // ...
}
```

#### 多租户数据过滤

在 `DefaultSqlSugarDbContext` 中自动应用：

```csharp
protected override void CustomDataFilter(ISqlSugarClient sqlSugarClient)
{
    if (IsMultiTenantFilterEnabled)
    {
        var expressionCurrentTenant = CurrentTenant.Id ?? null;
        sqlSugarClient.QueryFilter.AddTableFilter<IMultiTenant>(
            u => u.TenantId == expressionCurrentTenant
        );
    }
}
```

## 数据库支持

### 支持的数据库类型

| 数据库 | DbType 枚举 | 适用场景 |
|--------|-------------|----------|
| SQLite | DbType.Sqlite | 低资源环境、工控机 |
| MySQL | DbType.MySql | 通用 Web 应用 |
| PostgreSQL | DbType.PostgreSQL | 高并发、复杂查询 |
| SQL Server | DbType.SqlServer | Windows 环境 |
| Oracle | DbType.Oracle | 企业级应用 |

### SequentialGuid 配置

根据不同数据库自动配置 GUID 生成策略：

```csharp
SequentialGuidType guidType;
switch (dbConnOptions.DbType)
{
    case DbType.MySql:
    case DbType.PostgreSQL:
        guidType = SequentialGuidType.SequentialAsString;
        break;
    case DbType.SqlServer:
        guidType = SequentialGuidType.SequentialAtEnd;
        break;
    case DbType.Oracle:
        guidType = SequentialGuidType.SequentialAsBinary;
        break;
    default:
        guidType = SequentialGuidType.SequentialAtEnd;
        break;
}

Configure<AbpSequentialGuidGeneratorOptions>(options =>
{
    options.DefaultSequentialGuidType = guidType;
});
```

## CodeFirst 自动建表

### 启用配置

```json
{
  "DbConnOptions": {
    "EnabledCodeFirst": true
  }
}
```

### 实体配置

```csharp
// 表名配置
[SugarTable("sys_user")]
public class UserAggregateRoot : Entity<Guid>
{
    // 主键配置
    [SugarColumn(IsPrimaryKey = true, IsIdentity = true)]
    public override Guid Id { get; protected set; }

    // 列名配置
    [SugarColumn(ColumnName = "user_name", Length = 50)]
    public string UserName { get; set; }

    // 忽略字段
    [SugarColumn(IsIgnore = true)]
    public string TempProp { get; set; }
}
```

### CodeFirst 执行

在 `YiFrameworkSqlSugarCoreModule` 中：

```csharp
public override async Task OnPreApplicationInitializationAsync(ApplicationInitializationContext context)
{
    // 如果开启 CodeFirst
    if (_dbConnOptions.EnabledCodeFirst)
    {
        var db = context.ServiceProvider.GetRequiredService<ISqlSugarClient>();
        // 创建所有表结构
        db.CodeFirst.InitTables(
            typeof(YiRbacDbContext).Assembly.ExportedTypes.ToArray()
        );
    }
}
```

## 自定义仓储

### 创建自定义仓储接口

```csharp
public interface IUserRepository : ISqlSugarRepository<UserAggregateRoot, Guid>
{
    Task<UserAggregateRoot?> GetByUserNameAsync(string userName);
    Task<List<UserAggregateRoot>> GetActiveUsersAsync();
}
```

### 实现自定义仓储

```csharp
public class UserRepository : SqlSugarRepository<UserAggregateRoot, Guid>,
    IUserRepository
{
    public UserRepository(
        ISugarDbContextProvider<ISqlSugarDbContext> sugarDbContextProvider
    ) : base(sugarDbContextProvider) { }

    public async Task<UserAggregateRoot?> GetByUserNameAsync(string userName)
    {
        var db = await _DbQueryable;
        return await db
            .Where(x => x.UserName == userName)
            .FirstAsync();
    }

    public async Task<List<UserAggregateRoot>> GetActiveUsersAsync()
    {
        var db = await _DbQueryable;
        return await db
            .Where(x => !x.IsDeleted)
            .ToListAsync();
    }
}
```

### 注册自定义仓储

```csharp
public class YiFrameworkRbacSqlSugarCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddYiDbContext<YiRbacDbContext>();
    }
}
```

## 事务管理

### 隐式事务（应用服务层）

```csharp
public class UserAppService : ApplicationService
{
    private readonly IUserRepository _userRepository;
    private readonly IRoleRepository _roleRepository;

    // 应用服务方法自动启用工作单元（事务）
    public async Task CreateUserWithRoleAsync(CreateUserDto input)
    {
        var user = ObjectMapper.Map<UserAggregateRoot>(input);
        await _userRepository.InsertAsync(user);

        var role = await _roleRepository.GetAsync(input.RoleId);
        // 所有操作在同一事务中
        await _roleRepository.AddUserAsync(role, user);
    }
}
```

### 显式事务

```csharp
public class CustomService
{
    private readonly IUnitOfWorkManager _unitOfWorkManager;
    private readonly IUserRepository _userRepository;

    public async Task ExecuteInTransactionAsync()
    {
        using var uow = _unitOfWorkManager.Begin();
        try
        {
            await _userRepository.InsertAsync(new User { /* ... */ });
            // 其他操作...

            await uow.CompleteAsync(); // 提交事务
        }
        catch
        {
            await uow.RollbackAsync(); // 回滚事务
            throw;
        }
    }
}
```

## 性能优化

### 批量操作

```csharp
// 插入 1000 条数据
var users = Enumerable.Range(1, 1000)
    .Select(i => new UserAggregateRoot { UserName = $"user{i}" })
    .ToList();

await _userRepository.InsertRangeAsync(users);
```

### 查询优化

```csharp
// 使用 AsQueryable 构建复杂查询
var db = await _userRepository.AsQueryable();
var result = await db
    .LeftJoin<Role>((u, r) => u.RoleId == r.Id)
    .Where((u, r) => !u.IsDeleted && r.IsActive)
    .OrderBy((u, r) => u.CreationTime, OrderByType.Desc)
    .Select((u, r) => new { u.UserName, r.RoleName })
    .ToPageListAsync(pageNum, pageSize);
```

### 读写分离

```json
{
  "DbConnOptions": {
    "EnabledReadWrite": true,
    "Url": "Host=master;Database=ast",  // 主库（写）
    "ReadUrl": [                         // 从库（读）
      "Host=slave1;Database=ast",
      "Host=slave2;Database=ast"
    ]
  }
}
```

## 最佳实践

1. **始终使用仓储接口**，避免直接使用 `ISqlSugarClient`
2. **启用工作单元**，确保数据一致性
3. **使用软删除**，避免数据丢失
4. **配置 SQL 日志**，便于调试
5. **批量操作**使用 `InsertRangeAsync` 等方法
6. **复杂查询**使用 `AsQueryable` 构建
7. **多租户场景**标记实体为 `IMultiTenant`

## 相关文档

- [Framework 概览](./README.md)
- [BackgroundWorkers.Hangfire](./BackgroundWorkersHangfire.md)
- [AspNetCore](./AspNetCore.md)
