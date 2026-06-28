---
name: frontend-doc-creator
description: 为前端代码函数生成 JSDoc/TSDoc 注释文档。当用户提供代码目录并要求生成文档、添加函数注释、补全 JSDoc、写函数说明时使用此 skill。即使用户只说"给这个目录加文档"或"补全注释"，也应触发此 skill。
---

# Frontend Doc Creator

为前端代码中的函数自动生成 JSDoc/TSDoc 内联注释，直接写入源代码文件。

## 何时使用

当用户提供一个代码目录路径，并希望为其中的函数生成文档注释时触发。典型场景：

- "帮这个目录下的函数加文档注释"
- "给 src/utils 下的函数补全 JSDoc"
- "这些函数缺少注释，帮我补上"
- "生成函数的文档注释"

## 工作流程

### 1. 确认目标目录

向用户确认需要生成文档的代码目录路径。如果用户已明确给出路径，直接使用。

### 2. 扫描代码文件

遍历目标目录，识别以下文件类型：
- `.js`、`.jsx`、`.ts`、`.tsx`、`.mjs`
- `.vue`（提取 `<script>` 部分）
- `.svelte`（提取 `<script>` 部分）

忽略以下目录：`node_modules`、`dist`、`build`、`.git`、`coverage`

### 3. 识别函数

扫描每个文件，识别需要生成注释的函数：

- `function` 声明
- 箭头函数赋值（`const foo = () => {}`）
- 对象方法
- 类方法（`public`/`private`/`protected`）
- Vue `methods`/`setup` 中的函数
- React 函数组件和自定义 Hook

**跳过以下情况**（已有注释则不覆盖）：
- 函数上方已有 JSDoc/TSDoc 注释（`/** ... */`）
- 匿名回调函数（如 `.then(() => {})`、`addEventListener` 回调）
- 仅含 `return` 的简单箭头函数（一行以内）
- `constructor` 方法

### 4. 生成注释

为每个识别到的函数生成 JSDoc/TSDoc 注释，遵循以下规则：

**TypeScript 文件**：使用 TSDoc 风格，优先从类型定义中提取信息，注释聚焦于描述用途和业务含义，不重复类型信息。

```typescript
/**
 * 根据用户 ID 获取用户详细信息
 * @param userId - 用户唯一标识
 * @param options - 查询选项，控制是否加载关联数据
 * @returns 包含用户详细信息的 Promise
 * @throws 当用户不存在时抛出 NotFoundError
 */
async function getUserById(userId: string, options?: QueryOptions): Promise<UserDetail> {
```

**JavaScript 文件**：使用 JSDoc 风格，需补充类型信息。

```javascript
/**
 * 根据用户 ID 获取用户详细信息
 * @param {string} userId - 用户唯一标识
 * @param {Object} [options] - 查询选项
 * @param {boolean} [options.withProfile] - 是否加载用户画像
 * @returns {Promise<Object>} 包含用户详细信息的 Promise
 */
async function getUserById(userId, options) {
```

**注释内容要求**：
- **描述**：用一句话说明函数做什么，以动词开头（获取、计算、校验、转换、格式化等）。不写"该函数用于"这类冗余前缀。
- **@param**：每个参数必须有描述。如果参数是对象，用 `@param {type} obj.key` 逐个说明属性。
- **@returns**：描述返回值含义，不写"返回 xxx"而写具体内容。void 函数省略此标签。
- **@throws**：仅当函数内部有 throw 或可能抛异常时才添加。
- **@example**：仅当函数用途不直观或参数复杂时添加，不要每个函数都加。

### 5. 写入文件

将生成的注释插入到源代码文件中对应函数的上方。注意：

- 注释紧贴函数声明上方，中间不留空行
- 不修改函数本身的代码逻辑
- 不修改已有的导入语句
- 保留文件原有缩进风格
- 如果一个文件中有多个函数需要添加注释，一次性完成所有修改

### 6. 输出报告

完成所有文件处理后，向用户输出简要报告：

```
已为 12 个文件的 47 个函数生成文档注释。
跳过 3 个已有注释的函数。
```

如果某文件中存在无法确定用途的函数（如变量命名不规范、缺少上下文），在报告中列出，让用户确认。

## 注意事项

- 不覆盖已有的 JSDoc/TSDoc 注释
- 不为第三方库代码或 `node_modules` 中的文件生成注释
- 如果函数参数类型已经通过 TypeScript 声明，TSDoc 中不重复类型信息，聚焦于语义描述
- 尊重项目已有的注释风格——如果项目中的注释使用英文，保持英文；如果使用中文，保持中文
- 不要为了生成注释而修改代码逻辑或重构代码
