---
name: API-data-to-view-data
description: 前端接口数据到视图数据映射验证工具，适用于 Vue 2 和 Vue 3 项目。扫描前端项目中的 API 定义、代理配置和认证方式，用 curl 调用接口获取后端真实数据，然后将数据与 Vue 文件中的视图展示方式做对应映射，最后校验边界条件。当用户说"验证接口数据"、"接口和视图对应"、"检查接口返回数据展示"、"数据映射"、"API 数据到视图"、"接口数据校验"、"前后端数据一致性"、"curl 调接口看数据"、"检查接口数据在页面怎么展示"等场景时使用。即使用户没有明确说"数据映射"或"校验"，只要涉及接口返回数据与页面展示的对应关系，都应该使用此 skill。
allowed-tools:
disable: false
---

# API 数据到视图数据映射验证

本 skill 的核心目标：**从前端项目的 API 定义出发，组装完整的请求 URL，用 curl 调用接口拿到后端真实数据，然后验证这些数据在 Vue 文件中是否正确展示。**同时支持 Vue 2 和 Vue 3 项目。

### Vue 版本识别

在开始流程前，先检查项目的 Vue 版本：

1. 读取 `package.json`，查看 `vue` 依赖的版本号
2. Vue 2.x → 使用 Options API 语法分析，注意不支持可选链 `?.`
3. Vue 3.x → 支持 Composition API / `<script setup>`，支持可选链 `?.`
4. 版本号影响后续的代码分析策略（组件写法、空值处理方式等）

整个流程分为三个步骤：

## Step 1: 组装请求并调用接口

### 1.1 扫描 API 定义文件

扫描前端项目中的 API 定义文件夹（默认为 `src/api/`，用户可自定义路径）。重点读取每个 `.js` 或 `.ts` 文件，提取：

- **接口 URL**：如 `/basic/api/prisoner/list`
- **请求方法**：GET / POST / PUT / DELETE
- **请求参数**：query 参数或 body 参数的字段名和类型
- **函数名**：如 `getXxxApi`、`addXxxApi`

```js
// 典型 API 文件示例
import request from '@/utils/request'

export function getListApi(data) {
  return request({ url: '/basic/api/prisoner/list', method: 'POST', data })
}
```

从上述文件提取出：URL 为 `/basic/api/prisoner/list`，方法为 POST，参数为 data（包含 pageNum、pageSize 等分页和搜索字段）。

### 1.2 扫描代理配置，组装完整 URL

前端项目在开发环境通常使用代理转发 API 请求。需要扫描以下配置文件，找到 proxy 配置：

- `vite.config.js` / `vite.config.ts` — Vite 项目
- `vue.config.js` — Vue CLI 项目
- `webpack.config.js` / `webpack.dev.conf.js` — Webpack 项目

代理配置典型形式：

```js
// vite.config.js
server: {
  proxy: {
    '/basic': {
      target: 'http://192.168.1.100:8080',
      changeOrigin: true
    }
  }
}

// vue.config.js
devServer: {
  proxy: {
    '/basic': {
      target: 'http://192.168.1.100:8080',
      changeOrigin: true
    }
  }
}
```

**组装完整 URL 的逻辑**：

1. 找到 API URL 的前缀（如 `/basic`）
2. 在 proxy 配置中匹配该前缀对应的 target
3. 将 target + API URL 拼接成完整地址：`http://192.168.1.100:8080/basic/api/prisoner/list`
4. 如果 proxy 配置中有 `pathRewrite`，需要按规则调整 URL 路径

如果找不到代理配置，询问用户后端服务地址，或使用 `localhost` 常见端口（3000、8080、8000 等）尝试。

### 1.3 检测接口认证方式

扫描项目中的请求封装文件（通常是 `src/utils/request.js` 或 `src/utils/request.ts`），以及相关的认证逻辑文件，检测以下三种认证方式：

**方式一：Authorization Bearer 头**

```js
// request.js 中的拦截器
service.interceptors.request.use(config => {
  const token = getToken()
  if (token) {
    config.headers['Authorization'] = `Bearer ${token}`
  }
  return config
})
```

