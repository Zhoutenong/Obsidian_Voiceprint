# DisplayComponentEntity

显示组件实体，定义数据在前端界面的展示组件配置。

## 表名与主键

- **表名**: `ast_display_component`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键ID | - |
| `key` | string | 组件标识（前端组件key） | - |
| `name` | string | 显示名称 | - |
| `description` | string | 描述 | - |
| `parameters` | text | 组件参数（JSON格式） | - |

## 关联实体（ER关系）

### 关联到当前实体

- **DataBindingTypeAggregateRoot** (1:N)
  - 通过 `display_component_id` 关联
  - 一个显示组件可以被多个绑定类型使用

## 被哪些服务读写

### 写入服务

- **SubstationDataSeed** (数据种子初始化)
  - `InitializeDisplayComponents()` - 初始化预设显示组件配置

- **DisplayComponentAppService** (推断)
  - 创建/更新/删除显示组件

### 读取服务

- **DisplayComponentAppService** (推断)
  - 获取显示组件列表
  - 获取显示组件详情

- **前端渲染服务** (推断)
  - 根据组件配置渲染对应的UI组件

## 业务规则约束

1. **组件标识唯一性**
   - `key` 字段必须唯一
   - 对应前端框架中的组件标识

2. **参数格式**
   - `parameters` 必须是有效的 JSON 格式
   - 参数结构由前端组件定义

3. **命名规范**
   - `name` 应该清晰描述组件用途
   - 建议使用中文描述

4. **组件可复用**
   - 同一显示组件可被多个绑定类型使用
   - 参数配置可以自定义组件行为

## 常见显示组件类型

### 1. 仪表盘组件 (Gauge)
用于显示温度、湿度等连续数值：

```json
{
  "key": "gauge-component",
  "name": "仪表盘",
  "description": "用于显示温度、湿度等连续数值的仪表盘组件",
  "parameters": {
    "min": 0,
    "max": 100,
    "unit": "℃",
    "threshold_high": 80,
    "threshold_low": 10,
    "colors": ["#00ff00", "#ffff00", "#ff0000"]
  }
}
```

### 2. 图表组件 (Chart)
用于显示振动、电流等时序数据：

```json
{
  "key": "chart-component",
  "name": "折线图",
  "description": "用于显示时序数据的折线图组件",
  "parameters": {
    "chart_type": "line",
    "x_axis_type": "time",
    "y_axis_unit": "mA",
    "refresh_interval": 1000,
    "max_data_points": 100
  }
}
```

### 3. 状态指示器组件 (StatusIndicator)
用于显示开关状态：

```json
{
  "key": "status-indicator",
  "name": "状态指示器",
  "description": "用于显示开关、告警等状态",
  "parameters": {
    "type": "switch",
    "on_text": "合闸",
    "off_text": "分闸",
    "on_color": "#00ff00",
    "off_color": "#ff0000"
  }
}
```

### 4. 音频播放器组件 (AudioPlayer)
用于播放声纹数据：

```json
{
  "key": "audio-player",
  "name": "音频播放器",
  "description": "用于播放和分析声纹数据的音频播放器",
  "parameters": {
    "autoplay": false,
    "loop": false,
    "show_waveform": true,
    "show_spectrum": true,
    "enable_download": true
  }
}
```

### 5. 表格组件 (Table)
用于显示多列数据：

```json
{
  "key": "table-component",
  "name": "数据表格",
  "description": "用于显示多列数据的表格组件",
  "parameters": {
    "columns": [
      {"field": "name", "title": "名称", "width": 120},
      {"field": "value", "title": "数值", "width": 100},
      {"field": "unit", "title": "单位", "width": 80},
      {"field": "time", "title": "时间", "width": 150}
    ],
    "pagination": true,
    "page_size": 20
  }
}
```

### 6. 波形图组件 (Waveform)
用于显示振动波形：

```json
{
  "key": "waveform-component",
  "name": "波形图",
  "description": "用于显示振动、超声波等波形数据",
  "parameters": {
    "amplitude_scale": 1.0,
    "time_scale": 1.0,
    "show_grid": true,
    "show_cursor": true,
    "enable_zoom": true
  }
}
```

### 7. 数值显示组件 (ValueDisplay)
用于显示单个数值：

```json
{
  "key": "value-display",
  "name": "数值显示",
  "description": "简单的数值显示组件",
  "parameters": {
    "font_size": 24,
    "font_color": "#333333",
    "show_unit": true,
    "show_trend": true,
    "decimal_places": 2
  }
}
```

## 数据示例

典型的显示组件配置：

```csharp
// 温度仪表盘
{
  "Id": "guid-1",
  "Key": "gauge-component",
  "Name": "温度仪表盘",
  "Description": "用于显示温度数据的仪表盘",
  "Parameters": "{\"min\":0,\"max\":100,\"unit\":\"℃\",\"colors\":[\"#00ff00\",\"#ffff00\",\"#ff0000\"]}"
}

// 声纹播放器
{
  "Key": "audio-player",
  "Name": "声纹播放器",
  "Description": "声纹数据播放器",
  "Parameters": "{\"show_waveform\":true,\"show_spectrum\":true,\"enable_download\":true}"
}

// 振动波形图
{
  "Key": "waveform-component",
  "Name": "振动波形图",
  "Description": "振动数据波形显示",
  "Parameters": "{\"amplitude_scale\":1.0,\"show_grid\":true,\"enable_zoom\":true}"
}
```

## 组件与绑定类型的关系

| 组件类型 | 适用绑定类型 | 典型场景 |
|----------|--------------|----------|
| 仪表盘 | 温度、湿度监测 | 实时数值监控 |
| 折线图 | 振动、电流监测 | 时序数据展示 |
| 状态指示器 | 分闸、合闸监测 | 开关状态显示 |
| 音频播放器 | 声纹监测 | 声纹回放和分析 |
| 波形图 | 超声波、振动监测 | 波形分析 |
| 数值显示 | 所有数值类型 | 简单数值展示 |
| 表格 | 多属性监测 | 综合数据显示 |

## 索引建议

- `key` - 用于按组件标识查询（建议唯一索引）

## 注意事项

1. **前端实现依赖**
   - `key` 必须与前端组件注册的标识一致
   - 前端必须实现对应的组件

2. **参数验证**
   - 参数 JSON 格式必须正确
   - 应根据组件类型验证参数结构

3. **组件版本管理**
   - 建议记录组件版本信息
   - 支持组件升级和兼容性处理

4. **性能考虑**
   - 复杂组件（如波形图）可能影响性能
   - 考虑组件懒加载和虚拟化

5. **扩展性**
   - 新增显示组件需要：
     1. 在前端实现新组件
     2. 创建对应的组件配置记录
     3. 在绑定类型中关联新组件

6. **用户体验**
   - 组件配置应考虑用户体验
   - 提供合理的默认参数
   - 支持主题定制
