# 后台菜单管理 - 前端到后端全链路分析

---

## 一、总体架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端 (soccer_v3)                          │
│                                                                   │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐ │
│  │ menuMange/   │   │ auth Store   │   │ dynamicRouter.ts     │ │
│  │ index.vue    │   │ (菜单权限)    │   │ (动态路由注册)        │ │
│  │ (管理页面)    │   │              │   │                      │ │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬───────────┘ │
│         │                  │                       │             │
│  ┌──────┴──────────────────┴───────────────────────┴───────────┐ │
│  │                    API 层 (HTTP Client)                       │ │
│  │  menu.ts:  /v1/menu/list|add|edit|delete                     │ │
│  │  login.ts: /v1/auth/menus|buttons                            │ │
│  └──────────────────────────┬───────────────────────────────────┘ │
└─────────────────────────────┼────────────────────────────────────┘
                              │ HTTP Request
┌─────────────────────────────┼────────────────────────────────────┐
│                        后端 (soccer_php)                          │
│                              │                                    │
│  ┌───────────────────────────┴──────────────────────────────┐   │
│  │                    Laravel 路由 (routes/api/v1.php)        │   │
│  │  /v1/auth/menus    → MenuController@authMenus              │   │
│  │  /v1/auth/buttons  → MenuController@authButtons            │   │
│  │  /v1/menu/list     → MenuController@list                   │   │
│  │  /v1/menu/add      → MenuController@add                    │   │
│  │  /v1/menu/edit     → MenuController@edit                   │   │
│  │  /v1/menu/delete   → MenuController@delete                 │   │
│  └───────────────────────────┬──────────────────────────────┘   │
│                              │                                    │
│  ┌───────────────────────────┴──────────────────────────────┐   │
│  │                  MenuService (业务逻辑层)                   │   │
│  │  authMenus() / authButtons() / list() / CRUD()             │   │
│  └───────────────┬───────────────────────┬───────────────────┘   │
│                  │                       │                        │
│  ┌───────────────┴───────┐  ┌────────────┴──────────────────┐   │
│  │   menus 表             │  │  Spatie permissions 表         │   │
│  │   (菜单/按钮元数据)     │  │  (RBAC 权限标识)               │   │
│  │   parent_id 自引用树    │  │  role_has_permissions 中间表   │   │
│  └───────────────────────┘  └───────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 二、数据库设计

### 2.1 `menus` 表结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint (PK) | 主键 |
| `parent_id` | foreignId (FK→menus) | 上级菜单，级联删除 |
| `type` | varchar(20) | 节点类型：`menu` / `button` |
| `title` | varchar(80) | 显示名称（如"用户管理"） |
| `name` | varchar(120) | 前端路由名称（如 `userManage`） |
| `path` | varchar(255) | 前端访问路径（如 `/system/userManage`） |
| `component` | varchar(255) | 前端组件路径（如 `/system/accountManage/index`） |
| `redirect` | varchar(255) | 默认重定向路径 |
| `icon` | varchar(80) | Element Plus 图标名称 |
| `permission` | varchar(160) UNIQUE | 权限唯一标识（如 `menu:system:user`） |
| `button_code` | varchar(80) | 按钮编码（如 `add`, `edit`, `delete`） |
| `sort` | unsignedInteger | 同级排序权重 |
| `status` | unsignedTinyInteger | 0=禁用, 1=启用 |
| `is_link` | varchar(255) | 外部链接地址 |
| `is_hide` | boolean | 侧边栏隐藏 |
| `is_full` | boolean | 全屏布局 |
| `is_affix` | boolean | 标签栏固定 |
| `is_keep_alive` | boolean | 页面缓存 |

**索引**：`(parent_id, type, sort)` 联合索引 + `(status, type)` 联合索引

### 2.2 Spatie RBAC 权限表

- `permissions` — 权限标识（name + guard_name 唯一）
- `roles` — 角色
- `model_has_permissions` — 用户↔权限 多态关联
- `model_has_roles` — 用户↔角色 多态关联
- `role_has_permissions` — 角色↔权限 中间表

### 2.3 双层架构设计理念

菜单管理采用 **menus 表 + Spatie RBAC** 双层架构：

