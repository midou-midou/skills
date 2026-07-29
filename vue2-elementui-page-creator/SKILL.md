---
name: vue2-elementui-page-creator
description: Generates complete Vue 2 table pages using Element UI, following the prison-web project conventions. Produces a full CRUD table page including: search form, el-table with pagination, add/edit dialog sub-components, and API file. The page is driven by four user-provided API definitions (pagination query, add, update, delete). Table columns are derived from query API response data; search filters from query API request params; add/edit dialogs from add/update API params; delete supports batch mode based on delete API params. Trigger when they say things like "generate a table page", "I need a list page", "create a user management page", "make a CRUD page", "add a table with search", or any request involving table/list/CRUD page generation with Element UI.
allowed-tools: 
disable: false
---

# Element UI Table Page Generator

Generates complete, production-ready Vue 2 table pages using Element UI, following the prison-web project conventions. A table page is a composite of: inline search form + el-table + el-pagination + dialog sub-components (add/edit) + API integration.

## What This Skill Does

When a user asks for a table page, this skill:
1. Requires the user to provide four API definitions (pagination query / add / update / delete)
2. Derives table columns from the query API's response data structure
3. Generates search/filter area from the query API's request parameters
4. Generates add/update dialog forms from add/update API request parameters
5. Generates the main `index.vue` (search + table + pagination + dialog wrappers)
6. Generates dialog sub-components (`addXxx.vue`, `editXxx.vue`) under `components/`
7. Generates the API file in `src/api/`
8. All code follows Vue 2 Options API and the project's existing patterns

## Step 1: Gather API Definitions (Required)

**核心原则：整个页面由用户提供的四个接口驱动，不再凭空猜测字段。**

要求用户提供以下四个接口的详细信息（必须全部提供才能开始生成）：

### 1.1 分页查询接口

```js
// 示例
GET / POST  <URL路径>
// 例如: POST /basic/api/prisoner/list

// 请求参数 (query params 或 body)
{
  "pageNum": 1,       // 当前页码 (必传)
  "pageSize": 10,     // 每页条数 (必传)
  "name": "",         // 搜索字段 — 姓名模糊搜索
  "status": null,     // 搜索字段 — 状态下拉筛选
  "startTime": "",    // 搜索字段 — 时间范围开始
  "endTime": ""       // 搜索字段 — 时间范围结束
}

// 响应数据
{
  "code": 200,
  "data": {
    "list": [
      {
        "id": 1,                    // number — 主键
        "name": "张三",              // string — 姓名
        "status": 1,                // number — 状态(0禁用 1启用 2审核中)
        "createTime": "2025-01-01", // string — 创建时间
        "updateTime": "2025-06-01", // string — 更新时间
        "remark": "备注内容"         // string — 备注
      }
    ],
    "totalRows": 100,
    "pageNum": 1,
    "pageSize": 10
  }
}
```

**提取规则**：
- 请求参数中除 `pageNum`/`pageSize` 之外的字段 → **搜索筛选区**的控件
- 响应 `list[0]` 中的字段 → **表格列**
- 响应中的 `totalRows` → 分页 total

### 1.2 新增接口

```js
// 示例
POST <URL路径>
// 例如: POST /basic/api/prisoner

// 请求参数 (body)
{
  "name": "张三",      // string — 姓名 (必填)
  "status": 1,         // number — 状态 (选填,默认1)
  "remark": ""         // string — 备注 (选填)
}

// 响应
{ "code": 200, "message": "success" }
```

**提取规则**：
- 请求参数 → **新增弹窗**的表单项（含必填校验）

### 1.3 更新接口

```js
// 示例
PUT / POST <URL路径>
// 例如: PUT /basic/api/prisoner

// 请求参数 (body)
{
  "id": 1,             // number — 主键 (必填)
  "name": "张三",      // string — 姓名 (必填)
  "status": 1,         // number — 状态 (选填)
  "remark": ""         // string — 备注 (选填)
}

// 响应
{ "code": 200, "message": "success" }
```

**提取规则**：
- 请求参数 → **编辑弹窗**的表单项（id 字段不展示在表单中）

### 1.4 删除接口

