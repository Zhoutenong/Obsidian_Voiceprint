# NoticeAggregateRoot — 公告通知聚合根

## 基本信息

- **实体名称**：`NoticeAggregateRoot`
- **数据库表**：`Notice`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Entities/`
- **继承关系**：`AggregateRoot<Guid>` → `ISoftDelete` → `IAuditedObject` → `IOrderNum` → `IState`

## 实体说明

公告通知聚合根用于管理系统中的公告和通知信息。支持公告类型、状态控制、排序等功能，可以向用户发布重要通知。

## 字段说明

### 主键与状态
- `Id` (Guid) - 主键
- `IsDeleted` (bool) - 软删除标记
- `State` (bool) - 发布状态（true=已发布，false=草稿）

### 公告信息
- `Title` (string) - 公告标题
- `Type` (NoticeTypeEnum) - 公告类型
- `Content` (string) - 公告内容（BigString类型）

### 排序与审计
- `OrderNum` (int) - 排序字段
- `CreationTime` (DateTime) - 创建时间
- `CreatorId` (Guid?) - 创建者ID
- `LastModifierId` (Guid?) - 最后修改者ID
- `LastModificationTime` (DateTime?) - 最后修改时间

## 枚举类型
### NoticeTypeEnum — 公告类型
- 系统公告
- 通知公告
- 紧急公告

## 业务规则
1. **发布控制**：通过State字段控制公告是否发布
2. **软删除**：删除公告仅标记IsDeleted
3. **排序支持**：OrderNum控制公告显示顺序
4. **内容丰富**：支持大文本内容的公告

## 相关服务
- NoticeService - 公告通知服务
- NoticeHub - SignalR通知Hub

## 数据查询示例
```sql
-- 查询已发布的公告
SELECT * FROM Notice 
WHERE state = 1 
AND is_deleted = 0 
ORDER BY order_num DESC;

-- 查询某类型的公告
SELECT * FROM Notice 
WHERE type = 0 
AND is_deleted = 0 
ORDER BY creation_time DESC;
```

## 业务价值
- 向用户发布系统公告和通知
- 支持草稿和发布状态管理
- 提供多种公告类型分类
- 通过SignalR实时推送通知

---

> **最后更新**：2026-06-04
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Entities/NoticeAggregateRoot.cs`