1. `menus` 表：存储菜单/按钮的完整元数据（路由、组件、图标、排序等），以 `parent_id` 自引用形成树
2. Spatie `permissions` 表：存储权限标识，通过 `role_has_permissions` 关联角色
3. **增删改菜单时**，`MenuService` 自动同步 Spatie 的 `permissions` 表
4. **用户登录后**，通过角色关联的权限来筛选可访问的菜单

---

## 三、后端完整链路

### 3.1 路由定义

```php
// routes/api/v1.php

// 用户认证时获取菜单和按钮
Route::prefix('auth')->as('auth.')->group(function () {
    Route::get('/menus', [MenuController::class, 'authMenus']);
    Route::get('/buttons', [MenuController::class, 'authButtons']);
});

// 菜单管理 CRUD
Route::prefix('menu')->as('menu.')->group(function () {
    Route::get('/list', [MenuController::class, 'list']);
    Route::post('/add', [MenuController::class, 'add']);
    Route::post('/edit', [MenuController::class, 'edit']);
    Route::post('/delete', [MenuController::class, 'delete']);
});
```

### 3.2 MenuController 控制器

```
MenuController
├── authMenus()    → GET  /v1/auth/menus    → 返回用户可访问的动态路由树
├── authButtons()  → GET  /v1/auth/buttons  → 返回用户按钮权限（按路由名分组）
├── list()         → GET  /v1/menu/list     → 返回管理用递归树（全部节点）
├── add()          → POST /v1/menu/add      → 创建菜单/按钮 + 同步权限
├── edit()         → POST /v1/menu/edit     → 更新菜单/按钮 + 同步权限
└── delete()       → POST /v1/menu/delete   → 删除节点及所有后代 + 清理权限
```

### 3.3 MenuService 核心业务逻辑

#### 3.3.1 用户动态菜单获取 (`authMenus`)

```
用户请求 /v1/auth/menus
         │
         ▼
  获取用户所有权限标识 (Spatie)
         │
         ▼
  查询 menus 表中 permission 匹配的 menu 类型节点
         │
         ▼
  收集所有父级 ID，补全上级菜单（确保树完整）
         │
         ▼
  重新查询：权限匹配 OR 父级 ID 匹配的 menu 节点
         │
         ▼
  buildRouterTree() 递归构建路由树
         │
         ▼
  toRouterArray() 转换为前端格式
  {
    path, name,
    meta: { icon, title, isLink, isHide, isFull, isAffix, isKeepAlive },
    component?, redirect?,
    children?: [...]
  }
```

#### 3.3.2 用户按钮权限获取 (`authButtons`)

```
用户请求 /v1/auth/buttons
         │
         ▼
  获取用户所有权限标识
         │
         ▼
  查询 menus 表中 type=button 且 permission 匹配的节点
         │
         ▼
  按父级菜单的 name 字段分组
         │
         ▼
  返回 { "路由名": ["add", "edit", "delete"], ... }
```

#### 3.3.3 菜单 CRUD 操作

**创建 (`create`)**：
1. 调用 `withDefaults()` 补齐默认值（sort=0, status=1, is_keep_alive=true 等）
2. 根据 type 互斥清理字段（button 类型清除路由字段，menu 类型清除 button_code）
3. `Menu::create()` 写入数据库
4. `Permission::findOrCreate()` 同步 Spatie 权限表

**更新 (`update`)**：
1. 查找菜单节点（不存在抛异常）
2. 记录旧权限标识 `oldPermission`
3. 更新菜单字段
4. 同步新权限到 Spatie
5. 若权限标识变化，删除旧权限标识

**删除 (`delete`)**：
1. 校验 ID 存在性
2. `withDescendantIds()` 递归收集所有后代 ID（循环查询直到无新子节点）
3. 收集所有待删除节点的 permission 标识
4. 批量删除 menus 记录
5. 批量删除 Spatie permissions 记录

#### 3.3.4 递归树构建

两个构建方法，结构相同：

```
buildRouterTree / buildManageTree

输入: 全部节点集合 + parentId (默认 null)
逻辑:
  1. 过滤出 parent_id === parentId 的节点
  2. 按 sort, id 排序
  3. 对每个节点:
     - 调用 toRouterArray() / toManageArray() 转换数据
     - 递归调用自身获取 children
     - 有 children 则附加
  4. 返回数组
```

