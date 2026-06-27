---
name: vue2-elementui-bmp-creator
description: |
  Generates complete Vue 2 table pages using Element UI, following the prison-web project conventions. Produces a full CRUD table page including: search form, el-table with pagination, add/edit dialog sub-components, and API file. Use this skill whenever the user wants to create a table page, list page, CRUD page, or any page that displays data in a table with search and pagination. Trigger when they say things like "generate a table page", "I need a list page", "create a user management page", "make a CRUD page", "add a table with search", or any request involving table/list/CRUD page generation with Element UI — even if they don't explicitly say "table".
---

# Element UI Table Page Generator

Generates complete, production-ready Vue 2 table pages using Element UI, following the prison-web project conventions. A table page is a composite of: inline search form + el-table + el-pagination + dialog sub-components (add/edit) + API integration.

## What This Skill Does

When a user asks for a table page, this skill:
1. Gathers requirements (columns, search fields, CRUD operations, API endpoints)
2. Generates the main `index.vue` (search + table + pagination + dialog wrappers)
3. Generates dialog sub-components (`addXxx.vue`, `editXxx.vue`) under `components/`
4. Generates the API file in `src/api/`
5. All code follows Vue 2 Options API and the project's existing patterns

## Step 1: Gather Requirements

Ask the user for these details (some can be inferred from context):

- **Page name**: What entity? (e.g., "prisoner", "device", "visitor")
- **Table columns**: What columns to display? For each column: label, prop name, column type (text, tag, date, image, etc.), width (if needed)
- **Search fields**: Which fields need a search filter? (input, select, date picker)
- **CRUD operations**: Add? Edit? Delete? Any special operations?
- **API prefix**: The backend API path (e.g., `/basic/api/prisoner`)
- **Permission keys**: If using `v-permission`, what are the keys? (e.g., `sys:prisoner:add`)

If the user provides a rough description, infer reasonable defaults rather than asking about every detail. You can always refine later.

## Step 2: Generate the Main Page (index.vue)

Follow this exact structure. This is the canonical pattern from the project — stick to it closely so the generated code fits seamlessly.

### Template Structure

```vue
<template>
  <div id="<entity>-container">
    <div id="app-container">
      <el-card v-loading="loading">
        <!-- Search bar -->
        <div class="header-search">
          <div>
            <el-button v-permission="['<perm>:add']" type="primary" @click="openAddDialog">添加</el-button>
          </div>
          <el-form :inline="true" :model="pageVO">
            <el-form-item label="字段1">
              <el-input v-model="pageVO.field1" clearable placeholder="请输入字段1" />
            </el-form-item>
            <el-form-item label="字段2">
              <el-select v-model="pageVO.field2" filterable clearable placeholder="请选择">
                <!-- options here -->
              </el-select>
            </el-form-item>
            <el-form-item>
              <el-button type="primary" @click="onSubmit">查询</el-button>
            </el-form-item>
          </el-form>
        </div>
        <!-- Table -->
        <el-table :data="list" border height="75.7vh">
          <el-table-column type="index" :index="indexMethod" label="序号" width="60" />
          <el-table-column prop="field1" label="字段1" />
          <el-table-column prop="field2" label="字段2" />
          <!-- More columns -->
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
```

### Script Structure

```vue
<script>
import { getXxxApi, deleteXxxApi } from '@/api/<entity>'
import Add<Entity> from './components/add<Entity>'
import Edit<Entity> from './components/edit<Entity>'

export default {
  name: '<Entity>Page',
  components: { Add<Entity>, Edit<Entity> },
  data() {
    return {
      // Pagination + search params combined in one object
      pageVO: {
        pageNum: 1,
        pageSize: 10,
        // + search fields (initialized as '' or null)
        field1: '',
        field2: null
      },
      list: [],
      totalRows: 0,
      loading: false,
      addDialogVisible: false,
      editDialogVisible: false,
      currentRow: {}
    }
  },
  created() {
    this.getList()
  },
  methods: {
    async getList() {
      this.loading = true
      const res = await getXxxApi(this.pageVO)
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
    handleSizeChange(val) {
      this.pageVO.pageSize = val
      this.getList()
    },
    handleCurrentChange(val) {
      this.pageVO.pageNum = val
      this.getList()
    },
    openAddDialog() {
      this.addDialogVisible = true
    },
    openEditDialog(row) {
      this.currentRow = row
      this.editDialogVisible = true
    },
    handleDelete(id) {
      this.$confirm('亲，您确定要删除吗？', '温馨提示', { type: 'warning' })
        .then(async () => {
          await deleteXxxApi(id)
          this.$message.success('删除成功！')
          await this.getList()
        }).catch(() => {})
    },
    indexMethod(index) {
      return (this.pageVO.pageNum - 1) * this.pageVO.pageSize + index + 1
    }
  }
}
</script>
```

### Style

