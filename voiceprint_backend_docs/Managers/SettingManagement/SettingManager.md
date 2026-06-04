# SettingManager — 系统设置管理器

## 基本信息

- **Manager名称**：`SettingManager`
- **模块位置**：`module/setting-management/Yi.Framework.SettingManagement.Domain/`
- **继承关系**：`ISettingManager`
- **生命周期**：`ISingletonDependency` - 单例
- **命名空间**：`Yi.Framework.SettingManagement.Domain`

## Manager概述

SettingManager 是系统设置管理器，负责设置的读取、写入和加密。它支持多Provider（提供者）架构，允许设置从不同来源获取（全局、租户、用户等），并支持设置的继承和加密。

## 核心职责

1. **设置读取** - 根据Provider获取设置值
2. **设置写入** - 保存设置值到指定Provider
3. **设置加密** - 自动加密/解密敏感设置
4. **设置继承** - 支持设置从全局到用户的继承链
5. **多Provider管理** - 管理多个设置提供者

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ISettingDefinitionManager` | 设置定义管理器 |
| `ISettingEncryptionService` | 设置加密服务 |
| `IOptions<SettingManagementOptions>` | 设置管理选项 |
| `IServiceProvider` | 服务提供者（获取Provider实例） |

## 核心方法

### 1. GetOrNullAsync - 获取单个设置值

**签名**：
```csharp
public virtual Task<string> GetOrNullAsync(
    string name, 
    string providerName, 
    string providerKey, 
    bool fallback = true)
```

**功能**：
获取指定名称的设置值，支持回退（fallback）到继承的设置。

**参数说明**：
- `name` - 设置名称
- `providerName` - 提供者名称（如："Global", "Tenant", "User"）
- `providerKey` - 提供者键（如租户ID、用户ID）
- `fallback` - 是否回退到继承的设置值

**执行流程**：

```
1. 参数验证
   │
   ├─ Check.NotNull(name, nameof(name))
   ├─ Check.NotNull(providerName, nameof(providerName))
   │
2. 调用内部方法
   │
   └─ GetOrNullInternalAsync(name, providerName, providerKey, fallback)
      │
   3. 获取设置定义
      │
      ├─ var setting = await SettingDefinitionManager.GetAsync(name)
      │
   4. 确定Provider范围
      │
      ├─ if (!fallback || !setting.IsInherited)
      │  └─ 只使用指定Provider
      │
      └─ if (fallback && setting.IsInherited)
         └─ 从指定Provider向下遍历所有Provider
            │
         5. 遍历Provider链
            │
            ├─ foreach (var provider in providers)
            │  │
            │  ├─ var value = await provider.GetOrNullAsync(setting, providerKey)
            │  │
            │  └─ if (value != null)
            │     └─ break（找到值后停止）
            │
         6. 解密设置值（如果加密）
            │
            └─ if (setting.IsEncrypted)
               └─ value = SettingEncryptionService.Decrypt(setting, value)
            │
         7. 返回值
```

### 2. GetAllAsync - 获取所有设置值

**签名**：
```csharp
public virtual async Task<List<SettingValue>> GetAllAsync(
    string providerName, 
    string providerKey, 
    bool fallback = true)
```

**功能**：
获取指定Provider的所有设置值。

**执行流程**：

```
1. 参数验证
   │
   └─ Check.NotNull(providerName, nameof(providerName))
   │
2. 获取所有设置定义
   │
   └─ var settingDefinitions = await SettingDefinitionManager.GetAllAsync()
   │
3. 确定Provider范围
   │
   ├─ var providers = Enumerable.Reverse(Providers)
   │     .SkipWhile(c => c.Name != providerName)
   │
   ├─ if (!fallback)
   │  └─ 只使用指定Provider
   │
   └─ if (fallback)
      └─ 包含继承链中的所有Provider
         │
4. 遍历每个设置定义
   │
   ├─ foreach (var setting in settingDefinitions)
   │  │
   │  ├─ 处理继承设置
   │  │  │
   │  │  ├─ if (setting.IsInherited)
   │  │  │  └─ 从Provider链中查找第一个非空值
   │  │  │     foreach (var provider in providers)
   │  │  │        if (providerValue != null)
   │  │  │           value = providerValue
   │  │  │
   │  └─ 处理非继承设置
   │     └─ value = await providerList[0].GetOrNullAsync(...)
   │
   ├─ 解密设置值
   │  │
   │  └─ if (setting.IsEncrypted)
   │     └─ value = SettingEncryptionService.Decrypt(setting, value)
   │
   └─ 添加到结果列表
      └─ settingValues[setting.Name] = new SettingValue(name, value)
   │
