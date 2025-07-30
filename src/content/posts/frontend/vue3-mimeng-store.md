---
title: Vue 3 实战 - 迷梦甄选
published: 2025-03-05
description: 这是一个 Vue3 企业级实战项目笔记，涵盖了大部分实际开发中可能会使用到的库。
image: https://d988089.webp.li/2025/03/05/202503052302860.jpg
tags: ["前端", "Vue", "项目实战"]
category: 前端
draft: false
---

> 这是一个 Vue3 企业级实战项目笔记，涵盖了大部分实际开发中可能会使用到的库。
> 笔记中有自己的一些见解，与视频有部分差异。
> 视频教程可参考：[尚硅谷Vue项目实战硅谷甄选，vue3项目+TypeScript前端项目一套通关_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Xh411V7b5/)

技术栈：

- Vue 3
- TypeScript
- Vite
- Element Plus
- Tailwind CSS
- ~~SCSS~~
- Axios
	规范化开发：

	- ESLint
	- Prettier
	- StyleLint
	- Husky
	- CommitLint

## 项目前期准备

### 项目初始化

```sh
pnpm create vite
> MiMengStore
> mi-meng-store
> Vue
> TypeScript

cd MiMengStore
pnpm install
code .
pnpm run dev
```

- 修改 `index.html` 的 `title` ，删除 `style.css` ，清空 `App.vue` 内的多余内容。
- 修改 `package.json` 中的 dev 脚本为 `vite --open` ，自动打开浏览器。

### 项目配置

#### ESLint 校验代码工具

##### 安装

```sh
pnpm add eslint -D
```

```sh
npx eslint --init
```
```sh
√ How would you like to use ESLint? · problems
√ What type of modules does your project use? · esm
√ Which framework does your project use? · vue
√ Does your project use TypeScript? · typescript
√ Where does your code run? · browser
The config that you've selected requires the following dependencies:

eslint, globals, @eslint/js, typescript-eslint, eslint-plugin-vue
√ Would you like to install them now? · No / Yes
√ Which package manager do you want to use? · pnpm
```

 在`eslint.config.js` 中添加 `"vue/multi-word-component-names": "off"` 规则：

```js
import globals from "globals";
import pluginJs from "@eslint/js";
import tseslint from "typescript-eslint";
import pluginVue from "eslint-plugin-vue";
import { rules } from "eslint-plugin-vue";


/** @type {import('eslint').Linter.Config[]} */
export default [
  { files: ["**/*.{js,mjs,cjs,ts,vue}"] },
  { languageOptions: { globals: globals.browser } },
  pluginJs.configs.recommended,
  ...tseslint.configs.recommended,
  ...pluginVue.configs["flat/essential"],
  {
    files: ["**/*.vue"], languageOptions: { parserOptions: { parser: tseslint.parser } }, rules: {
      ...rules,
      "vue/multi-word-component-names": "off",
    }
  }
];
```

##### eslintignore 忽略文件

创建 `.eslintignore` 文件：

```
dist
node_modules

.vscode
```

##### eslint 脚本

添加以下脚本到 `package.json` ，`lint` 用于校验语法， `fix` 用于自动修补：

```json
"scripts": {
	"lint": "eslint src",
    "fix": "eslint src --fix"
}
```

#### Prettier 格式化工具

##### 安装

```sh
pnpm add -D eslint-config-prettier eslint-plugin-prettier prettier
```

##### .prettierrc 规则文件

```json
{
  "singleQuote": true,
  "semi": false,
  "tabWidth": 2
}
```

##### .prettierignore 忽略文件

```
**/*.svg
**/*.sh

.local

/dist/*
/node_modules/*
/public/* 
```

> 作为个人项目，下列 4 个没有太大的配置必要，这里不做详细说明，感兴趣可自行搜索。

#### StyleLint 样式校验工具（待填）

#### Husky Git 钩子（待填）

#### CommitLint （待填）

#### 统一包管理器（待填）

### 项目集成

#### Element Plus