```js
// 示例 A：单条删除
DELETE /basic/api/prisoner/1

// 示例 B：批量删除
DELETE /basic/api/prisoner/batch
// body: { "ids": [1, 2, 3] }

// 响应
{ "code": 200, "message": "success" }
```

**提取规则**：
- URL 中有动态 ID → 单条删除
- 参数中有 `ids` 数组 → 支持批量删除，需要在表格前新增「批量删除」按钮 + 多选列
- 如果接口同时支持单条和批量（URL 方式单删 + ids 方式批删），优先采用批量删除逻辑以增强用户体验

### 1.5 额外确认项

如果用户提供的接口信息不完整，需补充确认以下内容：

- **页面名称/实体名**：如未提供，从 URL 路径或文件名推断（`/prisoner` → entity = "prisoner"）
- **权限标识**：如项目使用 `v-permission`，确认对应的权限 key
- **枚举映射**：如果查询接口返回的字段值是数字编码（如 status: 0/1/2），需用户提供中文映射（0=禁用,1=启用,2=审核中）
- **字段筛选方式**：对于查询接口的搜索字段，询问用户使用哪种筛选控件：
  - 文本字段 → `el-input`（精确/模糊由后端决定）
  - 枚举字段 → `el-select`，需用户提供下拉选项列表
  - 时间字段 → `el-date-picker`，需确认是单日期还是日期范围
- **字段展示控制**：是否需要某些响应字段在表格中隐藏（如 id 通常只在操作时使用，不在表格列中展示）

## Step 2: Generate the Main Page (index.vue)

整个 `index.vue` 的结构由 Step 1 中用户提供的四个接口驱动。

### 2.1 搜索筛选区生成规则

从**分页查询接口的请求参数**中提取搜索字段（排除 `pageNum`、`pageSize`），按以下规则生成筛选控件：

| 参数类型 | 控件 | 示例 |
|---------|------|------|
| 文本字段 (name/remark 等) | `el-input` + clearable | `<el-input v-model="pageVO.name" clearable placeholder="请输入名称" />` |
| 枚举字段 (status/type 等) | `el-select` + filterable + clearable | 需用户提供下拉选项 |
| 日期字段 (startTime/endTime) | `el-date-picker` | 需确认单日期还是日期范围(如果有 startTime + endTime 两个字段，优先生成日期范围选择器 startEndTime) |
| 日期范围 (startEndTime) | `el-date-picker type="daterange"` | `<el-date-picker v-model="pageVO.startEndTime" type="daterange" value-format="yyyy-MM-dd" range-separator="至" start-placeholder="开始日期" end-placeholder="结束日期" />` |

**筛选区布局规则**：
- 左侧：新增按钮（如有新增接口）+ 批量删除按钮（如删除接口支持批量）
- 右侧：el-form :inline="true" + 搜索字段 + 查询按钮 + 重置按钮

**需要用户确认**：
- 每个枚举字段的下拉选项列表（如 `[{label:'启用',value:1},{label:'禁用',value:0},{label:'审核中',value:2}]`）
- 如果查询参数中有匹配 `start`/`end` + 相同名称的字段，询问是否合并为日期范围选择器
- 是否需要"重置"按钮清空所有搜索条件

### 2.2 表格列生成规则

从**分页查询接口的响应数据 `list[0]`**中提取所有字段，按以下规则生成表格列：

**列生成优先级**：
1. **id 字段** — 默认不展示为表格列（仅在操作中使用），用户可指定展示
2. **createTime / createdAt / gmtCreate** — 自动展示为「创建时间」列，**默认按降序排序**（`sortable="custom"` + `:default-sort="{prop:'createTime',order:'descending'}"`）
3. **updateTime / updatedAt / gmtModified** — 自动展示为「更新时间」列，默认不设排序
4. **其他字段** — 自动展示

**列类型自动推断**：

| 字段名特征 | 列类型 | 模板 |
|-----------|--------|------|
| 包含 `Time`/`Date` | 时间列 | `<el-table-column prop="xxx" label="xxx" width="160" sortable="custom" />` |
| 包含 `status`/`state`/`type` | 标签列 | `<el-table-column prop="xxx" label="xxx"><template slot-scope="scope"><el-tag :type="statusTypeMap[scope.row.xxx]">{{ statusLabelMap[scope.row.xxx] }}</el-tag></template></el-table-column>` |
| 包含 `avatar`/`image`/`img`/`pic` | 图片列 | `<el-table-column prop="xxx" label="xxx" width="80"><template slot-scope="scope"><el-image :src="scope.row.xxx" style="width:40px;height:40px" fit="cover" /></template></el-table-column>` |
| 包含 `remark`/`desc`/`note`/`content` | 溢出提示列 | `<el-table-column prop="xxx" label="xxx" show-overflow-tooltip />` |
| 其他 | 普通文本列 | `<el-table-column prop="xxx" label="xxx" />` |

