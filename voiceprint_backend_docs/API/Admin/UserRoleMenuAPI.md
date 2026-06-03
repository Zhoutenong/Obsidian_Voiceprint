# 用户角色菜单 API 文档

## 概述

本文档详细说明 RBAC 模块中的核心管理接口，包括用户管理、角色管理、菜单管理、部门管理和岗位管理的 CRUD 操作及权限分配功能。

## 基础路径

```
/api/app/rbac
```

---

## 用户管理

### 获取用户列表

**接口**: `GET /api/app/rbac/user`

**描述**: 获取用户列表，支持分页和条件查询。

**权限**: `system:user:list`

**请求参数**:

```typescript
interface UserGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  userName?: string;           // 用户名（模糊查询）
  nickName?: string;           // 昵称（模糊查询）
  deptId?: Guid;               // 部门ID
  postId?: Guid;               // 岗位ID
  state?: boolean;             // 状态（true=启用，false=禁用）
  sorting?: string;            // 排序字段
}
```

**请求示例**:
```json
{
  "skipCount": 0,
  "maxResultCount": 10,
  "userName": "admin",
  "state": true
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 100,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "userName": "admin",
        "nickName": "管理员",
        "email": "admin@example.com",
        "phone": "13800138000",
        "sex": 1,
        "deptId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "deptName": "技术部",
        "postId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "postName": "技术总监",
        "state": true,
        "icon": "base64-image-data",
        "orderNum": 1,
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 获取用户详情

**接口**: `GET /api/app/rbac/user/{id}`

**描述**: 获取指定用户的详细信息。

**权限**: `system:user:query`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 用户ID |

**响应**: 返回 `UserGetOutputDto` 对象（同列表项结构）

---

### 创建用户

**接口**: `POST /api/app/rbac/user`

**描述**: 创建新用户。

**权限**: `system:user:add`

**请求参数**:

```typescript
interface UserCreateInputVo {
  userName: string;           // 用户名（必填，唯一）
  nickName: string;           // 昵称（必填）
  password: string;           // 密码（必填）
  email?: string;             // 邮箱
  phone?: string;             // 手机号
  sex?: number;               // 性别（1=男，2=女）
  deptId?: Guid;              // 部门ID
  postId?: Guid;              // 岗位ID
  icon?: string;              // 头像（Base64）
  orderNum?: number;           // 排序号
}
```

**请求示例**:
```json
{
  "userName": "testuser",
  "nickName": "测试用户",
  "password": "123456",
  "email": "test@example.com",
  "phone": "13900139000",
  "sex": 1,
  "deptId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "postId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

**响应**: 返回创建的用户信息 `UserGetOutputDto`

---

### 更新用户

**接口**: `PUT /api/app/rbac/user/{id}`

**描述**: 更新用户信息。

**权限**: `system:user:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 用户ID |

**请求参数**: 同 `UserCreateInputVo`（所有字段可选）

**响应**: 返回更新后的用户信息 `UserGetOutputDto`

---

### 删除用户

**接口**: `DELETE /api/app/rbac/user/{id}`

**描述**: 删除指定用户。

**权限**: `system:user:remove`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 用户ID |

**响应**: `204 No Content`

---

### 批量删除用户

**接口**: `DELETE /api/app/rbac/user`

**描述**: 批量删除多个用户。

**权限**: `system:user:remove`

**请求参数**:

```typescript
interface DeleteManyInput {
  ids: Guid[];  // 用户ID列表
}
```

**响应**: `204 No Content`

---

### 更新用户状态

**接口**: `PUT /api/app/rbac/user/{id}/{state}`

**描述**: 启用或禁用用户。

**权限**: `system:user:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 用户ID |
| state | boolean | 状态（true=启用，false=禁用） |

**响应**: 返回更新后的用户信息 `UserGetOutputDto`

---

### 为用户分配角色

**接口**: `POST /api/app/rbac/user/{id}/role`

**描述**: 为用户分配一个或多个角色。

**权限**: `system:user:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 用户ID |

**请求参数**:

```typescript
interface AssignRoleInput {
  roleIds: Guid[];  // 角色ID列表
}
```

**响应**: `204 No Content`

---

### 为用户分配岗位

**接口**: `POST /api/app/rbac/user/{id}/post`

**描述**: 为用户分配岗位。

**权限**: `system:user:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 用户ID |

**请求参数**:

```typescript
interface AssignPostInput {
  postId: Guid;  // 岗位ID
}
```

**响应**: `204 No Content`

---

### 为用户分配部门

**接口**: `POST /api/app/rbac/user/{id}/dept`

**描述**: 为用户分配部门。

**权限**: `system:user:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 用户ID |

**请求参数**:

```typescript
interface AssignDeptInput {
  deptId: Guid;  // 部门ID
}
```

**响应**: `204 No Content`

---

## 角色管理

### 获取角色列表

**接口**: `GET /api/app/rbac/role`

**描述**: 获取角色列表，支持分页和条件查询。

**权限**: `system:role:list`

**请求参数**:

```typescript
interface RoleGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  roleCode?: string;           // 角色编码（模糊查询）
  roleName?: string;           // 角色名称（模糊查询）
  state?: boolean;             // 状态（true=启用，false=禁用）
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 20,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "roleCode": "admin",
        "roleName": "管理员",
        "state": true,
        "dataScope": 1,
        "dataScopeName": "全部数据",
        "orderNum": 1,
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 创建角色

**接口**: `POST /api/app/rbac/role`

**描述**: 创建新角色。

**权限**: `system:role:add`

**请求参数**:

```typescript
interface RoleCreateInputVo {
  roleCode: string;           // 角色编码（必填，唯一）
  roleName: string;           // 角色名称（必填）
  state?: boolean;            // 状态
  dataScope?: number;         // 数据权限（1=全部，2=自定义，3=本部门，4=本部门及以下，5=仅本人）
  orderNum?: number;          // 排序号
}
```

**响应**: 返回创建的角色信息 `RoleGetOutputDto`

---

### 更新角色

**接口**: `PUT /api/app/rbac/role/{id}`

**描述**: 更新角色信息，同时更新角色菜单权限。

**权限**: `system:role:edit`

**请求参数**:

```typescript
interface RoleUpdateInputVo {
  roleCode: string;           // 角色编码（必填，唯一）
  roleName: string;           // 角色名称（必填）
  state?: boolean;            // 状态
  dataScope?: number;         // 数据权限
  orderNum?: number;          // 排序号
  menuIds?: Guid[];           // 菜单ID列表（分配菜单权限）
}
```

**响应**: 返回更新后的角色信息 `RoleGetOutputDto`

---

### 为角色分配菜单

**接口**: `POST /api/app/rbac/role/{id}/menu`

**描述**: 为角色分配菜单权限。

**权限**: `system:role:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 角色ID |

**请求参数**:

```typescript
interface AssignMenuInput {
  menuIds: Guid[];  // 菜单ID列表
}
```

**响应**: `204 No Content`

---

### 设置角色数据权限

**接口**: `POST /api/app/rbac/role/{id}/dept`

**描述**: 设置角色的数据权限范围，当数据权限为自定义时指定部门。

**权限**: `system:role:edit`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| id | Guid | 角色ID |

**请求参数**:

```typescript
interface UpdateDataScopeInput {
  roleId: Guid;           // 角色ID
  dataScope: number;      // 数据权限（1=全部，2=自定义，3=本部门，4=本部门及以下，5=仅本人）
  deptIds?: Guid[];       // 部门ID列表（dataScope=2时必填）
}
```

**响应**: `204 No Content`

---

### 获取角色下的用户

**接口**: `GET /api/app/rbac/role/{roleId}/user`

**描述**: 获取指定角色下的用户列表。

**权限**: `system:role:query`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| roleId | Guid | 角色ID |
| isAllocated | boolean | 是否已分配给该角色（查询参数） |
| skipCount | number | 跳过记录数（查询参数） |
| maxResultCount | number | 每页记录数（查询参数） |

**响应**: 返回用户分页列表 `PagedResultDto<UserGetListOutputDto>`

---

### 为用户分配角色（反向操作）

**接口**: `POST /api/app/rbac/role/auth-user`

**描述**: 为用户分配角色。

**权限**: `system:role:edit`

**请求参数**:

```typescript
interface RoleAuthUserCreateOrDeleteInput {
  userId: Guid;           // 用户ID
  roleIds: Guid[];       // 角色ID列表
}
```

**响应**: `204 No Content`

---

### 取消用户角色分配

**接口**: `DELETE /api/app/rbac/role/auth-user`

**描述**: 取消用户的角色分配。

**权限**: `system:role:edit`

**请求参数**: 同 `RoleAuthUserCreateOrDeleteInput`

**响应**: `204 No Content`

---

## 菜单管理

### 获取菜单列表

**接口**: `GET /api/app/rbac/menu`

**描述**: 获取菜单列表，支持分页和条件查询。

**权限**: `system:menu:list`

**请求参数**:

```typescript
interface MenuGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  menuName?: string;            // 菜单名称（模糊查询）
  state?: boolean;             // 状态（true=启用，false=禁用）
  menuSource?: number;         // 菜单来源（0=后台，1=纯前端路由）
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 50,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "menuName": "系统管理",
        "menuType": 0,
        "menuTypeName": "目录",
        "parentId": null,
        "path": "/system",
        "component": null,
        "orderNum": 1,
        "icon": "system",
        "state": true,
        "menuSource": 0,
        "isHide": false,
        "isKeepAlive": true,
        "isAffix": false,
        "permission": "system:manage",
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 创建菜单

**接口**: `POST /api/app/rbac/menu`

**描述**: 创建新菜单。

**权限**: `system:menu:add`

**请求参数**:

```typescript
interface MenuCreateInputVo {
  menuName: string;           // 菜单名称（必填）
  menuType: number;           // 菜单类型（0=目录，1=菜单，2=按钮）
  parentId?: Guid;            // 父菜单ID
  path?: string;              // 路由路径
  component?: string;         // 组件路径
  orderNum?: number;          // 排序号
  icon?: string;              // 图标
  state?: boolean;            // 状态
  menuSource?: number;        // 菜单来源（0=后台，1=纯前端路由）
  isHide?: boolean;           // 是否隐藏
  isKeepAlive?: boolean;      // 是否缓存
  isAffix?: boolean;          // 是否固定
  permission?: string;        // 权限标识
}
```

**响应**: 返回创建的菜单信息 `MenuGetOutputDto`

---

### 更新菜单

**接口**: `PUT /api/app/rbac/menu/{id}`

**描述**: 更新菜单信息。

**权限**: `system:menu:edit`

**请求参数**: 同 `MenuCreateInputVo`

**响应**: 返回更新后的菜单信息 `MenuGetOutputDto`

---

### 获取角色菜单

**接口**: `GET /api/app/rbac/menu/role/{roleId}`

**描述**: 获取指定角色的菜单列表，用于前端路由生成。

**权限**: `system:menu:query`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| roleId | Guid | 角色ID |

**响应**: 返回菜单列表 `List<MenuGetListOutputDto>`

---

## 部门管理

### 获取部门列表

**接口**: `GET /api/app/rbac/dept`

**描述**: 获取部门列表，支持分页和条件查询。

**权限**: `system:dept:list`

**请求参数**:

```typescript
interface DeptGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  deptName?: string;           // 部门名称（模糊查询）
  state?: boolean;             // 状态（true=启用，false=禁用）
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 10,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "deptName": "技术部",
        "parentId": null,
        "orderNum": 1,
        "state": true,
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 创建部门

**接口**: `POST /api/app/rbac/dept`

**描述**: 创建新部门。

**权限**: `system:dept:add`

**请求参数**:

```typescript
interface DeptCreateInputVo {
  deptName: string;           // 部门名称（必填）
  parentId?: Guid;            // 父部门ID
  orderNum?: number;          // 排序号
  state?: boolean;            // 状态
}
```

**响应**: 返回创建的部门信息 `DeptGetOutputDto`

---

### 获取角色部门

**接口**: `GET /api/app/rbac/dept/role/{roleId}`

**描述**: 获取指定角色的部门列表（用于数据权限）。

**权限**: `system:dept:query`

**路径参数**:

| 参数 | 类型 | 说明 |
|-----|------|------|
| roleId | Guid | 角色ID |

**响应**: 返回部门列表 `List<DeptGetListOutputDto>`

---

## 岗位管理

### 获取岗位列表

**接口**: `GET /api/app/rbac/post`

**描述**: 获取岗位列表，支持分页和条件查询。

**权限**: `system:post:list`

**请求参数**:

```typescript
interface PostGetListInputVo {
  skipCount: number;           // 跳过记录数（必填）
  maxResultCount: number;      // 每页记录数（必填）
  postName?: string;           // 岗位名称（模糊查询）
  state?: boolean;             // 状态（true=启用，false=禁用）
  sorting?: string;            // 排序字段
}
```

**响应示例**:
```json
{
  "result": {
    "totalCount": 15,
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "postCode": "CTO",
        "postName": "技术总监",
        "orderNum": 1,
        "state": true,
        "creationTime": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 创建岗位

**接口**: `POST /api/app/rbac/post`

**描述**: 创建新岗位。

**权限**: `system:post:add`

**请求参数**:

```typescript
interface PostCreateInputVo {
  postCode: string;           // 岗位编码（必填，唯一）
  postName: string;           // 岗位名称（必填）
  orderNum?: number;          // 排序号
  state?: boolean;            // 状态
}
```

**响应**: 返回创建的岗位信息 `PostGetOutputDto`

---

### 更新岗位

**接口**: `PUT /api/app/rbac/post/{id}`

**描述**: 更新岗位信息。

**权限**: `system:post:edit`

**请求参数**: 同 `PostCreateInputVo`

**响应**: 返回更新后的岗位信息 `PostGetOutputDto`

---

## 数据结构

### 用户数据结构

```typescript
interface UserGetOutputDto {
  id: Guid;                    // 用户ID
  userName: string;            // 用户名
  nickName: string;            // 昵称
  email?: string;              // 邮箱
  phone?: string;              // 手机号
  sex?: number;                // 性别（1=男，2=女）
  deptId?: Guid;               // 部门ID
  deptName?: string;           // 部门名称
  postId?: Guid;               // 岗位ID
  postName?: string;           // 岗位名称
  state: boolean;              // 状态
  icon?: string;               // 头像（Base64）
  orderNum?: number;           // 排序号
  creationTime: Date;          // 创建时间
}
```

### 角色数据结构

```typescript
interface RoleGetOutputDto {
  id: Guid;                    // 角色ID
  roleCode: string;            // 角色编码
  roleName: string;            // 角色名称
  state: boolean;              // 状态
  dataScope: number;           // 数据权限（1=全部，2=自定义，3=本部门，4=本部门及以下，5=仅本人）
  dataScopeName?: string;      // 数据权限名称
  orderNum?: number;           // 排序号
  creationTime: Date;          // 创建时间
}
```

### 菜单数据结构

```typescript
interface MenuGetOutputDto {
  id: Guid;                    // 菜单ID
  menuName: string;            // 菜单名称
  menuType: number;            // 菜单类型（0=目录，1=菜单，2=按钮）
  menuTypeName: string;        // 菜单类型名称
  parentId?: Guid;             // 父菜单ID
  path?: string;               // 路由路径
  component?: string;          // 组件路径
  orderNum?: number;           // 排序号
  icon?: string;               // 图标
  state: boolean;              // 状态
  menuSource: number;          // 菜单来源（0=后台，1=纯前端路由）
  isHide: boolean;             // 是否隐藏
  isKeepAlive: boolean;        // 是否缓存
  isAffix: boolean;            // 是否固定
  permission?: string;         // 权限标识
  creationTime: Date;          // 创建时间
}
```

---

## 前端使用示例

### 获取用户列表

```typescript
const response = await fetch('/api/app/rbac/user?skipCount=0&maxResultCount=10', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const data = await response.json();
console.log(data.result.items);  // 用户列表
console.log(data.result.totalCount);  // 总记录数
```

### 创建用户

```typescript
const response = await fetch('/api/app/rbac/user', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    userName: 'testuser',
    nickName: '测试用户',
    password: '123456',
    email: 'test@example.com'
  })
});

const user = await response.json();
console.log(user.result);  // 创建的用户信息
```

### 为用户分配角色

```typescript
const response = await fetch(`/api/app/rbac/user/${userId}/role`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    roleIds: ['role-id-1', 'role-id-2']
  })
});
```

---

## 相关文档

- [[AdminAPI索引]] - RBAC 管理端 API 总览
- [[系统配置API]] - 系统配置与日志 API 文档
- [[监控API]] - 监控与 SignalR Hub API 文档
- [[权限系统设计]] - 权限系统设计文档
- [[数据权限设计]] - 数据权限实现文档