```vue
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

Each dialog lives in `src/views/<module>/<entity>/components/`.

### Add Dialog (addXxx.vue)

```vue
<template>
  <el-dialog title="添加" :visible.sync="addDialogVisible" width="600px" :before-close="handleClose">
    <el-form ref="formData" :model="formData" :rules="rules" label-width="100px">
      <el-form-item label="字段1" prop="field1">
        <el-input v-model="formData.field1" placeholder="请输入字段1" />
      </el-form-item>
      <!-- More form items -->
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
      formData: {
        field1: '',
        field2: ''
      },
      rules: {
        field1: [{ required: true, message: '请输入字段1', trigger: 'blur' }]
      }
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

### Edit Dialog (editXxx.vue)

The edit dialog is nearly identical to the add dialog, with these differences:
- Receives `:node` prop (the row data to edit)
- Watches `editDialogVisible` to populate form when dialog opens
- Calls `updateXxxApi` instead of `addXxxApi`
- Emits `edit-success` instead of `add-success`

```vue
<script>
import { updateXxxApi } from '@/api/<entity>'

export default {
  name: 'Edit<Entity>',
  props: {
    editDialogVisible: { type: Boolean, default: false },
    node: { type: Object, default: () => ({}) }
  },
  watch: {
    editDialogVisible(val) {
      if (val) {
        this.formData = { ...this.node }
      }
    }
  },
  // ... same data/methods as add, but uses updateXxxApi and emits edit-success
}
</script>
```

## Step 4: Generate the API File

Create `src/api/<entity>.js` following the project convention:

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

export function deleteXxxApi(id) {
  return request({ url: '/basic/api/<entity>/' + id, method: 'DELETE' })
}
```

Key conventions:
- All API functions end with `Api` suffix
- List endpoint uses **POST** with the pageVO as request body
- Delete uses **POST** with ID in the URL path
- Add uses POST, update uses POST
- Import from `@/utils/request` (the project's axios wrapper)

## Column Type Reference

When generating columns, choose the appropriate template based on the data type:

| Data Type | Column Template |
|-----------|----------------|
| Text (default) | `<el-table-column prop="name" label="姓名" />` |
| Index/序号 | `<el-table-column type="index" :index="indexMethod" label="序号" width="60" />` |
| Tag/状态 | `<el-table-column prop="status" label="状态"><template slot-scope="scope"><el-tag :type="scope.row.status === 1 ? 'success' : 'danger'">{{ scope.row.status === 1 ? '启用' : '禁用' }}</el-tag></template></el-table-column>` |
| Date/时间 | `<el-table-column prop="createTime" label="创建时间" width="160" />` |
| Image | `<el-table-column prop="avatar" label="头像" width="80"><template slot-scope="scope"><el-image :src="scope.row.avatar" style="width:40px;height:40px" fit="cover" /></template></el-table-column>` |
| Long text | `<el-table-column prop="remark" label="备注" show-overflow-tooltip />` |

## Search Form Component Reference

| Field Type | Search Component |
|------------|-----------------|
| Text | `<el-input v-model="pageVO.field" clearable placeholder="请输入" />` |
| Select | `<el-select v-model="pageVO.field" filterable clearable placeholder="请选择"><el-option v-for="item in options" :key="item.value" :label="item.label" :value="item.value" /></el-select>` |
| Date | `<el-date-picker v-model="pageVO.startTime" type="date" placeholder="开始时间" value-format="yyyy-MM-dd" />` |

## Important Conventions

- **Vue 2 Options API only** — this project runs Vue 2.6.10. Do not use `<script setup>`, Composition API, or Vue 3 syntax.
- **Pagination params in pageVO** — combine search fields and pagination (pageNum, pageSize) into one object so the entire query state is sent as a single POST body.
- **API response shape** — the backend returns `{ data: { list: [], total: 0, pageNum: 1, pageSize: 10 } }`. Always destructure from `res.data`.
- **Delete confirmation** — always use `this.$confirm('确定要删除吗？', '温馨提示', { type: 'warning' })` before deleting.
- **Dialog close** — always call `this.$refs.formData.resetFields()` before closing a dialog, and emit the close event so the parent can set the visible flag to false.
- **Permission directive** — use `v-permission="['key']"` on CRUD buttons if the project uses permission control.
- **Index method** — use the continuous index across pages: `(pageNum - 1) * pageSize + index + 1`.
- **Table** — use fixed header scrolling.
- **el-table border** — always add `border` attribute to el-table.

## File Output Checklist

When generating a complete table page, produce these files:

1. `src/views/<module>/<entity>/index.vue` — Main page (search + table + pagination)
2. `src/views/<module>/<entity>/components/add<Entity>.vue` — Add dialog
3. `src/views/<module>/<entity>/components/edit<Entity>.vue` — Edit dialog
4. `src/api/<entity>.js` — API functions

Tell the user what files were created and where, so they can verify and integrate.