**排序行为**：
- 如果响应数据中存在 `createTime` 字段 → 在 `el-table` 上添加 `@sort-change="handleSortChange"`，并在 `getList` 方法中传递排序参数
- 创建时间默认降序（最新的在前面），更新时间默认不排序
- 在 `pageVO` 中增加 `sortField` 和 `sortOrder` 字段用于传参

### 2.3 批量删除功能

根据删除接口判断是否生成批量删除逻辑：

**判断规则**：
- 删除接口参数中有 `ids` 数组 → **启用批量删除**，需要：
  - 在 `el-table` 中添加 `type="selection"` 的多选列
  - 在新增按钮旁添加「批量删除」按钮，当 `selection.length === 0` 时 disabled
  - 在 `methods` 中增加 `handleBatchDelete` 方法

```vue
<!-- 多选列 -->
<el-table-column type="selection" width="55" />

<!-- 批量删除按钮 -->
<el-button 
  v-permission="['<perm>:delete']" 
  type="danger" 
  :disabled="selection.length === 0" 
  @click="handleBatchDelete"
>批量删除</el-button>
```

- 删除接口仅接受单个 ID → **仅支持单条删除**，每行操作列中显示删除按钮

### 2.4 完整模板示例

以下是根据示例接口生成的完整 `index.vue`（假设支持批量删除，有 createTime 默认降序）：

