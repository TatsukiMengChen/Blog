---
title: 为你的 Fuwari 集成 Mermaid 图表支持
published: 2025-08-03
description: 详细介绍如何在 Fuwari 中集成 Mermaid 图表支持，采用客户端渲染方式，完美适配深色模式切换。
image: https://d988089.webp.li/2025/08/03/20250803172528859.jpg
tags: ["前端", "Fuwari", "Astro", "Mermaid"]
category: 前端
draft: false
---

> 本文记录了在 Fuwari 主题中集成 Mermaid 图表的完整实现过程。通过 MutationObserver 监听主题切换，实现图表与深色/浅色模式的完美同步。

## 效果展示

首先看看最终的效果：

```mermaid
graph LR
    A[Markdown 代码块] --> B[Remark 插件处理]
    B --> C[转换为 HTML 容器]
    C --> D[Rehype 插件注入脚本]
    D --> E[客户端动态渲染]
    
    style A fill:#e1f5fe
    style E fill:#c8e6c9
```

## 技术方案

### 整体架构

本方案采用客户端渲染，通过以下组件实现：

1. **Remark 插件** - 识别 Mermaid 代码块
2. **Rehype 插件** - 生成 HTML 容器和渲染脚本  
3. **渲染脚本** - 动态加载 Mermaid 库并处理主题切换
4. **样式文件** - 提供响应式布局和主题适配

### 主题切换机制

使用 `MutationObserver` 监听 `document.documentElement` 的 `class` 变化，当检测到 `dark` 类的添加或移除时，自动重新渲染图表以匹配新主题。

## 核心代码文件

所有实现代码都可以在 GitHub 仓库中找到：

### 插件文件

1. **[remark-mermaid.js](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/plugins/remark-mermaid.js)** - Remark 插件，识别并转换 Mermaid 代码块
2. **[rehype-mermaid.mjs](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/plugins/rehype-mermaid.mjs)** - Rehype 插件，生成 HTML 容器和注入渲染脚本
3. **[mermaid-render-script.js](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/plugins/mermaid-render-script.js)** - 客户端渲染脚本，处理图表渲染和主题同步

### 配置文件

4. **[astro.config.mjs](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/astro.config.mjs)** - Astro 配置，注册插件
5. **[markdown-extend.styl](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/styles/markdown-extend.styl)** - 样式文件，包含 Mermaid 相关样式

## 快速集成步骤

### 1. 复制插件文件

将以下文件复制到你的项目中：

- [`src/plugins/remark-mermaid.js`](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/plugins/remark-mermaid.js)
- [`src/plugins/rehype-mermaid.mjs`](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/plugins/rehype-mermaid.mjs)  
- [`src/plugins/mermaid-render-script.js`](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/plugins/mermaid-render-script.js)

### 2. 配置 Astro

在 `astro.config.mjs` 中添加插件：

```javascript
import { remarkMermaid } from "./src/plugins/remark-mermaid.js";
import { rehypeMermaid } from "./src/plugins/rehype-mermaid.mjs";

export default defineConfig({
    markdown: {
        remarkPlugins: [
            // ... 其他插件
            remarkMermaid,
        ],
        rehypePlugins: [
            // ... 其他插件  
            rehypeMermaid,
        ],
    },
});
```

### 3. 添加样式

复制 [样式文件](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/styles/markdown-extend.styl) 中 Mermaid 相关的样式到你的样式文件中。

### 4. 开始使用

在 Markdown 文件中使用：

````markdown
```mermaid
graph TD
    A[开始] --> B[结束]
```
````

## 关键实现细节

### 1. 单例模式防重复初始化

渲染脚本使用单例模式确保 Mermaid 只初始化一次：

```javascript
if (window.mermaidInitialized) {
    return;
}
window.mermaidInitialized = true;
```

### 2. 智能主题检测

通过 `MutationObserver` 监听 DOM 变化，精确检测主题切换：

```javascript
const observer = new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
        if (mutation.type === "attributes" && mutation.attributeName === "class") {
            const wasDark = mutation.oldValue?.includes("dark") || false;
            const isDark = mutation.target.classList.contains("dark");
            
            if (wasDark !== isDark) {
                renderMermaidDiagrams(); // 重新渲染图表
            }
        }
    });
});
```

### 3. 批量渲染优化

使用 `Promise.all` 并行渲染多个图表，提升性能：

```javascript
const renderPromises = Array.from(mermaidElements).map(async (element) => {
    // 渲染单个图表
});
await Promise.all(renderPromises);
```

### 4. 防并发渲染

通过标志位防止并发渲染造成的问题：

```javascript
let isRendering = false;

function renderMermaidDiagrams() {
    if (isRendering) return;
    isRendering = true;
    // 渲染逻辑...
    isRendering = false;
}
```

## 图表示例

### 流程图

```mermaid
graph LR
    A[开始] --> B{条件判断}
    B -->|条件1| C[处理1]
    B -->|条件2| D[处理2]
    C --> E[结束]
    D --> E
```

### 序列图

```mermaid
sequenceDiagram
    participant 用户
    participant 浏览器
    participant 服务器
    participant 数据库
    
    用户->>浏览器: 输入网址
    浏览器->>服务器: 发送HTTP请求
    服务器->>数据库: 查询数据
    数据库-->>服务器: 返回数据
    服务器-->>浏览器: 返回HTML
    浏览器-->>用户: 显示页面
```

### 类图

```mermaid
classDiagram
    class Animal {
        +String name
        +int age
        +makeSound()
    }
    
    class Dog {
        +String breed
        +bark()
    }
    
    class Cat {
        +String color
        +meow()
    }
    
    Animal <|-- Dog
    Animal <|-- Cat
```

### 甘特图

```mermaid
gantt
    title 项目开发时间线
    dateFormat  YYYY-MM-DD
    section 需求分析
    需求收集           :a1, 2025-08-01, 2025-08-05
    需求整理           :a2, after a1, 3d
    section 设计阶段
    UI设计            :b1, 2025-08-08, 7d
    数据库设计         :b2, 2025-08-08, 5d
    section 开发阶段
    前端开发          :c1, 2025-08-15, 14d
    后端开发          :c2, 2025-08-15, 10d
    section 测试阶段
    功能测试          :d1, after c1, 5d
    性能测试          :d2, after d1, 3d
```

### 饼图

```mermaid
pie
    title 技术栈分布
    "JavaScript" : 35
    "TypeScript" : 25
    "CSS" : 20
    "HTML" : 20
```

## 参考资料

- [Mermaid 官方文档](https://mermaid.js.org/) - 图表语法和配置
- [Astro 官方文档](https://docs.astro.build/) - 插件开发指南
- [Fuwari 主题](https://github.com/saicaca/fuwari) - 主题源码和文档
- [完整实现代码](https://github.com/TatsukiMengChen/Blog/tree/feature/mermaid/src/plugins) - 本文所有插件代码