5. 返回设置值列表
```

### 3. SetAsync - 设置值

**签名**：
```csharp
public virtual async Task SetAsync(
    string name, 
    string value, 
    string providerName, 
    string providerKey, 
    bool forceToSet = false)
```

**功能**：
设置指定名称的设置值。

**参数说明**：
- `name` - 设置名称
- `value` - 设置值
- `providerName` - 提供者名称
- `providerKey` - 提供者键
- `forceToSet` - 是否强制设置（即使值与继承值相同）

**执行流程**：

```
1. 参数验证
   │
   ├─ Check.NotNull(name, nameof(name))
   ├─ Check.NotNull(providerName, nameof(providerName))
   │
2. 获取设置定义
   │
   └─ var setting = await SettingDefinitionManager.GetAsync(name)
   │
3. 确定Provider范围
   │
   └─ var providers = Enumerable.Reverse(Providers)
      .SkipWhile(p => p.Name != providerName)
      .ToList()
   │
4. 加密设置值（如果需要）
   │
   └─ if (setting.IsEncrypted)
      └─ value = SettingEncryptionService.Encrypt(setting, value)
   │
5. 处理继承设置优化
   │
   ├─ if (providers.Count > 1 && !forceToSet && setting.IsInherited && value != null)
   │  │
   │  ├─ 获取回退值（下一个Provider的值）
   │  │  └─ var fallbackValue = await GetOrNullInternalAsync(name, providers[1].Name, null)
   │  │
   │  └─ 如果值相同，清空当前值（避免重复存储）
   │     └─ if (fallbackValue == value)
   │        └─ value = null
   │
6. 过滤到相同ProviderName的Provider
   │
   └─ providers = providers.TakeWhile(p => p.Name == providerName).ToList()
   │
7. 设置或清空值
   │
   ├─ if (value == null)
   │  │
   │  └─ 清空所有Provider的值
   │     └─ foreach (var provider in providers)
   │        └─ await provider.ClearAsync(setting, providerKey)
   │
   └─ else
      │
      └─ 设置所有Provider的值
         └─ foreach (var provider in providers)
            └─ await provider.SetAsync(setting, value, providerKey)
```

## 设置继承链

### Provider优先级（从高到低）

```
User（用户设置）
   │
   ├─ 继承 → Tenant（租户设置）
   │             │
   │             └─ 继承 → Global（全局设置）
```

### 继承示例

**场景**：获取用户的"Smtp.Host"设置

```
1. 查询用户设置（UserId="123"）
   │
   ├─ User Provider → 未找到
   │
2. 回退到租户设置（TenantId="456"）
   │
   ├─ Tenant Provider → 找到值 "smtp.tenant.com"
   │
3. 返回 "smtp.tenant.com"
```

**如果租户也没有**：
```
1. 查询用户设置
   │
   ├─ User Provider → 未找到
   │
2. 回退到租户设置
   │
   ├─ Tenant Provider → 未找到
   │
3. 回退到全局设置
   │
   ├─ Global Provider → 找到值 "smtp.global.com"
   │
4. 返回 "smtp.global.com"
```

## 设置加密

### 加密设置处理

```csharp
// 定义加密设置
public class SmtpSettings
{
    [SettingDefinition(IsEncrypted = true)]
    public string Password { get; set; }
}

// 保存时自动加密
await _settingManager.SetAsync(
    "Smtp.Password",
    "plain_text_password", // 明文
    "Global",
    null
);
// 内部调用：SettingEncryptionService.Encrypt(setting, value)
// 存储到数据库的是加密后的值

// 读取时自动解密
var password = await _settingManager.GetOrNullAsync(
    "Smtp.Password",
    "Global",
    null
);
// 内部调用：SettingEncryptionService.Decrypt(setting, value)
// 返回的是解密后的明文值
```

## 设置管理选项

### SettingManagementOptions

```csharp
public class SettingManagementOptions
{
    public List<Type> Providers { get; set; }
}
```

### 配置示例

```json
{
  "Settings": {
    "Providers": [
      "Yi.Framework.SettingManagement.Providers.GlobalSettingProvider",
      "Yi.Framework.SettingManagement.Providers.TenantSettingProvider",
      "Yi.Framework.SettingManagement.Providers.UserSettingProvider"
    ]
  }
}
```

## 使用场景

### 1. 获取用户设置（带回退）

```csharp
// 获取用户的主题设置
// 如果用户没有设置，回退到租户设置
// 如果租户也没有，回退到全局设置
var theme = await _settingManager.GetOrNullAsync(
    "Ui.Theme",
    "User",    // Provider名称
    userId,    // 用户ID
    true       // 启用回退
);
```

### 2. 设置租户配置

```csharp
// 为租户设置SMTP服务器
await _settingManager.SetAsync(
    "Smtp.Host",
    "smtp.tenant456.com",
    "Tenant",
    tenantId
);
```

### 3. 获取全局设置（无回退）

```csharp
// 只获取全局设置，不回退
var setting = await _settingManager.GetOrNullAsync(
    "System.Name",
    "Global",
    null,
    false  // 禁用回退
);
```

### 4. 批量获取设置

```csharp
// 获取用户的所有设置
var settings = await _settingManager.GetAllAsync(
    "User",
    userId,
    true  // 包含继承的设置
);

