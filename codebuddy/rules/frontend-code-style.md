---
enabled: true
alwaysApply: false
paths: ["*.vue", "*.ts", "*tsx", "*.jsx", "*.js", "*.css", "*.scss", "*.sass"]
---

# 前端代码风格规范
- 始终使用两空格缩进
- 变量，函数命名遵循《代码整洁之道》

# javascript规范
- js，ts 规范遵循 airbnb 规范
- 使用 async和 await处理异步方法
- 不用回调写法
- 使用!判断 undefined 和 null

# vue规范
- <style>标签中超过100行需要提出到同级目录下的单独的样式文件中
- 使用v-model处理自定义组件、原生标签的双向绑定，不要用.sync
- v-if和v-for不要写在同一个组件上


## SFC
- 组件使用PascalCase风格的方式命名并创建文件夹
- 组件的子组件放到组件文件夹中的components文件夹里


# react规范

## react 项目
- 组件使用PascalCase风格的方式命名并创建文件夹，内部有如下组件
  - UI组件
  - 容器组件
  - Hook
  - service

# 样式规范
- 禁止直接覆写组件库组件原生样式，需要提示词指定覆写才可以覆写
- 不使用!important，使用选择权权重