```vue
<template>
  <div id="<entity>-container">
    <div id="app-container">
      <el-card v-loading="loading">
        <!-- Search bar -->
        <div class="header-search">
          <div>
            <el-button v-permission="['<perm>:add']" type="primary" @click="openAddDialog">添加</el-button>
            <el-button v-permission="['<perm>:delete']" type="danger" :disabled="selection.length === 0" @click="handleBatchDelete">批量删除</el-button>
          </div>
          <el-form :inline="true" :model="pageVO">
            <el-form-item label="姓名">
              <el-input v-model="pageVO.name" clearable placeholder="请输入姓名" />
            </el-form-item>
            <el-form-item label="状态">
              <el-select v-model="pageVO.status" filterable clearable placeholder="请选择">
                <el-option v-for="item in statusOptions" :key="item.value" :label="item.label" :value="item.value" />
              </el-select>
            </el-form-item>
            <el-form-item label="创建时间">
              <el-date-picker v-model="pageVO.startEndTime" type="daterange" value-format="yyyy-MM-dd" range-separator="至" start-placeholder="开始日期" end-placeholder="结束日期" />
            </el-form-item>
            <el-form-item>
              <el-button type="primary" @click="onSubmit">查询</el-button>
              <el-button @click="onReset">重置</el-button>
            </el-form-item>
          </el-form>
        </div>
        <!-- Table -->
        <el-table :data="list" border @sort-change="handleSortChange" :default-sort="{prop:'createTime',order:'descending'}">
          <el-table-column type="selection" width="55" />
          <el-table-column type="index" :index="indexMethod" label="序号" width="60" />
          <el-table-column prop="name" label="姓名" />
          <el-table-column prop="status" label="状态">
            <template slot-scope="scope">
              <el-tag :type="statusTypeMap[scope.row.status]">{{ statusLabelMap[scope.row.status] }}</el-tag>
            </template>
          </el-table-column>
          <el-table-column prop="createTime" label="创建时间" width="160" sortable="custom" />
          <el-table-column prop="updateTime" label="更新时间" width="160" sortable="custom" />
          <el-table-column prop="remark" label="备注" show-overflow-tooltip />
          <el-table-column fixed="right" label="操作" width="180">
            <template slot-scope="scope">
              <el-button v-permission="['<perm>:update']" type="text" @click="openEditDialog(scope.row)">编辑</el-button>
              <el-button v-permission="['<perm>:delete']" type="text" @click="handleDelete(scope.row.id)">删除</el-button>
            </template>
          </el-table-column>
        </el-table>
        <!-- Pagination -->
        <el-pagination
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
          :current-page="pageVO.pageNum"
          :page-sizes="[10, 20, 30, 40, 50]"
          :page-size="pageVO.pageSize"
          layout="total, sizes, prev, pager, next, jumper"
          :total="totalRows"
        />
      </el-card>
      <!-- Dialog sub-components -->
      <add-<entity> :add-dialog-visible="addDialogVisible" @add-success="getList()" @close-dialog="addDialogVisible = false" />
      <edit-<entity> :edit-dialog-visible="editDialogVisible" :node="currentRow" @edit-success="getList()" @close-dialog="editDialogVisible = false" />
    </div>
  </div>
</template>

<script>
import { getXxxApi, deleteXxxApi, batchDeleteXxxApi } from '@/api/<entity>'
import Add<Entity> from './components/add<Entity>'
import Edit<Entity> from './components/edit<Entity>'

export default {
  name: '<Entity>Page',
  components: { Add<Entity>, Edit<Entity> },
  data() {
    return {
      pageVO: {
        pageNum: 1,
        pageSize: 10,
        name: '',
        status: null,
        startEndTime: [],
        sortField: 'createTime',
        sortOrder: 'descending'
      },
      list: [],
      totalRows: 0,
      loading: false,
      selection: [],
      addDialogVisible: false,
      editDialogVisible: false,
      currentRow: {},
      // 枚举映射 — 由用户提供
      statusOptions: [
        { label: '启用', value: 1 },
        { label: '禁用', value: 0 },
        { label: '审核中', value: 2 }
      ],
      statusTypeMap: { 0: 'info', 1: 'success', 2: 'warning' },
      statusLabelMap: { 0: '禁用', 1: '启用', 2: '审核中' }
    }
  },
  created() {
    this.getList()
  },
  methods: {
    async getList() {
      this.loading = true
      const params = { ...this.pageVO }
      // 日期范围处理：拆分 startEndTime 为 startTime / endTime
      if (params.startEndTime && params.startEndTime.length === 2) {
        params.startTime = params.startEndTime[0]
        params.endTime = params.startEndTime[1]
      }
      delete params.startEndTime
      const res = await getXxxApi(params)
      this.list = res.data.list
      this.totalRows = res.data.totalRows
      this.pageVO.pageNum = res.data.pageNum
      this.pageVO.pageSize = res.data.pageSize
      this.loading = false
    },
    onSubmit() {
      this.pageVO.pageNum = 1
      this.getList()
    },
    onReset() {
      this.pageVO.name = ''
      this.pageVO.status = null
      this.pageVO.startEndTime = []
      this.pageVO.pageNum = 1
      this.getList()
    },
    handleSortChange({ prop, order }) {
      this.pageVO.sortField = prop
      this.pageVO.sortOrder = order || ''
      this.pageVO.pageNum = 1
      this.getList()
    },
    handleSizeChange(val) {
      this.pageVO.pageSize = val
      this.getList()
    },
    handleCurrentChange(val) {
      this.pageVO.pageNum = val
      this.getList()
    },
    handleSelectionChange(val) {
      this.selection = val
    },
    openAddDialog() {
      this.addDialogVisible = true
    },
    openEditDialog(row) {
      this.currentRow = row
      this.editDialogVisible = true
    },
    handleDelete(id) {
      this.$confirm('确定要删除该条记录吗？', '温馨提示', { type: 'warning' })
        .then(async () => {
          await deleteXxxApi(id)
          this.$message.success('删除成功！')
          await this.getList()
        }).catch(() => {})
    },
    handleBatchDelete() {
      if (this.selection.length === 0) return
      const ids = this.selection.map(item => item.id)
      this.$confirm(`确定要删除选中的 ${ids.length} 条记录吗？`, '温馨提示', { type: 'warning' })
        .then(async () => {
          await batchDeleteXxxApi({ ids })
          this.$message.success('删除成功！')
          this.selection = []
          await this.getList()
        }).catch(() => {})
    },
    indexMethod(index) {
      return (this.pageVO.pageNum - 1) * this.pageVO.pageSize + index + 1
    }
  }
}
</script>

<style scoped>
.header-search {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 15px;
}
</style>
```