> [一个 Vue 3 UI 框架 | Element Plus](https://cn.element-plus.org/zh-CN/)

##### 安装

```sh
pnpm install element-plus
```

##### 按需引入

> [快速开始 | Element Plus](https://cn.element-plus.org/zh-CN/guide/quickstart.html#%E6%8C%89%E9%9C%80%E5%AF%BC%E5%85%A5)

```sh
pnpm add -D unplugin-vue-components unplugin-auto-import
```

修改 `vite.config.ts`

```ts
import { defineConfig } from 'vite'
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers'

export default defineConfig({
  // ...
  plugins: [
    // ...
    AutoImport({
      resolvers: [ElementPlusResolver()],
    }),
    Components({
      resolvers: [ElementPlusResolver()],
    }),
  ],
})
```

##### Icon 图标

> [Icon 图标 | Element Plus](https://cn.element-plus.org/zh-CN/component/icon.html)

```sh
pnpm add @element-plus/icons-vue
```

###### 按需导入以及 Iconify 集成

```sh
pnpm add -D unplugin-icons
```

（可选）

```sh
pnpm add -D @iconify/json
```

修改 `vite.config.ts`

```ts
import { defineConfig } from 'vite'
import Icons from 'unplugin-icons/vite'
import IconsResolver from 'unplugin-icons/resolver'
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'

export default defineConfig({
  plugins: [
    AutoImport({
      resolvers: [
        // 自动导入图标组件
        IconsResolver({
          prefix: 'Icon',
        }),
      ],
    }),

    Components({
      resolvers: [
        // 自动注册图标组件
        IconsResolver({
          enabledCollections: ['ep'],
        }),
      ],
    }),
    Icons({
      autoInstall: true,
    }),
  ],
})
```

> 完整配置代码示例见 [element-plus-best-practices/vite.config.ts](https://github.com/sxzz/element-plus-best-practices/blob/db2dfc983ccda5570033a0ac608a1bd9d9a7f658/vite.config.ts#L21-L58)

##### i18n 国际化

> [国际化 | Element Plus](https://cn.element-plus.org/zh-CN/guide/i18n.html)

修改 `main.ts` ，改为中文

```ts
import { createApp } from 'vue'
import App from './App.vue'
import ElementPlus from 'element-plus'
import zhCn from 'element-plus/es/locale/lang/zh-cn'

const app = createApp(App)

app.use(ElementPlus, {
  locale: zhCn,
})

app.mount('#app')
```

#### src 别名路径

 修改 `vite.config.ts` ：
`
```ts
import { defineConfig } from 'vite'
export default defineConfig({
  resolve: {
    alias: {
      // 配置别名
      '@': '/src',
    }
  }
})
```

修改 `tsconfig.json` ：

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"] 
    }
  }
}
```

#### 环境变量

创建以下三个文件，分别为开发、生产、测试环境：

- `.env.development`
- `.env.production`
- `.env.test`

```env
# 变量必须以 VITE_ 为前缀才能暴露给外部读取
NODE_ENV = 'development'
VITE_APP_TITLE = '迷梦甄选'
VITE_APP_BASE_URL = '/dev-api'
```

```env
NODE_ENV = 'production'
VITE_APP_TITLE = '迷梦甄选'
VITE_APP_BASE_URL = '/prod-api'
```

```env
NODE_ENV = 'test'
VITE_APP_TITLE = '迷梦甄选'
VITE_APP_BASE_URL = '/test-api'
```

可通过 `console.log(import.meta.env)` 查看环境变量是否被正常加载。

package.json 添加以下两条脚本：

```json
  "scripts": {
    "build:prod": "vue-tsc -b && vite build --mode production",
    "build:test": "vue-tsc -b && vite build --mode test"
  },
```

#### SVG 图标集成

```sh
pnpm add -D vite-plugin-svg-icons
```

修改 `vite.config.ts`

```ts
import { createSvgIconsPlugin } from 'vite-plugin-svg-icons'
import path from 'path'
export default () => {
  return {
    plugins: [
      createSvgIconsPlugin({
        iconDirs: [path.resolve(process.cwd(), 'src/assets/icons')],
        symbolId: 'icon-[dir]-[name]',
      }),
    ],
  }
}
```

在入口文件 `main.ts` 中引入

```ts
//@ts-expect-error: virtual module for SVG icons registration
import 'virtual:svg-icons-register'
```
##### 自定义组件（SvgIcon.vue）

```vue
<template>
  <div>
    <svg :style="{ width: width, height: height }">
      <use :xlink:href="prefix + name" :fill="color"></use>
    </svg>
  </div>
</template>

<script setup lang="ts">
defineProps({
  //xlink:href属性值的前缀
  prefix: {
    type: String,
    default: '#icon-'
  },
  //svg矢量图的名字
  name: String,
  //svg图标的颜色
  color: {
    type: String,
    default: ""
  },
  //svg宽度
  width: {
    type: String,
    default: '16px'
  },
  //svg高度
  height: {
    type: String,
    default: '16px'
  }

})
</script>
<style scoped></style>
```

在 `components` 目录下创建 `index.ts` ：

```ts
import type { App } from "vue"
import SvgIcon from "./SvgIcon.vue"

const allGlobalComponents = { SvgIcon }

export default {
  install: (app: App) => {
    for (const key in allGlobalComponents) {
      console.log(key)
      app.component(key, allGlobalComponents[key as keyof typeof allGlobalComponents])
    }
  }
}
```

在 `main.ts` 入口文件中注册为全局组件，避免频繁导入：

```ts
import globalConponents from '@/components'
app.use(globalConponents)
```

#### TailWind CSS 集成

> [Installing Tailwind CSS with Vite - Tailwind CSS](https://tailwindcss.com/docs/installation/using-vite)

> 如果你更加习惯使用 Sass ，可以跳过这部分，采用后面的集成方案

```
pnpm add tailwindcss @tailwindcss/vite
```

修改 `vite.config.ts` ：
```ts
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'
export default defineConfig({
  plugins: [
    tailwindcss(),
  ],
})
```

创建 `styles` 目录，创建 `index.css` 文件

在 `main.ts` 中引入样式文件

```ts
import './styles/index.css'
```

在 `index.css` 中 引入 `tainwindcss`

```scss
@import "tailwindcss";
```

#### ~~Sass 样式集成~~

创建 `styles` 目录，新建 `index.scss` 、 `variable.scss` 文件

在 `main.ts` 中引入样式文件

```ts
import './styles/index.scss'
```

修改 `vite.config.ts` 以支持全局变量：

```ts
export default defineConfig((config) => {
	css: {
      preprocessorOptions: {
        scss: {
          additionalData: '@import "/src/styles/variable.scss";',
        },
      },
    },
})
```

#### Normalize.css

```sh
pnpm add normalize.css
```

在 `main.ts` 引入：

```ts
import 'normalize.css'
```

#### Mock 接口

```sh
pnpm add -D vite-plugin-mock mockjs
```

修改 `vite.config.ts` 

```ts
import { defineConfig } from 'vite'
import { viteMockServe } from 'vite-plugin-mock'
export default ({ command })=> {
  return {
    plugins: [
      viteMockServe({
        localEnabled: command === 'serve',
      }),
    ],
  }
}
```

在根目录创建 `mock` 文件夹，创建 `user.ts` :

```ts
//用户信息数据
function createUserList() {
    return [
        {
            userId: 1,
            avatar:
                'https://wpimg.wallstcn.com/f778738c-e4f8-4870-b634-56703b4acafe.gif',
            username: 'admin',
            password: '111111',
            desc: '平台管理员',
            roles: ['平台管理员'],
            buttons: ['cuser.detail'],
            routes: ['home'],
            token: 'Admin Token',
        },
        {
            userId: 2,
            avatar:
                'https://wpimg.wallstcn.com/f778738c-e4f8-4870-b634-56703b4acafe.gif',
            username: 'system',
            password: '111111',
            desc: '系统管理员',
            roles: ['系统管理员'],
            buttons: ['cuser.detail', 'cuser.user'],
            routes: ['home'],
            token: 'System Token',
        },
    ]
}

export default [
    // 用户登录接口
    {
        url: '/api/user/login',//请求地址
        method: 'post',//请求方式
        response: ({ body }) => {
            //获取请求体携带过来的用户名与密码
            const { username, password } = body;
            //调用获取用户信息函数,用于判断是否有此用户
            const checkUser = createUserList().find(
                (item) => item.username === username && item.password === password,
            )
            //没有用户返回失败信息
            if (!checkUser) {
                return { code: 201, data: { message: '账号或者密码不正确' } }
            }
            //如果有返回成功信息
            const { token } = checkUser
            return { code: 200, data: { token } }
        },
    },
    // 获取用户信息
    {
        url: '/api/user/info',
        method: 'get',
        response: (request) => {
            //获取请求头携带token
            const token = request.headers.token;
            //查看用户信息是否包含有次token用户
            const checkUser = createUserList().find((item) => item.token === token)
            //没有返回失败的信息
            if (!checkUser) {
                return { code: 201, data: { message: '获取用户信息失败' } }
            }
            //如果有返回成功信息
            return { code: 200, data: {checkUser} }
        },
    },
]
```
 
#### Axios 网络请求库

```sh
pnpm add axios
```

测试 Mock 接口：

```ts
import axios from 'axios'

axios({
  method: 'post',
  url: '/api/user/login',
  data: {
    username: 'admin',
    password: '111111',
  },
}).then((res) => {
  console.log(res.data)
})
```

能够看到如下输出：

```json
{
    "code": 200,
    "data": {
        "token": "Admin Token"
    }
}
```

##### 二次封装

创建 `utils/http.ts` ：

```ts
import axios from 'axios'
//创建axios实例
const http = axios.create({
  baseURL: import.meta.env.VITE_APP_BASE_URL,
  timeout: 5000,
})
//请求拦截器
http.interceptors.request.use((config) => {
  return config
})
//响应拦截器
http.interceptors.response.use(
  (response) => {
    return response.data
  },
  (error) => {
    //处理网络错误
    let msg = ''
    const status = error.response.status
    switch (status) {
      case 401:
        msg = 'token过期'
        break
      case 403:
        msg = '无权访问'
        break
      case 404:
        msg = '请求地址错误'
        break
      case 500:
        msg = '服务器出现问题'
        break
      default:
        msg = '无网络'
    }
    ElMessage.error(msg)
    return Promise.reject(error)
  },
)
export default http
```

 如果你设置了 **Element Plus** 按需引入，上面的代码中 `ElMessage` 可能会报错，请不要添加 `import { ElMessage } from "element-plus";` ，这会导致样式丢失！
可在 `tsconfig.json` 的 **include** 添加 `auto-imports.d.ts`：

```json
{
  "include": ["auto-imports.d.ts"]
}
```

为了能成功请求，需要修改开发环境变量 `VITE_APP_BASE_URL` 为 `/api` 。

#### Pinia 状态管理

```sh
pnpm add pinia
```

创建 `store/index.ts` ：

```ts
import { createPinia } from 'pinia'

const pinia = createPinia()

export default pinia
```

在 `main.ts` 中使用：

```ts
import pinia from './store'
app.use(pinia)
```

## 正式开发

在此之前可以先在 `index.css` 中编写一些基础样式，例如：

```css
body {
  width: 100vw;
  height: 100vh;
}

#app {
  width: 100%;
  height: 100%;
}

#app main {
  width: 100%;
  height: 100%;
}
```

### API 接口统一管理

创建 `api/user.ts`  ：

```ts
import http from '@/utils/http'

//项目用户相关的请求地址
enum API {
  LOGIN_URL = '/user/login',
  USERINFO_URL = '/user/info',
  LOGOUT_URL = '/user/logout',
}

// 登录表单数据类型
export interface loginFormData {
  username: string
  password: string
}

// 登录响应数据类型
export interface loginResponseData {
  code: number
  data: {
    token: string
  }
}

export interface userInfo {
  userId: number
  avatar: string
  username: string
  desc: string
  roles: string[]
  buttons: string[]
  routes: string[]
  token: string
}

// 用户信息响应数据类型
export interface userInfoReponseData {
  code: number
  data: userInfo
}

//登录接口
export const LoginAPI = (data: loginFormData) =>
  http.post<loginFormData, loginResponseData>(API.LOGIN_URL, data)

//获取用户信息
export const UserInfoAPI = () => http.get<userInfoReponseData>(API.USERINFO_URL)

//退出登录
export const LogoutAPI = () => http.post(API.LOGOUT_URL)
```

### 基础路由配置

```sh
pnpm add vue-router
```

在 `views` 目录下分别创建 `404` 、 `home` 、 `login` 目录，并分别创建 `index.vue` ，编写基础内容：

```vue
<template>
  <div>
    <h1>Home</h1>
  </div>
</template>

<script setup lang="ts">

</script>

<style lang="scss" scoped></style>
```

创建 `router/routes.ts` ，暴露一个常量路由：

```ts
export const constantRoutes = [
  {
    path: '/login',
    component: () => import('@/views/login/index.vue'),
    name: 'login',
  },
  {
    path: '/',
    component: () => import('@/views/home/index.vue'),
    name: 'home',
  },
  {
    path: '/404',
    component: () => import('@/views/404/index.vue'),
  },
  {
    path: '/:pathMatch(.*)*',
    redirect: '/404',
  },
]
```

创建 `router/index.ts`：

```ts
import { createRouter, createWebHashHistory } from 'vue-router'
import { constantRoutes } from './routes'

const router = createRouter({
  history: createWebHashHistory(),
  routes: constantRoutes,
  scrollBehavior: () => ({ left: 0, top: 0 }),
})

export default router
```

在 `main.ts` 中使用路由

```ts
import router from './router'
app.use(router)
```

在 `App.vue` 添加 `<router-view />` ：

```vue
<template>
  <main>
    <router-view />
  </main>
</template>
```

### 登录页

> 见 `src/views/login/index.vue`

编写基础样式后，为 Button 添加 `@click` 事件绑定到 `login` 函数，并且处理登录请求成功与失败情况。

创建 `store/modules/user.ts` ，实现以下功能：

- 用户信息持久化存储
- 请求登录方法

```ts
import { LoginAPI, loginFormData } from '@/api/user'
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => {
    return {
      token: localStorage.getItem('token') || null,
    }
  },
  getters: {},
  actions: {
    async login(loginForm: loginFormData): Promise<boolean> {
      const res = await LoginAPI(loginForm)
      if (res.code === 200) {
        this.token = res.data.token
        localStorage.setItem('token', res.data.token)
        return true
      } else {
        return false
      }
    },
  },
})
```

编写 login 函数：

```ts
import { useUserStore } from '@store/modules/user'
const userStore = useUserStore()
const login = async () => {
  try {
    const result = await userStore.login(loginForm);
    if (result) {
      $router.replace('/');
    }
  } catch (error) {
    console.log(error);
  }
}
```

> 后续内容流程类似，不做详细记录，具体参见视频教程。