// settings 是 List<SettingValue>
foreach (var setting in settings)
{
    Console.WriteLine($"{setting.Name}: {setting.Value}");
}
```

### 5. 清空设置

```csharp
// 清空用户的某个设置（将回退到继承的设置）
await _settingManager.SetAsync(
    "Ui.Theme",
    null,  // 传null表示清空
    "User",
    userId
);
```

### 6. 强制设置（即使与继承值相同）

```csharp
// 强制为用户设置值，即使值与全局相同
await _settingManager.SetAsync(
    "System.Language",
    "zh-CN",
    "User",
    userId,
    true  // forceToSet = true
);
```

## 设计特点

1. **单例模式** - 使用 `ISingletonDependency` 确保全局唯一实例
2. **Provider模式** - 支持多个设置来源（全局、租户、用户）
3. **继承机制** - 设置可以从全局向下继承
4. **自动加密** - 敏感设置自动加密存储
5. **延迟加载** - Provider列表使用 `Lazy<T>` 延迟初始化
6. **优化存储** - 如果值与继承值相同，自动清空避免重复

## Provider管理

### Provider接口

```csharp
public interface ISettingManagementProvider
{
    string Name { get; }
    
    Task<string> GetOrNullAsync(
        SettingDefinition setting, 
        string providerKey);
    
    Task SetAsync(
        SettingDefinition setting, 
        string value, 
        string providerKey);
    
    Task ClearAsync(
        SettingDefinition setting, 
        string providerKey);
}
```

### 内置Provider

1. **GlobalSettingProvider** - 全局设置（providerKey = null）
2. **TenantSettingProvider** - 租户设置（providerKey = tenantId）
3. **UserSettingProvider** - 用户设置（providerKey = userId）

## 性能考虑

### 延迟加载

```csharp
private readonly Lazy<List<ISettingManagementProvider>> _lazyProviders;

// 只在第一次访问时初始化
protected List<ISettingManagementProvider> Providers => _lazyProviders.Value;
```

### 继承优化

```csharp
// 如果设置的值与继承值相同，清空当前值
// 避免存储重复数据
if (fallbackValue == value)
{
    value = null;  // 清空
}
```

## 相关扩展

### UserSettingManagerExtensions

```csharp
public static class UserSettingManagerExtensions
{
    public static Task<string> GetOrNullForUserAsync(
        this ISettingManager manager,
        string name,
        Guid userId)
    {
        return manager.GetOrNullAsync(name, "User", userId.ToString());
    }
}
```

### TenantSettingManagerExtensions

```csharp
public static class TenantSettingManagerExtensions
{
    public static Task<string> GetOrNullForTenantAsync(
        this ISettingManager manager,
        string name,
        Guid tenantId)
    {
        return manager.GetOrNullAsync(name, "Tenant", tenantId.ToString());
    }
}
```

## 相关服务

- `SettingService` - 设置服务（应用层）
- `SettingDefinitionManager` - 设置定义管理器
- `SettingEncryptionService` - 设置加密服务

## 相关实体

- `SettingAggregateRoot` - 设置聚合根
- `SettingDefinition` - 设置定义

## 注意事项

1. **Provider顺序** - Provider在选项中的顺序决定了继承优先级
2. **加密设置** - 加密设置的值在数据库中是密文，只在读取时解密
3. **回退行为** - 使用 `fallback=false` 可以禁用继承回退
4. **性能优化** - 批量获取时使用 `GetAllAsync` 而非循环调用 `GetOrNullAsync`

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/setting-management/Yi.Framework.SettingManagement.Domain/SettingManager.cs`  
> **接口定义**：`module/setting-management/Yi.Framework.SettingManagement.Domain/ISettingManager.cs`