## Step 3: Generate Dialog Sub-Components

每个弹窗的表单项完全由对应接口的**请求参数**驱动。弹窗放在 `src/views/<module>/<entity>/components/`。

### 3.1 表单项生成规则

根据接口请求参数的类型，自动选择对应的表单控件：

| 参数类型/特征 | 表单控件 | 示例 |
|------------|---------|------|
| 文本 (name/title 等) | `el-input` | `<el-input v-model="formData.name" placeholder="请输入名称" />` |
| 枚举 (status/type/sex 等) | `el-select` + 下拉选项 | `<el-select v-model="formData.status" placeholder="请选择"><el-option v-for="item in statusOptions" :key="item.value" :label="item.label" :value="item.value" /></el-select>` |
| 日期 (xxxTime/xxxDate) | `el-date-picker` | `<el-date-picker v-model="formData.createTime" type="date" value-format="yyyy-MM-dd" placeholder="请选择日期" />` |
| 布尔 (xxxFlag/isXxx) | `el-switch` | `<el-switch v-model="formData.isEnabled" active-value="1" inactive-value="0" />` |
| 数字 (age/sort/priority) | `el-input-number` | `<el-input-number v-model="formData.age" :min="0" :max="150" />` |
| 长文本 (remark/desc/content) | `el-input type="textarea"` | `<el-input v-model="formData.remark" type="textarea" :rows="3" placeholder="请输入备注" />` |

**必填 vs 选填**：

- 用户标明"必填"的参数 → 添加 `rules` 校验规则
- 用户标明"选填"或无标注的参数 → 不添加 required 规则
- 必填参数在 label 后添加红色星号提示

**枚举选项处理**：
- 新增弹窗和编辑弹窗共用相同的枚举选项（如 statusOptions）
- 如果弹窗需要从后端接口动态获取下拉选项，需用户提供对应的字典接口

### 3.2 新增弹窗 (addXxx.vue)

表单项完全来自**新增接口的请求参数**：

```vue
<template>
  <el-dialog title="添加" :visible.sync="addDialogVisible" width="600px" :before-close="handleClose">
    <el-form ref="formData" :model="formData" :rules="rules" label-width="100px">
      <!-- 以下字段全部来自新增接口的请求参数 -->
      <el-form-item label="姓名" prop="name">
        <el-input v-model="formData.name" placeholder="请输入姓名" />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select v-model="formData.status" placeholder="请选择状态">
          <el-option v-for="item in statusOptions" :key="item.value" :label="item.label" :value="item.value" />
        </el-select>
      </el-form-item>
      <el-form-item label="备注" prop="remark">
        <el-input v-model="formData.remark" type="textarea" :rows="3" placeholder="请输入备注" />
      </el-form-item>
    </el-form>
    <div slot="footer">
      <el-button @click="handleClose">取 消</el-button>
      <el-button type="primary" @click="handleSubmit">确 定</el-button>
    </div>
  </el-dialog>
</template>

<script>
import { addXxxApi } from '@/api/<entity>'

export default {
  name: 'Add<Entity>',
  props: {
    addDialogVisible: { type: Boolean, default: false }
  },
  data() {
    return {
      // 初始值根据新增接口参数类型推断：string → ''，number → null/null，array → []
      formData: {
        name: '',
        status: null,
        remark: ''
      },
      rules: {
        // 仅必填字段生成校验规则
        name: [{ required: true, message: '请输入姓名', trigger: 'blur' }]
        // status 为选填，不生成 rules
      },
      // 枚举选项 — 新增和编辑共用，可提取为常量或由父组件传入
      statusOptions: [
        { label: '启用', value: 1 },
        { label: '禁用', value: 0 }
      ]
    }
  },
  methods: {
    async handleSubmit() {
      this.$refs.formData.validate(async (valid) => {
        if (!valid) return
        await addXxxApi(this.formData)
        this.$message.success('添加成功！')
        this.$emit('add-success')
        this.handleClose()
      })
    },
    handleClose() {
      this.$refs.formData.resetFields()
      this.$emit('close-dialog')
    }
  }
}
</script>
```

### 3.3 编辑弹窗 (editXxx.vue)

