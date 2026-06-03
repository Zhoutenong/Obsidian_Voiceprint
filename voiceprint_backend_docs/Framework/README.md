# Framework 核心组件

Framework 层提供可复用的基础设施组件，支撑上层业务模块的快速开发。

## 组件依赖图

```
┌─────────────────────────────────────────────────────────────┐
│                    Yi.Framework.Core                         │
│                  (核心工具、枚举、扩展)                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼────────┐ ┌──▼──────────┐ ┌─▼──────────────────┐
│  SqlSugarCore  │ │AspNetCore   │ │BackgroundWorkers   │
│  (数据访问层)   │ │(Web API层)  │ │(后台任务调度)       │
└───────┬────────┘ └──┬──────────┘ └─┬──────────────────┘
        │              │              │
        └──────────────┼──────────────┘
                       │
              ┌────────▼─────────┐
              │   上层业务模块     │
              │ (rbac/ast-*等)    │
              └──────────────────┘
```

## 核心模块

### 1. Yi.Framework.SqlSugarCore

**职责：** 提供 SqlSugar ORM 的 ABP 集成，实现仓储模式和工作单元管理

**核心功能：**
- 仓储基类 `SqlSugarRepository<TEntity>` / `SqlSugarRepository<TEntity, TKey>`
- 工作单元提供器 `UnitOfWorkSqlsugarDbContextProvider`
- 数据库上下文 `DefaultSqlSugarDbContext`
- 多租户数据过滤
- 软删除审计
- CodeFirst 表结构自动创建

**数据库支持：**
- SQLite（默认，低资源模式）
- MySQL
- PostgreSQL
- SQL Server
- Oracle

### 2. Yi.Framework.AspNetCore

**职责：** 统一 Web API 层的返回格式、路由约定和异常处理

**核心功能：**
- 统一返回结果 `RESTfulResult`
- 全局异常过滤器 `FriendlyExceptionFilter`
- 成功响应过滤器 `SucceededUnifyResultFilter`
- RESTful 风格结果提供器 `RESTfulResultProvider`
- ABP 路由约定 `YiConventionalRouteBuilder`
- Swagger/OpenAPI 配置

### 3. Yi.Framework.BackgroundWorkers.Hangfire

**职责：** 基于 Hangfire 的后台任务调度和管理

**核心功能：**
- 工作单元过滤器 `UnitOfWorkHangfireFilter`
- 任务自动注册 `YiHangfireConventionalRegistrar`
- Token 认证过滤器 `YiTokenAuthorizationFilter`
- Cron 表达式配置
- 内存/Redis 存储支持

### 4. Yi.Framework.Caching.FreeRedis

**职责：** 基于 FreeRedis 的分布式缓存实现

**核心功能：**
- Redis 缓存键规范化 `YiDistributedCacheKeyNormalizer`
- 多租户缓存隔离
- 连接配置管理

### 5. Yi.Framework.Mapster

**职责：** 对象映射，替代 AutoMapper

**核心功能：**
- 自动对象映射提供器 `MapsterAutoObjectMappingProvider`
- 性能优化的映射器

### 6. Yi.Framework.Ddd.Application

**职责：** DDD 应用服务基类

**核心功能：**
- CRUD 应用服务基类 `YiCrudAppService`
- 缓存 CRUD 应用服务 `YiCacheCrudAppService`

## 模块间依赖关系

```
业务模块依赖：
- module/* 依赖 framework/*

Framework 内部依赖：
- Yi.Framework.SqlSugarCore 依赖 Yi.Framework.Core
- Yi.Framework.AspNetCore 依赖 Yi.Framework.Core
- Yi.Framework.BackgroundWorkers.Hangfire 依赖 ABP Hangfire 模块
- Yi.Framework.Ddd.Application 依赖 Yi.Framework.Core 和 ABP DDD 模块
```

## 设计模式

### 仓储模式（Repository Pattern）

SqlSugarCore 实现了标准的仓储模式，将数据访问逻辑封装在仓储接口中：

```csharp
public interface ISqlSugarRepository<TEntity> where TEntity : class, IEntity
{
    Task<TEntity> GetByIdAsync(dynamic id);
    Task<List<TEntity>> GetListAsync();
    Task<bool> InsertAsync(TEntity insertObj);
    Task<bool> UpdateAsync(TEntity updateObj);
    Task<bool> DeleteAsync(dynamic id);
    // ... 更多方法
}
```

### 工作单元模式（Unit of Work）

通过 `UnitOfWorkSqlsugarDbContextProvider` 管理，确保：
- 事务边界明确
- 数据库连接复用
- 多仓储操作的一致性

### 依赖注入（Dependency Injection）

所有组件都遵循 ABP 的依赖注入约定：
- `ITransientDependency` - 瞬态生命周期
- `ISingletonDependency` - 单例生命周期
- `IScopedDependency` - 作用域生命周期

## 配置示例

### appsettings.json

```json
{
  "DbConnOptions": {
    "DeploymentMode": "LowResource",
    "Url": "Data Source=db/ast_intellisub.db",
    "DbType": "Sqlite",
    "EnabledCodeFirst": true,
    "EnabledSqlLog": true
  },
  "Hangfire": {
    "StorageMode": "Memory"
  },
  "Redis": {
    "IsEnabled": false
  },
  "UnifyResultSettings": {
    "Return200StatusCodes": [401, 403],
    "SupportMvcController": false
  }
}
```

## 扩展点

### 自定义仓储

```csharp
public class CustomRepository : SqlSugarRepository<MyEntity>, ICustomRepository
{
    public CustomRepository(ISugarDbContextProvider<ISqlSugarDbContext> provider)
        : base(provider) { }

    // 添加自定义查询方法
    public async Task<List<MyEntity>> GetCustomDataAsync()
    {
        var db = await _DbQueryable;
        return await db.Where(x => x.Status == 1).ToListAsync();
    }
}
```

### 自定义后台任务

```csharp
public class MyBackgroundWorker : HangfireBackgroundWorker
{
    public MyBackgroundWorker()
    {
        RecurringJobId = "my-job";
        CronExpression = "0 */5 * * * *"; // 每5分钟
    }

    public override async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        // 任务逻辑
    }
}
```

### 自定义统一返回

```csharp
public class CustomResultProvider : IUnifyResultProvider, ITransientDependency
{
    public IActionResult OnException(ExceptionContext context, ExceptionMetadata metadata)
    {
        // 自定义异常返回格式
        return new JsonResult(new { error = metadata.Errors });
    }

    public IActionResult OnSucceeded(ActionExecutedContext context, object data)
    {
        // 自定义成功返回格式
        return new JsonResult(new { success = true, result = data });
    }
}
```

## 最佳实践

1. **仓储使用：** 优先使用 `ISqlSugarRepository` 接口，避免直接使用 `ISqlSugarClient`
2. **工作单元：** 应用服务方法自动启用工作单元，无需手动管理
3. **后台任务：** 继承 `HangfireBackgroundWorker` 基类，配置 `RecurringJobId` 和 `CronExpression`
4. **异常处理：** 抛出 `UserFriendlyException` 实现友好异常提示
5. **数据库配置：** 低资源环境使用 SQLite，高并发环境使用 PostgreSQL + Redis

## 相关文档

- [SqlSugarCore 详细文档](./SqlSugarCore.md)
- [BackgroundWorkers.Hangfire 详细文档](./BackgroundWorkersHangfire.md)
- [AspNetCore 详细文档](./AspNetCore.md)
