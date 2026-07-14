# 待办事项应用 - 实现文档

## 技术概述

纯前端单文件应用，由 **HTML + CSS + 原生 JavaScript** 构成，无需框架和构建工具，浏览器直接打开即可运行。

## 项目结构

```
todo-app/
└── index.html          # 唯一文件，包含全部代码
    ├── <style>         # CSS 样式
    ├── <body>          # HTML 结构
    └── <script>        # JavaScript 逻辑
```

---

## 一、数据模型

每个任务是一个 JavaScript 对象，结构如下：

```js
{
  id: "m5f8k3x2a1b2",   // 唯一标识，时间戳36进制 + 随机字符串
  text: "完成周报",       // 任务内容
  completed: false,      // 是否已完成
  priority: "high",      // 优先级: high | medium | low
  category: "work"       // 分类: work | personal | study | life | other
}
```

所有任务存储在 `tasks` 数组中（`tasks[]`），新任务用 `unshift` 插入数组头部，保证最新的显示在最上面。

## 二、数据持久化

使用浏览器内置的 `localStorage` 实现数据持久化：

```
保存: JSON.stringify(tasks) → localStorage.setItem("todo-app-tasks", json)
加载: localStorage.getItem("todo-app-tasks") → JSON.parse(json) → tasks[]
```

- 刷新页面不丢失数据
- 兼容旧版本数据（缺少 `priority`/`category` 字段时自动补充默认值）
- JSON 解析失败时自动降级为空数组

## 三、核心架构

### 单向数据流

```
用户操作 → 修改 tasks 数组 → saveTasks() 持久化 → render() 重新绘制界面
```

所有数据变化都遵循这个流程：先改数据，再存盘，最后渲染。

### 渲染函数（render）

`render()` 是核心函数，负责把 `tasks` 数据转换成页面上的 DOM：

```
render() 执行流程:
  1. 根据 searchQuery 过滤 tasks（大小写不敏感的模糊匹配）
  2. 判断三种情况:
     - 无任务 → 显示空状态引导页
     - 有任务但全被过滤 → 显示"没有找到匹配的任务"
     - 有匹配任务 → 逐个调用 taskItemHTML() 生成 HTML 并拼接到列表容器
  3. 调用 updateStats() 更新统计数字
```

### 任务卡片生成（taskItemHTML）

两种渲染模式：

| 模式 | 条件 | 显示内容 |
|------|------|----------|
| 普通模式 | `editingId !== task.id` | 复选框 + 优先级徽章 + 分类徽章 + 文字 + 编辑/删除按钮 |
| 编辑模式 | `editingId === task.id` | 文本输入框 + 保存/取消按钮 |

## 四、功能实现细节

### 1. 添加任务

```
用户点击"添加"或按回车
  → 取输入框文字 + 优先级下拉值 + 分类下拉值
  → 构建任务对象，unshift 到 tasks 头部
  → 保存 + 渲染 + 清空输入框并聚焦
```

### 2. 完成状态切换

```
点击复选框 → toggleTask(id)
  → find 找到对应任务 → completed 取反
  → 保存 + 渲染
```

### 3. 删除任务

```
点击删除按钮 → deleteTask(id)
  → filter 过滤掉该任务（创建新数组，原数组不变）
  → 保存 + 渲染
```

### 4. 内联编辑

```
点击编辑按钮 → startEdit(id)
  → 设置 editingId → 渲染为编辑模式 → 输入框自动聚焦，光标移到末尾
  → 按回车/点击保存 → saveEdit(id, value) → 更新 task.text → 清除 editingId → 渲染
  → 按 Esc/点击取消 → cancelEdit() → 清除 editingId → 渲染
```

### 5. 任务搜索

```
搜索框输入 → searchInput 'input' 事件
  → 更新 searchQuery → 调用 render()
  → render 内将任务文字和搜索词都转小写，用 includes() 做模糊匹配
```

特点是**实时过滤**，每输入一个字符就立即更新列表。

### 6. 任务分类

```
下拉框选择分类
  → 添加任务时读取 categorySelect.value 存入任务对象
  → 渲染时从 CATEGORY_LABELS 映射表读取中文显示名
  → CSS 类名匹配不同颜色（.category-badge.work = 蓝色, .personal = 粉色...）
```

### 7. 导出功能

```
点击"导出任务"
  → 生成时间戳（如 2026-07-14-15-30）
  → 逐行构建文本内容:
     标题 → 分隔线 → 每个任务的序号/状态/优先级/分类/文字 → 统计
  → 创建 Blob 对象（带 \uFEFF BOM，确保记事本正确识别 UTF-8）
  → 创建隐藏 <a> 标签，href 设为 Blob URL
  → 模拟点击触发浏览器下载
  → 清理临时元素和 URL
```

### 8. 确认弹窗

```
点击"全部清空" → showConfirm(title, desc, callback)
  → 动态创建遮罩层 <div.overlay> + 弹窗 <div.dialog>
  → 插入 body，显示弹窗
  → "取消" / 点击背景 → 移除遮罩
  → "确认" → 移除遮罩 + 执行 callback（清空 tasks）
```

### 9. 事件委托

任务列表容器上只绑定**一个** `click` 事件，通过 `e.target.closest()` 判断用户点击了哪个按钮：

```
点击 .btn-delete  → deleteTask
点击 .checkbox    → toggleTask
点击 .btn-edit    → startEdit
点击 .btn-save    → saveEdit
点击 .btn-cancel  → cancelEdit
```

这样做的好处：无论任务列表怎么变化，新增的任务不需要重新绑定事件。

### 10. 键盘快捷键

| 快捷键 | 场景 | 作用 |
|--------|------|------|
| `Enter` | 输入框聚焦 | 添加任务 |
| `Enter` | 编辑模式 | 保存编辑 |
| `Esc` | 编辑模式 | 取消编辑 |

## 五、CSS 设计

- **CSS 变量（`:root`）**：统一管理颜色、圆角、阴影，便于维护
- **渐变背景**：`linear-gradient` + `::before/::after` 装饰圆
- **动画**：`slideIn`（任务出现）、`fadeIn`（空状态）、`scaleIn`（弹窗）
- **响应式**：`@media (max-width: 540px)` 适配手机屏幕
- **组件化命名**：`.card`、`.btn-add`、`.task-item` 等语义化类名

## 六、安全措施

- **XSS 防护**：`escapeHTML()` 函数利用浏览器 DOM API 自动转义 `<>"&` 等特殊字符
- **数据校验**：`maxlength="200"` 限制输入长度，空内容拒绝添加