curl 命令对应：
```bash
curl -X POST 'http://192.168.1.100:8080/basic/api/prisoner/list' \
  -H 'Authorization: Bearer <token_value>' \
  -H 'Content-Type: application/json' \
  -d '{"pageNum":1,"pageSize":10}'
```

**方式二：Cookies 带 token 字段**

```js
// 使用 cookie 存储 token
import Cookies from 'js-cookie'
service.interceptors.request.use(config => {
  const token = Cookies.get('token')
  if (token) {
    config.headers['token'] = token  // 或 config.cookies.token
  }
  return config
})
```

curl 命令对应：
```bash
curl -X POST 'http://192.168.1.100:8080/basic/api/prisoner/list' \
  -H 'Cookie: token=<token_value>' \
  -H 'Content-Type: application/json' \
  -d '{"pageNum":1,"pageSize":10}'
```

**方式三：自定义请求头**

```js
// 自定义 header 名
config.headers['X-Access-Token'] = token
config.headers['X-User-Id'] = userId
```

curl 命令对应：
```bash
curl -X POST 'http://192.168.1.100:8080/basic/api/prisoner/list' \
  -H 'X-Access-Token: <token_value>' \
  -H 'Content-Type: application/json' \
  -d '{"pageNum":1,"pageSize":10}'
```

**获取 token 的方法**：

1. 检查项目中 token 的存储位置（localStorage、sessionStorage、Cookie）
2. 检查 token 的获取方式（登录接口返回、SSO 单点登录等）
3. 如果有登录接口，先调用登录接口获取 token，再用于后续请求
4. 如果无法自动获取 token，需要用户提供

### 1.4 执行 curl 请求

组装好完整 URL、认证方式和请求参数后，使用 `execute_command` 工具执行 curl 命令。

**请求参数的确定**：

- 对于 POST 请求，从 API 文件和 Vue 文件中推断必要的请求体字段
- 列表查询接口通常需要 `{ pageNum: 1, pageSize: 10 }` 作为最小参数集
- **Vue 2 Options API**：从 `data()` 返回值和搜索表单推断参数字段
- **Vue 3 `<script setup>`**：从 `ref()` / `reactive()` 定义推断参数字段
- 如果接口需要其他必填参数，从 Vue 文件中推断

**处理请求结果**：

- 请求成功：解析响应 JSON，提取数据结构（字段名、类型、嵌套关系）
- 请求失败（4xx/5xx）：检查认证是否正确、参数是否完整、URL 是否正确
- 网络错误：检查代理地址是否可达

---

## Step 2: 数据与视图映射

### 2.1 找到调用该接口的 Vue 文件

根据 API 函数名（如 `getListApi`），在项目中搜索哪些 `.vue` 文件 import 并调用了该函数。

```bash
# 搜索使用某个 API 函数的 Vue 文件
grep -r "getListApi" src/views/ --include="*.vue"
```

找到对应的 Vue 文件后，分析其视图展示方式。注意 Vue 文件可能使用两种写法：

- **Vue 2 Options API**：`<script>` 中有 `export default { methods: { ... } }`，API 调用在 methods 中
- **Vue 3 `<script setup>`**：`<script setup>` 中直接 import 和调用，无需 methods 包装

### 2.2 分析数据展示方式

读取 Vue 文件的 `<template>` 和 `<script>` 部分，识别数据的展示形式。注意区分 Vue 2 和 Vue 3 的写法差异：

| 展示方式 | 识别特征 | 数据绑定 | Vue 2 写法 | Vue 3 写法 |
|---------|---------|---------|-----------|-----------|
| 表格 | `el-table` / `el-table-column` | `:data="list"`，列 `prop` 对应字段名 | Options API `data()` | `ref()` / `reactive()` |
| 表单详情 | `el-form` + `el-form-item` | `:model="detail"`，item `prop` 对应字段名 | `this.detail` | `detail.value` |
| 卡片列表 | `el-card` 循环渲染 | `v-for="item in list"` | `this.list` | `list.value` |
| 描述列表 | `el-descriptions` / `el-descriptions-item` | `:label` + 内容绑定 | 同上 | 同上 |
| 树形 | `el-tree` | `:data="treeData"`，`props` 配置映射 | 同上 | 同上 |
| 统计图 | `echarts` / 图表组件 | 数据转换为图表配置格式 | 同上 | 同上 |