### 3.4 Menu 模型关键方法

| 方法 | 说明 |
|------|------|
| `scopeEnabled()` | 只查 status=1 |
| `scopeMenus()` | 只查 type='menu' |
| `scopeButtons()` | 只查 type='button' |
| `toRouterArray()` | 转为前端路由格式（含 meta 对象） |
| `toManageArray()` | 转为管理页面格式（含 meta + 扁平字段） |
| `parent()` | BelongsTo 自引用上级 |
| `children()` | HasMany 自引用下级（按 sort, id 排序） |

---

## 四、前端完整链路

### 4.1 两条数据流

前端菜单有两条独立的数据流：

```
┌─────────────────────────────────────────────────────────────┐
│  数据流 1: 用户菜单权限（侧边栏 & 动态路由）                    │
│                                                               │
│  authStore.getAuthMenuList()                                  │
│    → GET /v1/auth/menus                                       │
│    → 存储到 authMenuList                                      │
│    → showMenuListGet (过滤隐藏) → 渲染侧边栏 SubMenu.vue       │
│    → flatMenuListGet (扁平化)   → 动态注册路由                 │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│  数据流 2: 菜单管理 CRUD（后台管理页面）                        │
│                                                               │
│  menuManageStore.fetchTree()                                  │
│    → GET /v1/menu/list                                        │
│    → 存储到 tree                                              │
│    → 渲染树形表格                                              │
│    → parentOptions 计算属性 → 父级选择器                       │
│                                                               │
│  menuManageStore.createMenu()  → POST /v1/menu/add            │
│  menuManageStore.updateMenu()  → POST /v1/menu/edit           │
│  menuManageStore.removeMenu()  → POST /v1/menu/delete         │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 用户登录 → 动态路由初始化流程

```
用户登录成功
    │
    ▼
router.beforeEach 路由守卫触发
    │
    ▼
initDynamicRouter()  // routers/modules/dynamicRouter.ts
    │
    ├── authStore.getAuthMenuList()     → GET /v1/auth/menus
    ├── authStore.getAuthButtonList()   → GET /v1/auth/buttons
    │
    ├── 遍历 flatMenuListGet:
    │   ├── 删除 children（扁平化后不需要）
    │   ├── 将 component 路径映射为动态 import:
    │   │   item.component = modules["/src/views" + item.component + ".vue"]
    │   └── 注册路由:
    │       ├── isFull=true  → router.addRoute(item)  // 顶级路由
    │       └── isFull=false → router.addRoute("layout", item)  // 布局子路由
    │
    ▼
用户看到侧边栏菜单 & 可访问对应页面
```

### 4.3 菜单管理页面交互流程

```
menuMange/index.vue 组件挂载
    │
    ▼
onMounted → menuStore.fetchTree() → GET /v1/menu/list
    │
    ▼
el-table 渲染树形表格 (tree-props: { children: 'children' })
  - 显示名称 + 图标 + 类型标签（菜单/按钮）
  - 路由名称、路径/按钮编码、组件路径、权限标识
  - 排序、状态
  - 操作按钮：新增子项 / 编辑 / 删除

────────────────────────────────────

【新增菜单】
  点击"新增菜单"按钮 → openCreate()
    → 表单类型默认 "menu"
    → 填写：类型、上级菜单、名称、权限标识、路由名、路径、组件、图标、排序等
    → submitForm()
      → 校验表单
      → 清理无关字段（button 清除路由字段，menu 清除 button_code）
      → menuStore.createMenu(payload) → POST /v1/menu/add
      → 刷新树 menuStore.fetchTree()

【新增按钮】
  在某菜单行点击"新增子项" → openCreate(row)
    → 表单类型默认 "button"，parent_id 预设
    → 填写：显示名称、权限标识、按钮编码
    → submitForm() → 同上流程

【编辑】
  点击"编辑" → openEdit(row)
    → 将 meta 字段展开到表单（isHide → is_hide 等）
    → submitForm() → menuStore.updateMenu(payload) → POST /v1/menu/edit

【删除】
  点击"删除" → removeMenu(row)
    → 确认弹窗（提示级联删除影响）
    → menuStore.removeMenu(id) → POST /v1/menu/delete
    → 刷新树
