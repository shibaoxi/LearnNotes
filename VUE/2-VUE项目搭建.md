# VUE 项目搭建

## vue 项目目录介绍

```bash
D:.
├─.vscode
├─node_modules    # npm 下载的包的源文件
├─public    # 放引入别人的文件，基本不会动的文件，比如图片，通过绝对路径访问
└─src    
    ├─assets       # assets 放自己写的css、js文件，后期可能会改的文件，通过相对路径访问
    ├─components    # 存放vue组件的目录
    ├─App.vue    # 是主组件，是页面入口文件
    ├─main.ts    # 项目自动执行的TS脚本文件
    └─vite-env.d.ts # 环境变量配置文件
├─.gitignore
├─index.html    # 首页的入口html文件
├─package-lock.json    #记录了node_modules 目录下所有模块具体来源和版本号以及其他信息
├─package.json    # 记录当前项目所依赖模块的版本信息
├─README.md
├─tsconfig.json    # ts脚本语言的配置文件
├─tsconfig.node.json
└─vite.config.ts    # vite 工具的配置文件
```

## 页面如何加载

### index.html 文件

![20220727114803](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220727114803.png)

### main.ts 文件

```nodejs
import { createApp } from 'vue' //导入CreateApp函数
import './style.css'
import App from './App.vue' // 引入App.vue 文件，此文件是模块的入口文件

createApp(App).mount('#app')    //使用vue中的CreateApp方法创建一个app.vue的组件，并放置到index
```

### setup 语法糖

#### 组件的引入

```html
<script setup lang="ts">
import HelloWorld from './components/HelloWorld.vue'
</script>
```

#### 定义组件的props

```html
<script setup lang="ts">
import { ref } from 'vue'

defineProps<{ msg: string }>()

const count = ref(0)
</script>
```

### 使用ElementPlus

#### 安装 element plus

```bash
npm install element-plus
```

#### 在main.ts 中引入element-plus

```html
import ElementPlus from "element-plus"; // 引入ElementPlus
import 'element-plus/dist/index.css'; // 引入ElementPlus css

createApp(App).use(ElementPlus).mount('#app')    //使用vue中的CreateApp方法创建一个app.vue的组件，并放置到index
```

### 使用vue-router

#### 安装组件

```bash
npm install vue-router
```

#### 新建路由匹配文件

在项目目录下找到src，在src目录下新建router目录，然后创建 index.ts 文件。

```javascript
// 导入组件
import { createRouter, createWebHistory, RouteRecordRaw } from "vue-router";
import HelloWorld from '../component/HelloWorld.vue'
import Demo from '../components/Demo.vue'

// 创建路由匹配的数据集合 

const routers: Array<RouteRecordRaw> = [
    {
        path: "/",
        component: HelloWorld
    },
    {
        path: "/demo",
        component: Demo
    }
]

// 创建一个vue-router 的对象

const router = createRouter({
    history: createWebHistory(), 
    routes,
})

// 暴露

export default router
```

#### 在main.ts中注册

```javascript
import router from './router'; // 导入router文件

createApp(App).use(router).use(ElementPlus).mount('#app')    //使用vue中的CreateApp方法创建一个app.vue的组件，并放置到index
```

#### 在App.vue 文件中添加router-view 标签

```html
<template>
  <router-view></router-view>
</template>

```
