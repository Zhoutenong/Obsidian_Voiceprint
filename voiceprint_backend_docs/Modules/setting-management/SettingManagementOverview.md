# 系统设置模块概述

## 概述

系统设置模块（setting-management）提供灵活的配置管理功能，支持不同层级（全局、租户、用户）的设置定义、存储和访问，用于管理应用程序的各种配置参数。

**模块路径**：`module/setting-management/`

## 功能说明

### 系统配置
- 应用级别的全局配置
- 模块级别的功能开关
- 第三方服务的集成配置

### 用户配置
- 用户个性化设置
- 用户界面偏好
- 用户通知设置

### 设置加密
敏感设置支持加密存储，如 API 密钥、密码等。

## 主要服务

### ISettingManager
设置管理器，提供设置的读取和修改功能。

**实现类**：`SettingManager`

**关键方法**：
- `GetOrNullAsync(string name, ...)` - 获取设置值
- `GetAllAsync(...)` - 获取所有设置
- `SetAsync(string name, string value, ...)` - 设置值
- `DeleteAsync(...)` - 删除设置

### ISettingManagementStore
设置存储接口，定义底层存储操作。

**实现**：通过 `ISettingRepository` 实现

### SettingManagementProvider
设置提供者基类，定义不同层级设置的访问策略。

**实现类**：
- `DefaultValueSettingManagementProvider` - 默认值提供者
- `ConfigurationSettingManagementProvider` - 配置文件提供者
- `GlobalSettingManagementProvider` - 全局设置提供者
- `TenantSettingManagementProvider` - 租户设置提供者
- `UserSettingManagementProvider` - 用户设置提供者

### ISettingRepository
设置仓储接口。

**实现类**：`SqlSugarCoreSettingRepository`

**关键方法**：
- `FindAsync(name, providerName, providerKey)` - 查找设置
- `GetListAsync(...)` - 获取设置列表

## 设置类型

### 设置层级

设置提供者的优先级（从高到低）：

1. **用户设置** (`UserSettingValueProvider.ProviderName`)
   - 针对特定用户的个性化设置
   - ProviderKey: 用户 ID

2. **租户设置** (`TenantSettingValueProvider.ProviderName`)
   - 针对特定租户的配置
   - ProviderKey: 租户 ID

3. **全局设置** (`GlobalSettingValueProvider.ProviderName`)
   - 应用于所有租户和用户
   - ProviderKey: null

4. **默认值** (`DefaultValueSettingValueProvider.ProviderName`)
   - 设置定义的默认值
   - 无法修改

### 设置定义
设置需要在代码中定义：

```csharp
public class MySettingDefinitionProvider : SettingDefinitionProvider
{
    public override void Define(ISettingDefinitionContext context)
    {
        context.Add(
            new SettingDefinition(
                name: "MyApp.MaxUploadSize",
                defaultValue: "10485760",
                displayName: "最大上传文件大小",
                isVisibleToClients: true
            )
        );
    }
}
```

### 设置加密
敏感设置需要加密：

```csharp
new SettingDefinition(
    name: "MyApp.ApiKey",
    defaultValue: "",
    displayName: "API 密钥",
    isEncrypted: true  // 加密存储
)
```

## 设置缓存

### 缓存策略
设置使用分布式缓存（Redis 或内存）提高性能：

```csharp
// 设置缓存项
public class SettingCacheItem
{
    public string Name { get; set; }
    public string Value { get; set; }
    public string ProviderName { get; set; }
    public string ProviderKey { get; set; }
}
```

### 缓存键
```
SettingCacheItem:{TenantId}:{ProviderName}:{ProviderKey}:{Name}
```

### 缓存失效
- 设置修改后自动清除缓存
- 支持按租户、用户维度清除
- 缓存考虑工作单元（UOW）

## 数据结构

### SettingAggregateRoot
设置实体。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| Name | string | 设置名称 |
| Value | string | 设置值 |
| ProviderName | string? | 提供者名称 |
| ProviderKey | string? | 提供者键 |

**表名**：`Setting`

## 使用示例

### 读取设置
```csharp
public class MyService
{
    private readonly ISettingManager _settingManager;

    public MyService(ISettingManager settingManager)
    {
        _settingManager = settingManager;
    }

    public async Task<long> GetMaxUploadSizeAsync()
    {
        // 获取当前用户的设置（fallback 到租户、全局、默认）
        var value = await _settingManagerOrNullAsync("MyApp.MaxUploadSize");
        return long.Parse(value ?? "10485760");
    }

    public async Task<string> GetTenantSettingAsync(Guid? tenantId)
    {
        // 获取指定租户的设置
        return await _settingManager.GetOrNullAsync(
            name: "MyApp.LogoUrl",
            providerName: TenantSettingValueProvider.ProviderName,
            providerKey: tenantId?.ToString()
        );
    }
}
```

### 修改设置
```csharp
// 修改全局设置
await _settingManager.SetForGlobalAsync("MyApp.MaintenanceMode", "true");

// 修改租户设置
await _settingManager.SetForTenantAsync(tenantId, "MyApp.Theme", "dark");

// 修改用户设置
await _settingManager.SetForUserAsync(userId, "MyApp.Language", "zh-CN");
```

### 批量获取
```csharp
// 获取所有全局设置
var settings = await _settingManager.GetAllAsync(
    providerName: GlobalSettingValueProvider.ProviderName,
    providerKey: null
);

// 获取特定设置
var values = await _settingManager.GetAllAsync(
    providerName: UserSettingValueProvider.ProviderName,
    providerKey: userId.ToString()
);
```

## 相关文档

- [[ABP 框架集成]] - ABP 设置系统说明
- [[多租户模块]] - 租户级设置说明
- [[缓存管理]] - 分布式缓存使用说明

## 注意事项

1. **性能考虑**：设置会缓存，避免频繁修改
2. **类型安全**：设置值为字符串，使用时需转换和验证
3. **敏感数据**：敏感设置必须加密存储
4. **定义优先**：使用前必须先定义设置