表单项来自**更新接口的请求参数**，排除 `id` 字段（id 不在表单中展示）：

```vue
<template>
  <el-dialog title="编辑" :visible.sync="editDialogVisible" width="600px" :before-close="handleClose">
    <el-form ref="formData" :model="formData" :rules="rules" label-width="100px">
      <!-- 编辑弹窗表单字段 = 更新接口请求参数 - id，其他与新增弹窗保持一致 -->
      <el-form-item label="姓名" prop="name">
        <el-input v-model="formData.name" placeholder="请输入姓名" />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select v-model="formData.status" placeholder="请选择状态">
          <el-option v-for="item in statusOptions" :key="item.value" :label="item.label" :value="item.value" />
        </el-select>
      </el-form-item>
      <el-form-item label="备注" prop="remark">
        <el-input v-model="formData.remark" type="textarea" :rows="3" placeholder="请输入备注" />
      </el-form-item>
    </el-form>
    <div slot="footer">
      <el-button @click="handleClose">取 消</el-button>
      <el-button type="primary" @click="handleSubmit">确 定</el-button>
    </div>
  </el-dialog>
</template>

<script>
import { updateXxxApi } from '@/api/<entity>'

export default {
  name: 'Edit<Entity>',
  props: {
    editDialogVisible: { type: Boolean, default: false },
    node: { type: Object, default: () => ({}) }
  },
  data() {
    return {
      formData: {
        id: null,    // 更新时必须传 id，但不在表单中展示
        name: '',
        status: null,
        remark: ''
      },
      rules: {
        name: [{ required: true, message: '请输入姓名', trigger: 'blur' }]
      },
      statusOptions: [
        { label: '启用', value: 1 },
        { label: '禁用', value: 0 }
      ]
    }
  },
  watch: {
    editDialogVisible(val) {
      if (val && this.node) {
        // 从父组件传入的 row 数据回填表单
        this.formData = {
          id: this.node.id,
          name: this.node.name || '',
          status: this.node.status != null ? this.node.status : null,
          remark: this.node.remark || ''
        }
      }
    }
  },
  methods: {
    async handleSubmit() {
      this.$refs.formData.validate(async (valid) => {
        if (!valid) return
        await updateXxxApi(this.formData)
        this.$message.success('编辑成功！')
        this.$emit('edit-success')
        this.handleClose()
      })
    },
    handleClose() {
      this.$refs.formData.resetFields()
      this.$emit('close-dialog')
    }
  }
}
</script>
```

**关键差异（新增 vs 编辑）**：

| 维度 | 新增弹窗 | 编辑弹窗 |
|------|---------|---------|
| 数据来源 | 新增接口请求参数 | 更新接口请求参数 - id 字段 |
| Props | `addDialogVisible` | `editDialogVisible` + `node` |
| 数据回填 | 无需回填（初始值全空） | 弹窗打开时 `watch visible` → 用 `node` 数据回填 |
| API 调用 | `addXxxApi` | `updateXxxApi` |
| 成功事件 | `add-success` | `edit-success` |
| 枚举选项 | 如果新增和编辑的字段值域相同，枚举选项可以共用；如果不同需要各自维护 | |

## Step 4: Generate the API File

⚠️ API 文件中的函数**完全由用户提供的四个接口定义驱动**，不做任何凭空假设。

Create `src/api/<entity>.js` following the project convention. 根据删除接口类型决定是否生成批量删除 API 函数：

### 单体删除版本（删除接口仅接受单个 ID）

```js
import request from '@/utils/request'

// 分页查询 — 来自用户提供的分页查询接口
export function getXxxApi(data) {
  return request({ url: '/basic/api/<entity>/list', method: 'POST', data })
}

// 新增 — 来自用户提供的新增接口
export function addXxxApi(data) {
  return request({ url: '/basic/api/<entity>', method: 'POST', data })
}

// 更新 — 来自用户提供的更新接口
export function updateXxxApi(data) {
  return request({ url: '/basic/api/<entity>', method: 'PUT', data })
}

// 删除（单体）— URL 中携带 ID
export function deleteXxxApi(id) {
  return request({ url: '/basic/api/<entity>/' + id, method: 'DELETE' })
}
```

### 批量删除版本（删除接口接受 ids 数组）