```

### 4.4 侧边栏菜单渲染

```
SubMenu.vue 递归组件
    │
    ├── 遍历 showMenuListGet (已过滤隐藏菜单)
    │
    ├── 无 children → 渲染 el-menu-item
    │   └── 有 isLink → 外链跳转
    │   └── 无 isLink → router-link 内部跳转
    │
    └── 有 children → 渲染 el-sub-menu
        └── 递归调用 SubMenu 渲染子级
```

### 4.5 核心文件清单

| 文件 | 职责 |
|------|------|
| `src/views/system/menuMange/index.vue` | 菜单管理主页面（树形表格 + 新增/编辑弹窗） |
| `src/api/modules/menu.ts` | 菜单 CRUD API 调用封装 |
| `src/api/modules/login.ts` | 认证 API（含获取菜单/按钮权限） |
| `src/api/interface/system.ts` | `MenuItem`、`MenuSaveRequest` 类型定义 |
| `src/stores/modules/menuManage.ts` | 菜单管理 Pinia Store（tree + loading + CRUD） |
| `src/stores/modules/auth.ts` | 认证 Store（authMenuList + authButtonList） |
| `src/routers/modules/dynamicRouter.ts` | 动态路由注册逻辑 |
| `src/routers/index.ts` | 路由守卫（触发动态路由初始化） |
| `src/layouts/components/Menu/SubMenu.vue` | 递归侧边栏菜单组件 |
| `src/utils/index.ts` | 菜单工具函数（扁平化、过滤、面包屑） |

---

## 五、关键设计要点

### 5.1 菜单与按钮的统一存储

菜单和按钮共用同一张 `menus` 表，通过 `type` 字段区分。这样的好处是：
- 统一的树形结构管理（按钮作为菜单的子节点）
- 权限标识统一存储在 `permission` 字段
- CRUD 逻辑可复用

### 5.2 菜单与权限的自动同步

增删改菜单时，`MenuService` 自动维护 Spatie `permissions` 表：
- 新增菜单 → `Permission::findOrCreate(permission, 'api')`
- 更新菜单 → 若 permission 变化，删旧建新
- 删除菜单 → 递归删除所有后代的 permissions

### 5.3 前端两种数据格式

同一个 Menu 模型通过两个方法输出不同格式：

- `toRouterArray()`：供动态路由使用，字段精简（path, name, component, redirect, meta）
- `toManageArray()`：供管理页面使用，保留所有字段（含 id, parent_id, permission 等管理字段）+ meta 对象

### 5.4 表单字段的互斥处理

前端和后端都做了类型互斥：
- 按钮类型：清除 name/path/component/redirect 路由字段
- 菜单类型：清除 button_code 字段

### 5.5 级联删除

删除菜单节点时：
1. 后端 `withDescendantIds()` 递归查询所有后代
2. 前端确认弹窗明确提示级联影响
3. 批量删除 menus + permissions

### 5.6 父级菜单选项

前端 `parentOptions` 计算属性：
- 递归遍历 `menuStore.tree`
- 只显示 `type='menu'` 的节点（按钮不能作为父级）
- 排除自身 ID（防止循环引用）
- 使用全角空格缩进表示层级深度

---

## 六、API 接口速查表

| 方法 | 路径 | 功能 | 请求参数 | 响应格式 |
|------|------|------|----------|----------|
| GET | `/v1/auth/menus` | 获取用户动态菜单 | 无（通过 token 识别用户） | `[{path, name, meta, component?, children?}]` |
| GET | `/v1/auth/buttons` | 获取用户按钮权限 | 无 | `{路由名: ["add","edit",...]}` |
| GET | `/v1/menu/list` | 获取菜单管理树 | 无 | `[{id, parent_id, type, title, ..., children?}]` |
| POST | `/v1/menu/add` | 新增菜单/按钮 | `{parent_id, type, title, permission, ...}` | 创建的菜单对象 |
| POST | `/v1/menu/edit` | 更新菜单/按钮 | `{id, ...}` | 更新后的菜单对象 |
| POST | `/v1/menu/delete` | 删除菜单/按钮 | `{id: [1, 2, ...]}` | `null` |