**Vue 3 `<script setup>` 特殊处理**：

```vue
<!-- Vue 3 script setup 写法 -->
<script setup>
import { ref, onMounted } from 'vue'
import { getListApi } from '@/api/prisoner'

const list = ref([])
const loading = ref(false)

const getList = async () => {
  loading.value = true
  const res = await getListApi({ pageNum: 1, pageSize: 10 })
  list.value = res.data.list
  loading.value = false
}

onMounted(() => { getList() })
</script>
```

分析 Vue 3 `<script setup>` 时，需要注意：
- 变量用 `ref()` / `reactive()` 定义，访问时需 `.value`（模板中自动解包）
- API 调用直接在 `<script setup>` 顶层，不需要 `methods` 包装
- `onMounted` / `onUnmounted` 等生命周期钩子替代 `created` / `mounted`

### 2.3 逐字段映射校验

将 curl 返回的每个数据字段，与 Vue 文件中对应的展示元素做一一映射：

```
## 数据映射报告 — /basic/api/prisoner/list

| 后端字段 | 字段类型 | Vue 展示方式 | 展示组件 | 映射状态 |
|---------|---------|------------|---------|---------|
| id | Number | 未展示 | — | 列表查询通常不展示 id |
| name | String | 表格列 | el-table-column prop="name" label="姓名" | ✓ 正常 |
| status | Number(0/1) | 表格列(标签) | el-tag :type 动态映射 | ✓ 正常 |
| createTime | String | 表格列 | el-table-column prop="createTime" label="创建时间" | ✓ 正常 |
| avatar | String(URL) | 表格列(图片) | el-image :src="row.avatar" | ✓ 正常 |
| remark | String | 表格列(溢出提示) | show-overflow-tooltip | ✓ 正常 |
| orgName | String | 未展示 | — | ⚠ 后端返回但前端未使用 |
| newField | String | — | — | ⚠ 后端新增字段，前端未展示 |
```

**映射校验关注点**：

- **字段缺失**：后端返回了字段，但 Vue 文件没有对应展示 → 新增字段未使用或遗漏
- **展示缺失**：Vue 文件展示了某个字段，但后端数据中没有 → 接口变更或前端硬编码
- **类型不匹配**：后端返回 Number，前端当作 String 处理 → 可能导致排序/比较异常
- **枚举值映射**：后端返回 0/1/2，前端需要映射为"禁用/启用/审核中" → 检查映射字典是否完整
- **空值处理**：后端可能返回 null/undefined/空字符串 → 前端是否有 v-if 或默认值守卫
- **嵌套对象**：后端返回 `{ user: { name: 'xxx' } }` → 前端是否正确访问 `row.user.name`。Vue 2 项目不支持可选链 `?.`，需用 `row.user && row.user.name` 或默认值；Vue 3 项目可直接用 `row.user?.name`

---

## Step 3: 校验结果与边界条件

### 3.1 数据结构校验

用 curl 返回的真实数据校验以下边界条件：

**空数据场景**
```bash
# 请求空结果（如搜索一个不存在的条件）
curl -X POST '...' -d '{"pageNum":1,"pageSize":10,"name":"不存在"}'
# 检查返回：{ data: { list: [], totalRows: 0 } }
# Vue 中 el-table 需要能正确渲染空列表，el-pagination 的 total 为 0
```

**边界数据场景**
- 分页边界：pageNum 超过总页数时的响应
- pageSize 极值：pageSize=1 和 pageSize=1000 时的响应
- 字段为 null：某个字段返回 null 时，表格/表单是否崩溃
- 超长字符串：remark 返回 500 字长文本时，show-overflow-tooltip 是否生效
- 特殊字符：name 包含 `<script>` 等特殊字符时，是否有 XSS 风险

**数据类型校验**
```js
// 检查后端返回的数据类型是否与前端使用一致
// 例如：status 是 Number 还是 String？
// 如果后端返回 "1"（字符串），而前端用 === 1 比较，永远不成立
const status = row.status  // 后端返回 "1" 
if (status === 1) { ... }  // ⚠ 类型不匹配，永远 false
```

