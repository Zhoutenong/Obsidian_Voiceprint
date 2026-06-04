# VideoLayoutDetailEntity — 视频布局详情实体

## 基本信息

- **实体名称**：`VideoLayoutDetailEntity`
- **数据库表**：`ast_video_layout_detail`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Video/`
- **继承关系**：`Entity<Guid>`

## 实体说明

视频布局详情实体用于定义视频布局中每个位置的摄像头配置。支持用户自定义多画面视频墙布局，如 2x2、3x3 等分屏显示。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `LayoutId` | `Guid` | 布局ID | Foreign Key |
| `DeviceId` | `string` | 摄像头设备ID | Foreign Key |

### 位置信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Position` | `int` | 位置序号 | 从1开始，表示在布局中的位置 |

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(LayoutId))]
public VideoLayoutFavoriteEntity VideoLayoutFavorite { get; set; }

[Navigate(NavigateType.OneToOne, nameof(DeviceId))]
public DeviceEntity Device { get; set; }
```

## 业务规则

### 布局配置规则
1. **位置唯一性**：同一布局内位置序号唯一
2. **设备可重复**：同一摄像头可以在多个布局中使用
3. **位置排序**：按 Position 顺序显示，支持灵活布局
4. **空位处理**：允许某些位置为空（不关联摄像头）

### 布局类型示例

#### 2x2 布局（4分屏）
```
┌─────────┬─────────┐
│  位置1   │  位置2  │
│ 摄像头A  │ 摄像头B │
├─────────┼─────────┤
│  位置3   │  位置4  │
│ 摄像头C  │   空    │
└─────────┴─────────┘
```

#### 3x3 布局（9分屏）
```
┌────┬────┬────┐
│ 1  │ 2  │ 3  │
├────┼────┼────┤
│ 4  │ 5  │ 6  │
├────┼────┼────┤
│ 7  │ 8  │ 9  │
└────┴────┴────┘
```

## 相关实体

- [[VideoLayoutFavoriteEntity]] — 视频布局收藏（父实体）
- [[DeviceEntity]] — 摄像头设备实体

## 相关服务

- [[CameraService]] — 摄像机服务
- [[MediaService]] — 媒体服务

## 数据查询示例

### 查询某布局的所有摄像头配置
```sql
SELECT
    vd.position,
    vd.device_id,
    d.name as device_name,
    d.rtsp_url,
    d.stream_url
FROM ast_video_layout_detail vd
LEFT JOIN ast_device d ON vd.device_id = d.device_id
WHERE vd.layout_id = 'xxx'
ORDER BY vd.position;
```

### 查询某摄像头在哪些布局中使用
```sql
SELECT
    vf.id as layout_id,
    vf.name as layout_name,
    vf.layout_type,
    vd.position
FROM ast_video_layout_detail vd
JOIN ast_video_layout_favorite vf ON vd.layout_id = vf.id
WHERE vd.device_id = 'xxx'
ORDER BY vf.last_use_time DESC;
```

## 索引建议

```sql
-- 布局ID索引（用于查询某布局的摄像头配置）
CREATE INDEX IX_layout_id ON ast_video_layout_detail(layout_id);

-- 设备ID索引（用于查询某摄像头在哪些布局中使用）
CREATE INDEX IX_device_id ON ast_video_layout_detail(device_id);

-- 位置索引（用于按位置排序显示）
CREATE INDEX IX_position ON ast_video_layout_detail(position);
```

## 前端展示建议

### 2x2 视频墙布局
```tsx
<div className="grid grid-cols-2 grid-rows-2 gap-2">
  {[1, 2, 3, 4].map(position => {
    const detail = layoutDetails.find(d => d.position === position);
    return (
      <VideoPlayer
        key={position}
        deviceId={detail?.device_id}
        position={position}
      />
    );
  })}
</div>
```

### 动态N x M布局
```tsx
<div className={`grid grid-cols-${cols} grid-rows-${rows} gap-2`}>
  {layoutDetails
    .sort((a, b) => a.position - b.position)
    .map(detail => (
      <VideoPlayer
        key={detail.id}
        deviceId={detail.device_id}
        position={detail.position}
      />
    ))}
</div>
```

## 业务价值

**核心作用**：
1. **个性化布局**：支持用户自定义视频墙布局
2. **灵活配置**：支持任意位置配置任意摄像头
3. **快速切换**：保存常用布局，一键切换
4. **多屏支持**：支持从1分屏到16分屏等多种布局

## 与VideoLayoutFavoriteEntity的关系

- `VideoLayoutFavoriteEntity` 定义布局的"名称、类型、用户"
- `VideoLayoutDetailEntity` 定义布局的"具体摄像头配置"
- 一对多关系：一个布局包含多个位置配置

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Video/VideoLayoutDetailEntity.cs`