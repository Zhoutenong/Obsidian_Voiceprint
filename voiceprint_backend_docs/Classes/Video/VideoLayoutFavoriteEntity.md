# VideoLayoutFavoriteEntity — 视频布局收藏实体

## 基本信息

- **实体名称**：`VideoLayoutFavoriteEntity`
- **数据库表**：`ast_video_layout_favorite`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Video/`
- **继承关系**：`AggregateRoot<Guid>`

## 实体说明

视频布局收藏实体用于保存用户的个性化视频墙布局配置。用户可以创建、保存、切换不同的布局方案，如"重要设备监控"、"全站概览"等。

## 字段说明

### 主键与基础信息

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `Name` | `string` | 布局名称 | 如："重要设备监控" |
| `UserId` | `Guid` | 用户ID | Foreign Key |
| `LayoutType` | `VideoLayoutTypeEnum` | 布局类型 | 见下方枚举 |

### 时间信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `CreationTime` | `DateTime` | 创建时间 | - |
| `LastUseTime` | `DateTime?` | 最后使用时间 | 用于排序最近使用 |

### 状态信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `IsDeleted` | `bool` | 是否删除 | 软删除标记 |

## 枚举类型

### VideoLayoutTypeEnum — 视频布局类型
```csharp
public enum VideoLayoutTypeEnum
{
    Single = 1,      // 1x1 单屏
    FourSplit = 2,   // 2x2 四分屏
    NineSplit = 3,   // 3x3 九分屏
    SixteenSplit = 4 // 4x4 十六分屏
}
```

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(UserId))]
public Yi.Framework.Rbac.Domain.Entities.UserAggregateRoot User { get; set; }
```

## 业务规则

### 布局管理规则
1. **用户专属**：每个用户有独立的布局收藏
2. **软删除**：删除布局仅标记 IsDeleted，不物理删除
3. **最近使用**：通过 LastUseTime 记录最后使用时间
4. **命名唯一**：同一用户下布局名称不能重复

### 预置布局建议
| 布局名称 | 布局类型 | 说明 | 适用场景 |
|---------|---------|------|---------|
| 全站概览 | NineSplit | 显示9路关键摄像头 | 总览监控 |
| 重要设备 | FourSplit | 显示4路重要设备 | 重点监控 |
| 单设备详看 | Single | 显示1路高清摄像头 | 详细查看 |
| 环境监测 | FourSplit | 显示4路环境摄像头 | 环境监控 |

## 使用场景

### 场景1：运维人员自定义布局
```
用户A创建的布局：
├── "主变监控"（2x2布局）
│   ├── 位置1: #1主变-正面
│   ├── 位置2: #1主变-侧面
│   ├── 位置3: #1主变-顶部
│   └── 位置4: #1主变-仪表
├── "开关柜室"（3x3布局）
│   └── 9个开关柜的摄像头
└── "全站概览"（3x3布局）
    └── 全站重要区域摄像头
```

### 场景2：布局切换与使用
```
1. 用户登录后，显示最近使用的布局
2. 用户可以从布局列表中快速切换
3. 系统记录 LastUseTime，用于下次登录时默认显示
4. 用户可以编辑现有布局或创建新布局
```

## 相关实体

- [[VideoLayoutDetailEntity]] — 视频布局详情（子实体，一对多）
- [[UserAggregateRoot]] — 用户实体（RBAC模块）

## 相关服务

- [[CameraService]] — 摄像机服务
- [[MediaService]] — 媒体服务
- [[StreamingApiService]] — 流媒体API服务

## 数据查询示例

### 查询某用户的所有布局
```sql
SELECT * FROM ast_video_layout_favorite
WHERE user_id = 'xxx'
AND is_deleted = 0
ORDER BY last_use_time DESC, creation_time DESC;
```

### 查询布局及其摄像头配置
```sql
SELECT
    vf.id as layout_id,
    vf.name as layout_name,
    vf.layout_type,
    vd.position,
    vd.device_id,
    d.name as device_name
FROM ast_video_layout_favorite vf
LEFT JOIN ast_video_layout_detail vd ON vf.id = vd.layout_id
LEFT JOIN ast_device d ON vd.device_id = d.device_id
WHERE vf.user_id = 'xxx'
AND vf.is_deleted = 0
ORDER BY vf.last_use_time DESC, vd.position;
```

### 查询最近使用的布局
```sql
SELECT TOP 1 * FROM ast_video_layout_favorite
WHERE user_id = 'xxx'
AND is_deleted = 0
ORDER BY last_use_time DESC;
```

## 索引建议

```sql
-- 用户ID索引（用于查询某用户的布局）
CREATE INDEX IX_user_id ON ast_video_layout_favorite(user_id);

-- 创建时间索引（用于按创建时间排序）
CREATE INDEX IX_creation_time ON ast_video_layout_favorite(creation_time);

-- 布局类型索引（用于按类型筛选）
CREATE INDEX IX_layout_type ON ast_video_layout_favorite(layout_type);
```

## 业务价值

**核心作用**：
1. **个性化体验**：每个用户可以有独立的监控布局
2. **快速切换**：保存常用布局，一键切换不同监控场景
3. **使用追踪**：记录最后使用时间，智能推荐
4. **灵活管理**：支持创建、编辑、删除布局

## 设计模式

### 聚合根模式
- `VideoLayoutFavoriteEntity` 是聚合根
- `VideoLayoutDetailEntity` 是聚合的一部分
- 只能通过聚合根访问和修改聚合内部对象

### 软删除模式
```csharp
// 删除布局
public async Task DeleteAsync(Guid layoutId)
{
    var layout = await _repository.GetAsync(layoutId);
    layout.IsDeleted = true;
    await _repository.UpdateAsync(layout);
}

// 查询时自动过滤
public async Task<List<VideoLayoutFavoriteEntity>> GetUserLayoutsAsync(Guid userId)
{
    return await _repository.GetListAsync(l => l.UserId == userId && !l.IsDeleted);
}
```

## 前端展示建议

### 布局列表
```tsx
<div className="layout-list">
  {layouts.map(layout => (
    <div
      key={layout.id}
      className={`layout-item ${isActive ? 'active' : ''}`}
      onClick={() => switchLayout(layout.id)}
    >
      <div className="layout-icon">
        {getLayoutIcon(layout.layoutType)}
      </div>
      <div className="layout-info">
        <h3>{layout.name}</h3>
        <p>{formatTime(layout.lastUseTime)}</p>
      </div>
    </div>
  ))}
</div>
```

### 布局创建向导
```tsx
<Wizard>
  <Step1>选择布局类型（1x1/2x2/3x3/4x4）</Step1>
  <Step2>为每个位置选择摄像头</Step2>
  <Step3>命名并保存布局</Step3>
</Wizard>
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Video/VideoLayoutFavoriteEntity.cs`