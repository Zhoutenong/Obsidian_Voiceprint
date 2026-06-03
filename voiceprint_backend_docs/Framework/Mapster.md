# Yi.Framework.Mapster

## 概述

`Yi.Framework.Mapster` 是基于 Mapster 的高性能对象映射框架，为 ABP 框架提供类型安全的对象转换能力。Mapster 是一个高性能的对象映射库，相比 AutoMapper 具有更快的转换速度和更低的内存占用。

**核心特性：**
- ✅ 高性能对象转换（比 AutoMapper 快）
- ✅ 零配置约定映射
- ✅ 支持投影查询优化
- ✅ 深度克隆支持
- ✅ 灵活的映射配置

## 模块配置

### 依赖模块

```csharp
[DependsOn(
    typeof(YiFrameworkCoreModule),
    typeof(AbpObjectMappingModule)
)]
public class YiFrameworkMapsterModule : AbpModule
```

### 服务注册

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddTransient<IAutoObjectMappingProvider, MapsterAutoObjectMappingProvider>();
}
```

**注册的服务：**
- `IAutoObjectMappingProvider`: 自动对象映射提供者接口实现

## 核心组件

### MapsterAutoObjectMappingProvider

实现 ABP 的 `IAutoObjectMappingProvider` 接口，使用 Mapster 的 `Adapt` 方法进行对象转换：

```csharp
public class MapsterAutoObjectMappingProvider : IAutoObjectMappingProvider
{
    public TDestination Map<TSource, TDestination>(object source)
    {
        var sss = typeof(TDestination).Name;
        return source.Adapt<TDestination>();
    }

    public TDestination Map<TSource, TDestination>(TSource source, TDestination destination)
    {
        return source.Adapt<TSource, TDestination>(destination);
    }
}
```

### MapsterObjectMapper

对象映射器接口实现（当前为空实现）：

```csharp
public class MapsterObjectMapper : IObjectMapper
{
    public IAutoObjectMappingProvider AutoObjectMappingProvider => throw new NotImplementedException();

    public TDestination Map<TSource, TDestination>(TSource source)
    {
        throw new NotImplementedException();
    }

    public TDestination Map<TSource, TDestination>(TSource source, TDestination destination)
    {
        throw new NotImplementedException();
    }
}
```

## 使用方式

### 应用服务集成

ABP 应用服务自动集成对象映射功能：

```csharp
public class UserAppService : ApplicationService
{
    private readonly IRepository<User, Guid> _userRepository;

    public UserAppService(IRepository<User, Guid> userRepository)
    {
        _userRepository = userRepository;
    }

    public async Task<UserDto>.GetAsync(Guid id)
    {
        var user = await _userRepository.GetAsync(id);
        
        // 自动映射 User -> UserDto
        return ObjectMapper.Map<UserDto>(user);
    }

    public async Task<User> CreateAsync(CreateUserDto input)
    {
        // 自动映射 CreateUserDto -> User
        var user = ObjectMapper.Map<User>(input);
        
        return await _userRepository.InsertAsync(user);
    }
}
```

### 直接使用映射

```csharp
public class MyService : ITransientDependency
{
    private readonly IAutoObjectMappingProvider _mappingProvider;

    public MyService(IAutoObjectMappingProvider mappingProvider)
    {
        _mappingProvider = mappingProvider;
    }

    public UserDto ConvertToDto(User user)
    {
        return _mappingProvider.Map<UserDto>(user);
    }

    public User UpdateFromDto(User user, UpdateUserDto dto)
    {
        return _mappingProvider.Map<UpdateUserDto, User>(dto, user);
    }
}
```

## 映射约定

### 默认映射规则

**1. 同名属性映射：**
```csharp
public class Source
{
    public string Name { get; set; }
    public int Age { get; set; }
}

public class Destination
{
    public string Name { get; set; }
    public int Age { get; set; }
}

// 自动映射 Name 和 Age
var dest = source.Adapt<Destination>();
```

**2. 嵌套对象映射：**
```csharp
public class Order
{
    public string OrderNo { get; set; }
    public Customer Customer { get; set; }
}

public class OrderDto
{
    public string OrderNo { get; set; }
    public CustomerDto Customer { get; set; }
}

// 自动映射嵌套对象
var orderDto = order.Adapt<OrderDto>();
```

**3. 集合映射：**
```csharp
List<User> users = await _userRepository.GetListAsync();
List<UserDto> userDtos = users.Adapt<List<UserDto>>();
```

### 自定义映射配置

**1. 全局映射配置：**
```csharp
// 在模块初始化时配置
TypeAdapterConfig<TSource, TDestination>
    .NewConfig()
    .Map(dest => dest.FullName, src => $"{src.FirstName} {src.LastName}");
