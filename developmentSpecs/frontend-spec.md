# 前端开发规范

> 适用于 Vue 3 + TypeScript + Element Plus (Web) 和 UniApp (小程序) 技术栈。

## 目录

- [1. 项目结构规范](#1-项目结构规范)
- [2. 命名规范](#2-命名规范)
- [3. TypeScript 规范](#3-typescript-规范)
- [4. Vue 组件规范](#4-vue-组件规范)
- [5. 状态管理规范](#5-状态管理规范)
- [6. API 请求规范](#6-api-请求规范)
- [7. 样式规范](#7-样式规范)
- [8. 性能优化规范](#8-性能优化规范)
- [9. UniApp 小程序规范](#9-uniapp-小程序规范)
- [10. 代码质量规范](#10-代码质量规范)

---

## 1. 项目结构规范

### 1.1 Vue3 Web 项目结构

```
src/
├── api/                    # API 接口定义
│   ├── modules/            # 按模块划分
│   ├── request.ts          # Axios 封装
│   └── index.ts            # 统一导出
├── assets/                 # 静态资源
│   ├── icons/              # 图标
│   ├── images/             # 图片
│   └── styles/             # 全局样式
├── components/             # 公共组件
│   ├── common/             # 通用组件
│   └── business/           # 业务组件
├── composables/            # 组合式函数
├── constants/              # 常量定义
├── directives/             # 自定义指令
├── enums/                  # 枚举定义
├── hooks/                  # 自定义 Hooks
├── layouts/                # 布局组件
├── plugins/                # 插件
├── router/                 # 路由配置
├── stores/                 # Pinia 状态管理
├── types/                  # TypeScript 类型定义
├── utils/                  # 工具函数
├── views/                  # 页面视图
├── App.vue
└── main.ts
```

### 1.2 UniApp 小程序项目结构

```
src/
├── api/                    # API 接口
├── components/             # 公共组件（ls-前缀）
├── pages/                  # 页面
├── static/                 # 静态资源
├── stores/                 # Pinia 状态管理
├── types/                  # 类型定义
├── utils/                  # 工具函数
├── App.vue
├── main.ts
├── manifest.json           # 应用配置
├── pages.json              # 页面配置
└── uni.scss                # 全局样式变量
```

---

## 2. 命名规范

### 2.1 文件命名

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 组件文件 | PascalCase | `SeatPicker.vue` |
| 组合式函数 | camelCase + use 前缀 | `useLoading.ts` |
| 工具函数 | camelCase | `format.ts` |
| 类型文件 | camelCase | `user.ts` |
| 样式文件 | kebab-case | `variables.scss` |

### 2.2 变量命名

```typescript
// 【强制】使用 camelCase
const userName = 'zhangsan';

// 【强制】布尔值使用 is/has/can/should 前缀
const isLoading = ref(false);
const hasPermission = ref(true);

// 【强制】常量使用 UPPER_SNAKE_CASE
const MAX_RETRY_COUNT = 3;

// 【强制】事件处理函数使用 handle/on 前缀
const handleSubmit = () => { ... };

// 【强制】请求函数使用动词开头
const fetchUserList = async () => { ... };
```

### 2.3 组件命名

```typescript
// 【强制】组件名使用 PascalCase，至少两个单词
// 基础组件: BaseButton, BaseTable
// 业务组件: LsSeatMap, LsTimeSlotPicker
```

---

## 3. TypeScript 规范

### 3.1 类型定义

```typescript
// 【强制】优先使用 interface 定义对象类型
interface User {
  id: number;
  userName: string;
  status: UserStatus;
}

// 【强制】使用 type 定义联合类型
type UserStatus = 'active' | 'inactive' | 'banned';

// 【强制】API 响应类型定义
interface ApiResponse<T = any> {
  code: string;
  message: string;
  data: T;
}

// 【强制】枚举使用 const enum
const enum ReservationStatus {
  PENDING = 0,
  CONFIRMED = 1,
  CANCELLED = 2,
}
```

### 3.2 类型使用

```typescript
// 【强制】禁止使用 any，使用 unknown 代替
// 【强制】函数必须声明返回类型
function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// 【强制】使用泛型提高复用性
function useFetch<T>(url: string): { data: Ref<T | null>; loading: Ref<boolean> } { ... }
```

---

## 4. Vue 组件规范

### 4.1 组件结构顺序

```vue
<template>
  <!-- 模板内容 -->
</template>

<script setup lang="ts">
// 1. 导入
// 2. Props & Emits
// 3. 响应式数据
// 4. 常量
// 5. 生命周期
// 6. 方法
// 7. 暴露方法
</script>

<style lang="scss" scoped>
/* 样式 */
</style>
```

### 4.2 组件设计原则

```typescript
// 【强制】单一职责：一个组件只做一件事
// 【强制】Props 向下，Events 向上
// 【强制】使用 v-model 实现双向绑定
// 【强制】使用 provide/inject 跨层级通信
```

### 4.3 组合式函数

```typescript
// composables/usePagination.ts
export function usePagination(defaultPageSize = 10) {
  const pagination = reactive({
    pageNum: 1,
    pageSize: defaultPageSize,
    total: 0,
  });

  function setTotal(total: number): void {
    pagination.total = total;
  }

  function reset(): void {
    pagination.pageNum = 1;
  }

  return { pagination, setTotal, reset };
}
```

---

## 5. 状态管理规范

### 5.1 Store 定义

```typescript
// stores/modules/user.ts
export const useUserStore = defineStore('user', () => {
  // State
  const token = ref<string>('');
  const userInfo = ref<UserVO | null>(null);

  // Getters
  const isLoggedIn = computed(() => !!token.value);

  // Actions
  async function loginAction(data: LoginDTO): Promise<void> { ... }
  async function logoutAction(): Promise<void> { ... }

  return { token, userInfo, isLoggedIn, loginAction, logoutAction };
});
```

### 5.2 Store 使用

```typescript
// 【强制】解构时使用 storeToRefs 保持响应性
const { userName, avatar } = storeToRefs(userStore);
const { loginAction, logoutAction } = userStore;

// 【强制】异步操作放在 action 中
```

---

## 6. API 请求规范

### 6.1 Axios 封装要点

- 统一配置 baseURL、timeout
- 请求拦截器添加 Token
- 响应拦截器处理错误码
- Token 过期自动跳转登录

### 6.2 API 模块定义

```typescript
// api/modules/reservation.ts
const BASE_URL = '/api/v1/reservations';

export function pageReservations(query: ReservationQuery) {
  return get<PageResult<ReservationVO>>(BASE_URL, query);
}

export function createReservation(data: ReservationDTO) {
  return post<number>(BASE_URL, data);
}

export function cancelReservation(id: number) {
  return put<void>(`${BASE_URL}/${id}/cancel`);
}
```

---

## 7. 样式规范

### 7.1 SCSS 变量

```scss
// 主题色
$primary-color: #409eff;
$success-color: #67c23a;
$warning-color: #e6a23c;
$danger-color: #f56c6c;

// 间距
$spacing-xs: 4px;
$spacing-sm: 8px;
$spacing-md: 16px;
$spacing-lg: 24px;
```

### 7.2 BEM 命名

```scss
.reservation-card {
  &__header { ... }
  &__content { ... }
  &--active { ... }
  &--disabled { ... }
}
```

### 7.3 样式规范

- 【强制】使用 scoped 避免样式污染
- 【强制】深度选择器使用 `:deep()`
- 【强制】避免使用 `!important`
- 【强制】避免过深嵌套（最多 3 层）

---

## 8. 性能优化规范

### 8.1 组件优化

```typescript
// 【强制】大列表使用虚拟滚动
// 【强制】使用 v-memo 缓存静态内容
// 【强制】使用 shallowRef 优化大对象
// 【强制】使用 computed 缓存计算结果
```

### 8.2 路由优化

```typescript
// 【强制】路由懒加载
component: () => import('@/views/reservation/index.vue')

// 【强制】使用 keep-alive 缓存页面
```

### 8.3 请求优化

```typescript
// 【强制】请求防抖
const debouncedSearch = useDebounceFn(search, 300);

// 【强制】请求取消（页面切换时）
// 【强制】请求缓存（短时间内重复请求）
```

---

## 9. UniApp 小程序规范

### 9.1 分包配置

```json
{
  "subPackages": [
    {
      "root": "pages-sub/reservation",
      "pages": [{ "path": "detail/index" }]
    }
  ],
  "preloadRule": {
    "pages/index/index": {
      "network": "all",
      "packages": ["pages-sub/reservation"]
    }
  }
}
```

### 9.2 性能优化

- 【强制】使用分包加载
- 【强制】图片使用 webp 格式 + lazy-load
- 【强制】长列表使用 scroll-view + 分页
- 【强制】避免频繁 setData，批量更新
- 【强制】使用骨架屏提升体验

### 9.3 组件命名

小程序组件使用 `ls-` 前缀（LingShu 缩写）：
- `ls-seat-map` 座位图组件
- `ls-time-picker` 时间选择器
- `ls-empty` 空状态组件

---

## 10. 代码质量规范

### 10.1 ESLint 核心规则

```javascript
{
  'vue/multi-word-component-names': 'error',
  'vue/no-mutating-props': 'error',
  '@typescript-eslint/no-explicit-any': 'warn',
  '@typescript-eslint/explicit-function-return-type': 'warn',
  'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
  'prefer-const': 'error',
  'eqeqeq': ['error', 'always'],
}
```

### 10.2 Prettier 配置

```javascript
{
  printWidth: 100,
  tabWidth: 2,
  semi: true,
  singleQuote: true,
  trailingComma: 'es5',
}
```

### 10.3 Git 提交规范

```
feat: 新功能
fix: 修复 Bug
docs: 文档更新
style: 代码格式
refactor: 重构
perf: 性能优化
test: 测试
chore: 构建/工具
```

