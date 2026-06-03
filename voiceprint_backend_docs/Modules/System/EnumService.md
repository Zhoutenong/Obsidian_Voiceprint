# EnumService (枚举服务)

## 概述
EnumService 是系统的枚举值查询服务，提供统一的枚举类型数据访问接口。该服务支持跨程序集枚举查询，允许前端动态获取系统中定义的所有枚举类型及其描述信息。

## 职责
- 动态扫描和加载系统程序集中的枚举类型
- 提供枚举值的查询服务，包含值、名称和描述信息
- 支持通过 [[EnumDescriptionAttribute]] 特性自定义枚举项描述
- 管理多模块程序集的枚举类型发现

## 主要接口

### GetEnumListAsync
获取指定枚举类型的所有枚举项列表

**参数：**
- `enumName` (string) - 枚举类型名称

**返回：**
- `Task<List<EnumItemDto>>` - 枚举项列表，包含值、名称和描述

**异常：**
- `UserFriendlyException` - 当指定的枚举类型不存在时抛出

## 技术实现

### 程序集加载策略
服务使用静态程序集缓存机制，避免重复加载：

1. **当前程序集** - 加载 EnumService 所在的程序集
2. **引用程序集** - 自动加载当前程序集引用的所有程序集
3. **特定模块** - 预配置加载以下核心模块：
   - Yi.Framework.Rbac.Domain.Shared
   - Yi.Framework.Rbac.Application.Contracts
   - Yi.Framework.Rbac.Application
   - Yi.Framework.Rbac.Domain
   - Ast.IntelliSub.Domain.Shared
   - Ast.IntelliSub.Application.Contracts
   - Ast.IntelliSub.Application
   - Ast.IntelliSub.Domain

### 枚举发现机制
1. 首先按枚举名称精确匹配
2. 如果未找到，在预定义命名空间中搜索：
   - Yi.Framework.Rbac.Domain.Shared.OperLog
   - Yi.Framework.Rbac.Domain.Shared
   - Ast.IntelliSub.Domain.Shared
   - Ast.IntelliSub.Application.Contracts
3. 如果仍未找到，列出所有可用枚举类型供参考

## 数据传输对象

### EnumItemDto
```csharp
public class EnumItemDto
{
    public int Value { get; set; }        // 枚举值
    public string Name { get; set; }      // 枚举名称
    public string Description { get; set; } // 枚举描述
}
```

## 认证与授权
- `[AllowAnonymous]` - 该服务允许匿名访问，无需身份认证

## 使用示例

### API 调用
```http
GET /api/app/enum/enum-list?enumName=DeviceStatusEnum
```

### 响应示例
```json
{
  "result": [
    {
      "value": 0,
      "name": "Online",
      "description": "在线"
    },
    {
      "value": 1,
      "name": "Offline",
      "description": "离线"
    }
  ]
}
```

## 相关服务
- [[ApplicationService]] - ABP 应用服务基类
- [[IEnumService]] - 枚举服务接口

## 相关文档
- [[枚举特性定义]]
- [[前端枚举集成指南]]
