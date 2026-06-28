# 常见性能瓶颈检测模式

## 大组件检测

### 检测方法

1. **按行数扫描**：统计单文件组件（.vue/.jsx/.tsx）的行数
   - 300-500 行：标记为潜在问题
   - 500+ 行：标记为严重问题

2. **职责分析**：检查组件是否承担了过多职责
   - 同时包含数据获取、业务逻辑、UI 展示
   - 嵌套超过 3 层的条件渲染
   - 超过 10 个 props（React）或超过 15 个 data 属性（Vue）

3. **模板/JSX 复杂度**：
   - v-for 嵌套超过 2 层
   - 大量 v-if/v-else 分支
   - 模板中包含复杂表达式

### 典型模式

```
大组件信号：
- 文件行数 > 300
- 导入超过 10 个子组件
- methods/actions 超过 15 个
- computed 超过 10 个
- 模板中有多层嵌套的 v-for/v-if
```

### 优化建议

- 按职责拆分：容器组件 vs 展示组件
- 提取可复用子组件
- 提取业务逻辑到 composables/hooks
- 大列表使用虚拟滚动

---

## 重复渲染检测

### Vue 项目

**信号 1：未使用计算属性**
```vue
<!-- 问题：模板中调用方法，每次渲染都重新计算 -->
<div>{{ formatPrice(item.price) }}</div>

<!-- 优化：使用计算属性 -->
<div>{{ formattedPrice }}</div>
```

**信号 2：v-for 缺少 key 或使用 index 作 key**
```vue
<!-- 问题 -->
<div v-for="(item, index) in list" :key="index">

<!-- 优化：使用唯一标识 -->
<div v-for="item in list" :key="item.id">
```

**信号 3：深层响应式数据不必要地触发更新**
```javascript
// 问题：大对象的任意属性变化都触发更新
data() {
  return { bigConfig: { ... } }
}

// 优化：将频繁变化的部分拆开
data() {
  return {
    staticConfig: { ... },
    dynamicValue: null
  }
}
```

**信号 4：watch 使用 deep: true**
```javascript
// 问题：深度监听大对象，每次变化都遍历全部属性
watch: {
  bigObject: {
    handler() { ... },
    deep: true
  }
}

// 优化：只监听需要的属性，或使用 computed 派生
```

### React 项目

**信号 1：缺少 memo/useMemo/useCallback**
```jsx
// 问题：子组件每次父组件渲染都重渲染
<ListItem data={item} onClick={handleClick} />

// 优化：memo + useCallback
const ListItem = memo(({ data, onClick }) => ...);
const handleClick = useCallback(() => { ... }, [deps]);
```

**信号 2：Context 值引用不稳定**
```jsx
// 问题：每次渲染创建新对象，导致所有消费者重渲染
<ThemeContext.Provider value={{ theme, setTheme }}>

// 优化：useMemo 稳定引用
const value = useMemo(() => ({ theme, setTheme }), [theme, setTheme]);
```

**信号 3：内联函数作为 props**
```jsx
// 问题：每次渲染创建新函数引用
<Button onClick={() => handleClick(id)}>

// 优化：useCallback 或提取为独立函数
const handleButtonClick = useCallback(() => handleClick(id), [id]);
```

---

## 内存泄漏检测

### 模式 1：未清理的定时器

```javascript
// 问题
mounted() {
  this.timer = setInterval(() => { ... }, 1000);
  // 缺少 beforeDestroy 中的 clearInterval
}

// 优化
mounted() {
  this.timer = setInterval(() => { ... }, 1000);
},
beforeDestroy() {
  clearInterval(this.timer);
}
```

React 等价：
```javascript
useEffect(() => {
  const timer = setInterval(() => { ... }, 1000);
  return () => clearInterval(timer); // 清理函数
}, []);
```

### 模式 2：未移除的事件监听器

```javascript
// 问题
mounted() {
  window.addEventListener('resize', this.handleResize);
  // 缺少 beforeDestroy 中的 removeEventListener
}

// 优化
mounted() {
  window.addEventListener('resize', this.handleResize);
},
beforeDestroy() {
  window.removeEventListener('resize', this.handleResize);
}
```

### 模式 3：未取消的订阅

```javascript
// 问题：WebSocket/EventBus/Redux 订阅未取消
mounted() {
  this.ws = new WebSocket(url);
  this.ws.onmessage = (e) => { ... };
  // 组件卸载后 ws 仍在接收消息
}

// 优化
mounted() {
  this.ws = new WebSocket(url);
  this.ws.onmessage = (e) => { ... };
},
beforeDestroy() {
  this.ws.close();
}
```

### 模式 4：闭包持有过期引用

```javascript
// 问题：闭包捕获了组件实例，阻止 GC
const cache = {};
function process(id) {
  const component = getComponent(id); // 持有组件引用
  cache[id] = () => component.getData(); // 闭包持有引用
}

// 优化：只缓存需要的数据，不持有组件实例
const cache = {};
function process(id) {
  const data = getComponent(id).getData();
  cache[id] = data; // 只存数据
}
```

### 模式 5：detached DOM 节点

```javascript
// 问题：移除 DOM 后仍持有引用
const removedNode = document.getElementById('old');
container.removeChild(removedNode);
// removedNode 变量仍持有引用，GC 无法回收

// 优化：移除后释放引用
container.removeChild(removedNode);
removedNode = null;
```

---

## 资源加载瓶颈检测

### 图片相关

```html
<!-- 问题：首屏图片懒加载 -->
<img loading="lazy" src="hero-image.jpg">
<!-- 首屏的 LCP 图片不应该懒加载 -->

<!-- 优化 -->
<img loading="eager" fetchpriority="high" src="hero-image.jpg">

<!-- 问题：未声明尺寸 -->
<img src="photo.jpg" alt="photo">
<!-- 优化 -->
<img width="800" height="600" src="photo.jpg" alt="photo">
```

### 脚本加载

```html
<!-- 问题：阻塞渲染的脚本 -->
<script src="analytics.js"></script>

<!-- 优化：非关键脚本延迟加载 -->
<script async src="analytics.js"></script>
<!-- 或 -->
<script defer src="analytics.js"></script>
```

### CSS 加载

```html
<!-- 问题：所有 CSS 阻塞渲染 -->
<link rel="stylesheet" href="all-styles.css">

<!-- 优化：内联关键 CSS，延迟加载其余 -->
<style>/* 关键 CSS 内联 */</style>
<link rel="preload" href="rest-styles.css" as="style" onload="this.rel='stylesheet'">
```

### 代码分割检测

检查构建配置和路由配置：
- 是否按路由拆分代码
- 大型第三方库是否单独打包
- 是否使用 dynamic import 延迟加载

```javascript
// 问题：同步导入大型组件
import HeavyChart from './HeavyChart.vue';

// 优化：动态导入
const HeavyChart = () => import('./HeavyChart.vue');
```

---

## 检测流程总结

```
1. 扫描项目结构
   ├── 识别框架（Vue/React/其他）
   ├── 找到入口文件和路由配置
   └── 找到构建配置

2. 识别大文件（> 300 行的组件）

3. 逐组件检查
   ├── 渲染优化（计算属性/memo/key）
   ├── 内存管理（定时器/监听器/订阅清理）
   └── 依赖分析（大型依赖/重复依赖）

4. 检查资源加载策略
   ├── 图片优化和加载方式
   ├── 脚本加载方式
   └── 代码分割策略

5. 汇总问题并生成报告
```