```

**2. 单次映射配置：**
```csharp
var result = source.Adapt<Destination>(config => 
    config.ForType<TSource, Destination>()
        .Map(dest => dest.DisplayName, src => src.Name)
);
```

### 忽略属性

```csharp
TypeAdapterConfig<TSource, TDestination>
    .NewConfig()
    .Ignore(dest => dest.InternalId)
    .Ignore(dest => dest.CreatedAt);
```

### 条件映射

```csharp
TypeAdapterConfig<User, UserDto>
    .NewConfig()
    .Map(dest => dest.Status, src => src.IsActive ? "Active" : "Inactive");
```

## 高级特性

### 投影查询优化

Mapster 支持 EF Core 查询投影优化，仅查询需要的字段：

```csharp
public async Task<List<UserDto>> GetActiveUsersAsync()
{
    var query = await _userRepository.GetQueryableAsync();
    
    // 投影到 UserDto，只查询需要的字段
    return await query
        .Where(u => u.IsActive)
        .ProjectToType<UserDto>()
        .ToListAsync();
}
```

### 深度克隆

```csharp
var clonedUser = originalUser.Adapt<User>();
```

### 类型转换

```csharp
TypeAdapterConfig<string, int>
    .NewConfig()
    .MapWith(str => int.Parse(str));
```

## 性能优化

### 编译时映射

Mapster 支持编译时代码生成：

```csharp
// 安装 Mapster.Tools 包
// 使用代码生成器生成映射代码

[AdaptTo(typeof(UserDto))]
public partial class User { }
```

### 配置缓存

Mapster 自动缓存映射配置，无需手动优化：

```csharp
// 首次映射时会编译并缓存
var dto = entity.Adapt<EntityDto>(); 
// 后续映射使用缓存的配置
```

## 与 ABP 集成

### IObjectMapper 接口

ABP 提供的 `IObjectMapper` 接口，在 `ApplicationService` 基类中自动可用：

```csharp
public class MyApplicationService : ApplicationService
{
    // 通过 ObjectMapper 属性访问
    public async Task<EntityDto> GetAsync(Guid id)
    {
        var entity = await _repository.GetAsync(id);
        return ObjectMapper.Map<EntityDto>(entity);
    }
}
```

### Repository 映射

在 Repository 层也可以使用对象映射：

```csharp
public class CustomRepository : EfCoreRepository<MyDbContext, Entity, Guid>
{
    public async Task<EntityDto> GetDtoAsync(Guid id)
    {
        var entity = await DbSet.FindAsync(id);
        return entity.Adapt<EntityDto>();
    }
}
```

## 最佳实践

### 1. DTO 设计原则

```csharp
// ✅ 好的做法：扁平化 DTO
public class UserProfileDto
{
    public string FullName { get; set; }
    public string Email { get; set; }
    public string Status { get; set; }
}

// ❌ 避免：过度嵌套
public class UserProfileDto
{
    public UserBasicInfo BasicInfo { get; set; }
    public UserContactInfo ContactInfo { get; set; }
    public UserStatusInfo StatusInfo { get; set; }
}
```

### 2. 验证映射结果

```csharp
var dto = entity.Adapt<EntityDto>();

// 验证必填字段
if (string.IsNullOrEmpty(dto.Name))
{
    throw new MappingException("Name cannot be null");
}
```

### 3. 处理循环引用

```csharp
TypeAdapterConfig<Entity, EntityDto>
    .NewConfig()
    .MaxDepth(2); // 限制映射深度
```

### 4. 性能监控

```csharp
using var activity = ActivitySource.StartActivity("MapObject");
var dto = entity.Adapt<EntityDto>();
```

## 故障排查

### 映射失败

**问题：** 属性名称不匹配导致映射失败

```csharp
// 源类型
public class Source { public string UserName { get; set; } }

// 目标类型
public class Dest { public string Name { get; set; } }

// 解决：配置映射规则
TypeAdapterConfig<Source, Dest>
    .NewConfig()
    .Map(dest => dest.Name, src => src.UserName);
```

### 集合映射问题

**问题：** 集合元素类型映射错误

```csharp
// 确保元素类型有映射配置
TypeAdapterConfig<User, UserDto>.NewConfig();
TypeAdapterConfig<Order, OrderDto>.NewConfig();

// 然后映射集合
var userDtos = users.Adapt<List<UserDto>>();
```

### 性能问题

**问题：** 映射性能不佳

```csharp
// 使用投影查询优化
var dtos = await query.ProjectToType<EntityDto>().ToListAsync();

// 而不是
var entities = await query.ToListAsync();
var dtos = entities.Adapt<List<EntityDto>>();
```

## 参考资源

- [Mapster Documentation](https://github.com/MapsterMapper/Mapster)
- [ABP Object Mapping](https://docs.abp.io/en/abp/latest/Object-Mapping)
- [Mapster vs AutoMapper](https://github.com/MapsterMapper/Mapster/blob/master/docs/performance.md)