```js
import request from '@/utils/request'

export function getXxxApi(data) {
  return request({ url: '/basic/api/<entity>/list', method: 'POST', data })
}

export function addXxxApi(data) {
  return request({ url: '/basic/api/<entity>', method: 'POST', data })
}

export function updateXxxApi(data) {
  return request({ url: '/basic/api/<entity>', method: 'PUT', data })
}

// 删除（单体）
export function deleteXxxApi(id) {
  return request({ url: '/basic/api/<entity>/' + id, method: 'DELETE' })
}

// 批量删除
export function batchDeleteXxxApi(data) {
  return request({ url: '/basic/api/<entity>/batch', method: 'DELETE', data })
}
```

### URL 和 Method 确定规则

- **严格使用用户提供的 URL 和 Method** — 不自行推断或修改
- 如果用户没有明确指定 method：GET 用于查询，POST 用于新增，PUT 用于更新，DELETE 用于删除
- 所有 API 函数名以 `Api` 后缀结尾
- 导入路径使用项目统一的 `@/utils/request`

---

## Step 5: 生成前的确认清单

在生成代码前，逐项确认以下内容。缺失项请用户补充，不要凭空推测：

| # | 确认项 | 来源 | 说明 |
|---|--------|------|------|
| 1 | 页面实体名 | 用户/URL推断 | 如 prisoner、device |
| 2 | 分页查询接口 URL + Method | 用户提供 | 必须有响应数据结构 |
| 3 | 分页查询接口的请求参数 | 用户提供 | pageNum/pageSize 之外的是筛选字段 |
| 4 | 分页查询接口的响应字段 | 用户提供 | list[0] 的字段决定表格列 |
| 5 | 新增接口 URL + Method + 请求参数 | 用户提供 | 参数决定新增弹窗表单项 |
| 6 | 更新接口 URL + Method + 请求参数 | 用户提供 | 参数 - id 决定编辑弹窗表单项 |
| 7 | 删除接口 URL + Method + 参数 | 用户提供 | 决定单条/批量删除模式 |
| 8 | 枚举字段的中文映射 | 用户提供 | 状态、类型等 0/1/2 的 label |
| 9 | 每个筛选字段的控件类型 | 用户/规则推断 | input/select/date-picker |
| 10 | 是否有 createTime/updateTime | 响应字段 | 决定默认排序 |
| 11 | 权限标识 | 用户提供 | v-permission 的 key |

## Important Conventions

- **Vue 2 Options API only** — this project runs Vue 2.6.10. Do not use `<script setup>`, Composition API, or Vue 3 syntax.
- **接口驱动生成** — 所有字段、控件、校验完全由用户提供的四个接口定义推导，不凭空生成。
- **Pagination params in pageVO** — combine search fields and pagination (pageNum, pageSize) + sort (sortField, sortOrder) into one object.
- **API response shape** — 默认后端返回 `{ data: { list: [], totalRows: 0, pageNum: 1, pageSize: 10 } }`。如果用户提供的响应结构不同，按实际结构适配。
- **Delete confirmation** — always use `this.$confirm('确定要删除吗？', '温馨提示', { type: 'warning' })` before deleting.
- **Dialog close** — always call `this.$refs.formData.resetFields()` before closing a dialog, and emit the close event.
- **Permission directive** — use `v-permission="['key']"` on CRUD buttons if the project uses permission control.
- **Index method** — use the continuous index across pages: `(pageNum - 1) * pageSize + index + 1`.
- **el-table border** — always add `border` attribute to el-table.
- **排序默认值** — 如查询接口响应中有 `createTime`/`createAt`，默认按创建时间降序；有 `updateTime`/`updateAt` 不设默认排序。

## File Output Checklist

When generating a complete table page, produce these files:

1. `src/views/<module>/<entity>/index.vue` — 主页面（搜索筛选区+表格+分页+弹窗容器）
2. `src/views/<module>/<entity>/components/add<Entity>.vue` — 新增弹窗（表单由新增接口参数驱动）
3. `src/views/<module>/<entity>/components/edit<Entity>.vue` — 编辑弹窗（表单由更新接口参数驱动）
4. `src/api/<entity>.js` — API 文件（由四个接口定义驱动，按需包含批量删除）

Tell the user what files were created and where, so they can verify and integrate.