### 3.2 字段完整性校验

对比 API 文件中定义的请求参数和 Vue 文件中实际传递的参数：

```js
// API 定义
export function getListApi(data) {
  return request({ url: '/xxx/list', method: 'POST', data })
}

// Vue 2 Options API 调用
async getList() {
  const res = await getListApi(this.pageVO)  // pageVO 包含什么字段？
}

// Vue 3 script setup 调用
const pageVO = reactive({ pageNum: 1, pageSize: 10 })
const getList = async () => {
  const res = await getListApi(pageVO)  // pageVO 包含什么字段？
}
```

检查：
- `pageVO` 是否包含 API 期望的所有必填字段
- 是否有多余字段被传递（后端可能忽略或报错）
- 分页参数 `pageNum` / `pageSize` 是否正确命名（有些接口用 `page` / `size`）
- Vue 3 的 `reactive()` 对象传递时不需要 `.value`，`ref()` 传递时需要 `.value`

### 3.3 输出校验报告

最终输出一份完整的校验报告：

```
# API 数据到视图校验报告

## 1. 接口信息
- URL: http://192.168.1.100:8080/basic/api/prisoner/list
- 方法: POST
- 认证: Authorization Bearer Token
- 代理来源: vite.config.js → /basic → http://192.168.1.100:8080

## 2. 数据结构
后端返回数据示例：
{
  "code": 200,
  "data": {
    "list": [...],
    "totalRows": 50,
    "pageNum": 1,
    "pageSize": 10
  }
}

## 3. 字段映射

### 正常映射 (5/7)
| 后端字段 | 前端展示 | 组件 |
|---------|---------|------|
| name | el-table-column | 姓名 |
| status | el-tag | 状态(0=禁用,1=启用) |
| createTime | el-table-column | 创建时间 |
| avatar | el-image | 头像 |
| remark | el-table-column(show-overflow-tooltip) | 备注 |

### 异常映射 (2/7)
| 后端字段 | 问题 | 建议 |
|---------|------|------|
| orgName | 后端返回但前端未展示 | 如需展示，添加 el-table-column |
| newField | 后端新增字段前端无对应 | 确认是否需要展示，如需则添加映射 |

## 4. 边界条件检测结果

| 测试场景 | 结果 | 详情 |
|---------|------|------|
| 空列表 | ✓ 正常 | el-table 渲染空数据，pagination total=0 |
| null 字段 | ⚠ 风险 | row.orgName 为 null 时，el-table-column 显示空白 |
| 分页超限 | ✓ 正常 | 后端返回空列表 |
| 类型不匹配 | ✓ 正常 | status 类型前后端一致(Number) |

## 5. 建议
1. orgName 字段未展示，如业务需要建议添加表格列
2. null 值场景建议在模板中添加默认值守卫：(row.orgName || '--')
```

---

## 执行流程总结

1. **扫描** → 读取 API 文件、proxy 配置、认证方式
2. **组装** → 拼完整 URL + 认证头 + 请求参数
3. **请求** → curl 调用，拿到真实数据
4. **映射** → 数据字段 ↔ Vue 展示元素一一对应
5. **校验** → 边界条件、类型一致性、空值处理
6. **报告** → 输出完整校验报告

## 注意事项

- curl 调用需要后端服务可访问，如果服务不可达，需要用户提供可用的地址
- token 可能过期，如果请求返回 401，需要重新获取 token
- **Vue 2 项目**不支持可选链 `?.`，空值处理需用 `&&` 或默认值，在分析时需特别注意
- **Vue 3 项目**支持可选链 `?.` 和 Composition API / `<script setup>`，分析代码时需适配
- 如果项目使用了 TypeScript，API 文件可能是 `.ts` 格式，解析逻辑相同
- 对于分页接口，默认请求第一页最小数据量（pageNum=1, pageSize=10）以减少响应体积
- Vue 3 项目可能使用 Element Plus（而非 Element UI），组件名和 API 有差异（如 `ElTable` → `el-table`），需在映射时注意
