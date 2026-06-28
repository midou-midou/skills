# Web Vitals 指标参考

## 核心指标（Core Web Vitals）

### LCP — Largest Contentful Paint

**含义**：最大内容绘制时间，衡量页面主要内容加载速度。

**评分标准**：
- 好：≤ 2.5s
- 需要改进：2.5s - 4.0s
- 差：> 4.0s

**代码层面常见问题**：
1. 首屏大图未压缩或未使用现代格式（WebP/AVIF）
2. 关键 CSS/JS 阻塞渲染
3. 服务端响应慢导致资源加载延迟
4. 图片未设置 width/height 导致布局偏移后重新绘制
5. 自定义字体加载阻塞文本渲染
6. 客户端渲染首屏内容依赖大量 JS 执行

**优化方向**：
- 图片优化：使用 `loading="eager"` 标记首屏图片，配合 srcset 提供合适尺寸
- 资源预加载：`<link rel="preload">` 关键资源
- 字体优化：`font-display: swap`，预加载关键字体
- 减少关键路径资源：内联关键 CSS，延迟非关键 JS
- 考虑 SSR/SSG 提升首屏渲染速度
- 使用 `fetchpriority="high"` 标记 LCP 图片

---

### FID — First Input Delay

**含义**：首次输入延迟，衡量页面首次交互响应速度。

**评分标准**：
- 好：≤ 100ms
- 需要改进：100ms - 300ms
- 差：> 300ms

**代码层面常见问题**：
1. 首屏加载时执行大量同步 JS
2. 第三方脚本阻塞主线程
3. 复杂的初始化逻辑未延迟执行
4. 大型依赖未按需加载

**优化方向**：
- 代码拆分：按路由/功能拆分，延迟加载非首屏代码
- 长任务拆分：使用 `requestIdleCallback` 或 `setTimeout(fn, 0)` 将长任务分片
- 第三方脚本延迟加载：使用 `async` 或动态加载
- 使用 Web Worker 处理计算密集型任务
- 减少首屏 JS 体积：tree-shaking、移除未使用依赖

---

### CLS — Cumulative Layout Shift

**含义**：累积布局偏移，衡量页面视觉稳定性。

**评分标准**：
- 好：≤ 0.1
- 需要改进：0.1 - 0.25
- 差：> 0.25

**代码层面常见问题**：
1. 图片/视频未设置 width/height 属性
2. 动态插入内容推挤已有元素（广告、推荐模块）
3. 字体加载导致文本闪烁和重排（FOIT/FOUT）
4. 异步加载的组件未预留占位空间
5. CSS 动画使用布局属性（top/left/width/height）而非 transform

**优化方向**：
- 图片/视频始终声明尺寸：`width`/`height` 或 `aspect-ratio`
- 为动态内容预留空间：使用固定高度容器或骨架屏
- 字体优化：`font-display: swap` + 尺寸匹配的回退字体
- 动画使用 `transform` 和 `opacity`（不触发 layout）
- 使用 CSS `contain` 属性限制布局影响范围

---

### INP — Interaction to Next Paint

**含义**：交互到下一次绘制的时间，衡量页面整体交互响应性（FID 的替代指标）。

**评分标准**：
- 好：≤ 200ms
- 需要改进：200ms - 500ms
- 差：> 500ms

**代码层面常见问题**：
1. 事件处理函数执行时间过长
2. 状态更新触发大范围重渲染
3. DOM 操作量大导致重排/重绘
4. 同步的计算密集型逻辑
5. 频繁的事件触发未做防抖/节流

**优化方向**：
- 拆分长任务：将事件处理中的耗时操作延迟或分片
- 减少重渲染范围：React memo/useMemo，Vue computed/v-memo
- 虚拟化长列表：只渲染可视区域
- 防抖/节流高频事件：scroll、resize、input
- 耗时计算使用 Web Worker
- 使用 `scheduler.yield()` 让出主线程

---

### TTFB — Time to First Byte

**含义**：首字节时间，衡量服务端响应速度。

**评分标准**：
- 好：≤ 800ms
- 需要改进：800ms - 1800ms
- 差：> 1800ms

**代码层面常见问题**：
1. API 调用瀑布流（串行请求依赖链）
2. SSR 渲染阻塞（大量数据查询）
3. 未使用 CDN
4. 服务端未做缓存

**优化方向**：
- 减少串行请求：并行化 API 调用、数据预取
- SSR 优化：流式渲染、缓存策略
- CDN 部署静态资源和 API
- 服务端缓存：Redis、HTTP 缓存头
- 使用 `<link rel="preconnect">` 预连接关键域名
- 使用 `<link rel="dns-prefetch">` 预解析域名

---

## 指标关联关系

```
TTFB ──→ LCP
  │         ▲
  │         │
  └──→ FID/INP

资源加载 ──→ LCP
          ──→ CLS (布局偏移)
JS 执行 ──→ FID/INP
         ──→ LCP (阻塞渲染)
```

- TTFB 慢通常会导致 LCP 慢
- JS 体积大同时影响 LCP（阻塞渲染）和 FID/INP（阻塞交互）
- 资源加载和 JS 执行都可能间接导致 CLS
