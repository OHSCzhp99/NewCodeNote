## Vue3
---
### 1. 使用 pnpm 创建Vue3：  
先要进行全局安装pnpm：  
`npm install -g pnpm`

pnpm创建vue：   
`pnpm create vue`  
选择必要的（4个YES）：  
VueRouter + Pinia  
ESLint + Prettier  

创建完成后cd 项目，进行依赖首次安装（下面有命令）

pnpm安装依赖：   
`pnpm install`

### 2. ESLint和prettier插件的配置：
settings.json文件：
```java
//ESLint插件 + Vscode配置 实现自动格式化修复
"editor.codeActionsOnSave": {
    "source.fixAll": true
},
//关闭保存自动格式化（防止与上面冲突）
"editor.formatOnSave": false,
```
.eslintrc.cjs文件：
```java
rules: {
    //prettier插件配置：
    //禁用格式化插件 prettier ，并且关闭settings.json中的format on save
    'prettier/prettier': [
        'warn',
        {
        singleQuote: true, //单引号
        semi: false, //无分号
        printWidth: 80, //每行宽度至多80字符
        trailingComma: 'none', //不加对象/数组最后的逗号
        endOfLine: 'auto' //换行符号不限制（win mac 不一致）
        }
    ],
    //ESLint插件配置：
    //安装ESLint插件，并配置保存时自动修复
    'vue/multi-word-component-names': [
        'warn',
        {
        ignores: ['index'] //vue组件名称多单词组成 (忽略index.vue)
        }
    ],
    'vue/no-setup-props-destructure': ['off'], //关闭props 解构的校验(props解构丢失响应式)
    'no-undef': 'error' //未定义变量错误提示
}
```

### 3. 目录文件夹结构的清理
```java
清空：
assets
components
router（只清空index.js里面routes[]）
stores
views

整理：
App.vue（只留stript、template(写个div)、style标签）
main.js（删掉assets的导入）

增加：
utils（工具相关，axios，设置后端的接口地址以及拦截等）
api（请求接口相关，对后端的接口进行打包 给前台用）
assets -> main.scss（新建main.scss）
```
main.scss文件：
```css
body{
    margin: 0;
    background-color: #fff;
}

.fade-slide-leave-active,
.fade-slide-enter-active{
    transition: all 0.3ms;
}

.fade-slide-enter-from{
    transform: translateX(-30px);
    opacity: 0;
}

.fade-slide-leave-to{
    transform: translateX(30px);
    opacity: 0;
}
```

### 4. Router4 语法
```java
vite.config.js中加入 base: '/'（跟plugins同级）
```

### 5. ElemenetPlus 导入
安装ElemenetPlus：  
`pnpm add element-plus`

安装按需导入：（能够让组件模板之间、elementUI组件直接使用，不用手动导入）  
`pnpm add -D unplugin-vue-components unplugin-auto-import`

在vite.config.js配置文件中导入：
```js
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

### 6. Pinia 持久化和独立维护
因为在创建vue3时已经安装了Pinia，这里不用再安装了  
但是还没安装Pinia持久化，需安装：  
`pnpm add pinia-plugin-persistedstate -D`  

在stores创建 index.js：（专门用于Pinia的统一维护、导出）  
将main.js中的关于Pinia的移除，统一交给上面的index.js管理  
并且在main.js中导入index.js  
```js
//index.js页面
import { createPinia } from 'pinia' //Pinia依赖包
import persist from 'pinia-plugin-persistedstate' //Pinia持久化

const pinia = createPinia() //创建Pinia实例
pinia.use(persist) //使用Pinia持久化

export default pinia //将index.js文件暴露出去

//将两个仓库暴露出去，让组件只用导入index.js也能使用仓库----（下面会创建这两个仓库）
export * from './modules/user' 
export * from './modules/counter'

```
```js
//main.js页面
import pinia from '@/stores/index' //导入Pinia统一管理的index.js
app.use(pinia)
```
创建仓库：  
在stores创建 modules 文件夹  
在modules -> 创建 user.js（用于用户登录携带的token）  
在modules -> 创建 counter.js（用于测试而已）  
```js
//user.js页面
import { defineStore } from 'pinia'
import { ref } from 'vue'

//用户模块 token setToken removeToken
export const useUserStore = defineStore('userStoe', () => {
  const token = ref('')
  const setToken = (newToken) => {
    token.value = newToken
  }
  const removeToken = () => {
    token.value = ''
  }

  return {
    token,
    setToken,
    removeToken
  }
})
```
组件使用：  
```js
import { useUserStore,useCountStore } from '@/stores' //组件只需导入Pinia的index.js
const userStore = useUserStore()
const countStore = useCountStore()
```

### 7. axios 请求拦截
axios配置： （读作：A可修斯）  
创建axios实例 - 基准地址、超时时间  
请求拦截器 - 携带token  
响应拦截器 - 业务失败处理、摘取核心响应数据、401处理  

安装axios：  
`pnpm add axios`

在utils中创建 request.js：
```js
import axios from 'axios'
import { useUserStore } from '@/stores'
import { ElMessage } from 'element-plus'
import router from '../router'

const baseURL = '/api' //基础地址(不写http://localhost:8080 可解决跨域问题)

const instance = axios.create({
  baseURL,
  timeout: 10000
})

//请求拦截器
instance.interceptors.request.use(
  (config) => {
    const userStore = useUserStore() //用仓库
    if (userStore.token) {
      //如果携带token则在加在请求头上
      config.headers.Authorization = userStore.token
    }
    return config
  },
  (err) => Promise.reject(err)
)

//响应拦截器
instance.interceptors.response.use(
  (res) => {
    //处理业务成功 状态码为 0
    if (res.data.code === 0) {
      return res.data
    }
    //处理业务失败 给错误提示，抛出错误
    ElMessage.error(res.data.message || '服务异常') //利用elementPlus弹窗给出错误提示
    return Promise.reject(res.data)
  },
  (err) => {
    //处理错误：401权限不足 或者 token过期 --> 拦截到登录页
    if (err.response?.status === 401) {
      router.push('/login')
    }

    //错误的默认情况 - 只给提示
    ElMessage.error(err.response.data.message || '服务异常')
    return Promise.reject(err)
  }
)

export default instance
export { baseURL }
```

### 8.1 后台管理 - 页面路由配置
这里只记录当前步骤，后续肯定会添加更多
```js
routes: [
    //登录路由
    { path: '/login', component: () => import('@/views/login/LoginPage.vue') },
    {
        //布局路由
        path: '/',
        component: () => import('@/views/layout/LayoutContainer.vue'),
        redirect: '/manage/user',
        children: [
        {
            path: '/manage/user',
            component: () => import('@/views/manage/ManageUser.vue')
        },
        {
            path: '/manage/post',
            component: () => import('@/views/manage/ManagePost.vue')
        },
        {
            path: '/manage/menu',
            component: () => import('@/views/manage/ManageMenu.vue')
        },
        {
            path: '/manage/link',
            component: () => import('@/views/manage/ManageLink.vue')
        },
        {
            path: '/manage/person',
            component: () => import('@/views/manage/ManagePerson.vue')
        }
        ]
    }
]
```

### 8.2 后台管理 - 登录页
先安装ElementPlus图标库：  
`pnpm install @element-plus/icons-vue`  
使用ElementPlus中的 Form 表单   

这里先了解一下： 
ElementPlus的布局：  
el-row 表示一行，一行分成24份  
el-col 表示列   
:span="12"  代表在一行中，占一半  
:offset="3" 代表在一行中，左侧margin数（空出3份） 

label="请输入用户名" 代表元素的显示名字  
label-width="120px" 代表元素左侧距离  
:prefix-icon="Lock" 代表input左侧的小图标，Lock为图标名字可以在官网找  


校验规则和绑定：  
el-form => :model="ruleForm" 绑定整个form的数据对象  
el-form => :rules="rules" 绑定整个form的规则对象  
el-form-item => prop="xxx" 配置生效的是哪个规则  
表单元素 => :v-model="ruleForm.xxx" 给表单元素，绑定form的子属性  

```vue
参考毕设中的 LoginPage 页面
```

### 8.3 后台管理 - 内容页（架子+登录拦截）
使用ElementPlus中的 Container 布局容器、Menu 菜单
```vue
参考毕设中的 LayoutContainer 页面
```

登录拦截：（在路由页面index.js中添加） 
```js
//登录拦截
router.beforeEach((to) => {
  const userStore = useUserStore()
  if (!userStore.token && to.path !== '/login') return '/login'
})
```

### 8.4 后台管理 - 内容页（用户信息+渲染退出）
在api->user.js，添加获取用户信息的接口
```js
//获取用户数据
export const userGetInfoService = () => request.get('my/userinfo')
```

在stores->user.js，添加保存用户信息到本地Pinia
```js
export const useUserStore = defineStore('userStoe', () => {
  //用户模块 token setToken removeToken
  ...此处省略之前代码

  //用户数据
  const user = ref({})
  //将用户信息存储本地Pinia
  const getUser = async () => {
    const res = await userGetInfoService()
    user.value = res.data.data
  }
  //清空本地Pinia的用户信息，赋值空对象即可
  const setUser = (obj) => {
    user.value = obj
  }

  return {
    ...此处省略之前代码
    user,
    getUser,
    setUser
  }
})
```

LayoutContainer 页面中，Pinia导入用户信息，和渲染退出等
```vue
参考毕设中的 LayoutContainer 页面
```

### 8.5 后台管理 - 内容页（菜单路由跳转）
更新router->index.js，将需要的页面编写好   
```js
参考毕设中的 router->index.js 页面
```
在 LayoutContainer 页面，主要完成以下才可以跳转
```vue
<script setup>
//2.下拉菜单切换
const router = useRouter()
const handleCommand = async (key) => {
  if (key == 'logout') {
    //退出操作
    await ElMessageBox.confirm('你真的要退出吗？', '温馨提示', {
      type: 'warning',
      confirmButtonText: '是的',
      cancelButtonText: '手滑了'
    })
    //清除本地的数据（token + user信息）
    // userStore.removeToken()
    // userStroe.setUser({})
    router.push('/login')
  } else {
    //跳转操作
    router.push(`/manage/user/${key}`)
  }
}
</script>

<!-- 侧边栏菜单跳转 -->
<el-menu
  :default-active="$route.path"
  :router="true"
></el-menu>
<el-menu-item index="/manage/post">
  <el-icon><Edit /></el-icon>
  <span>帖子编辑</span>
</el-menu-item>
<!-- 下拉菜单跳转 -->
<el-dropdown placement="bottom-end" @command="handleCommand">
  <span class="el-dropdown-link">
    <el-avatar :src="User" />
    <el-icon class="el-icon--right">
      <CaretBottom />
    </el-icon>
  </span>
  <template #dropdown>
    <el-dropdown-menu>
      <el-dropdown-item command="profile">基本资料</el-dropdown-item>
      <el-dropdown-item command="avatar">更换头像</el-dropdown-item>
      <el-dropdown-item command="password">重置密码</el-dropdown-item>
      <el-dropdown-item command="logout">退出登录</el-dropdown-item>
    </el-dropdown-menu>
  </template>
</el-dropdown>
```

### 8.6 后台管理 - 内容页（具体页面架子+模板抽取）
在components文件夹中，新建 PageContainer 页面（存放具体页面架子的模板）   
PageContainer 页面：
```vue
<script setup>
//父传子（因为模板是子）
defineProps({
  title: {
    required: true,
    type: String
  }
})
</script>

<template>
  <!-- 1. 卡片插槽 -->
  <el-card class="page-container">
    <!-- 1.1 头部 -->
    <template #header>
      <div class="header">
        <!-- 1.1.1 标题 -->
        <span>{{ title }}</span>
        <!-- 1.1.2 按钮 -->
        <div class="extra">
          <slot name="extra"></slot>
        </div>
      </div>
    </template>

    <!-- 2. 内容插槽 -->
    <slot></slot>
  </el-card>
</template>

<style lang="scss" scoped>
.page-container {
  min-height: 100%;
  box-sizing: border-box;
  .header {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
}
</style>
```
具体的页面（这里是ManagePost页面）：
```vue
<template>
  <page-container title="帖子编辑">
    <template #extra>
      <el-button type="primary">发布帖子</el-button>
    </template>
  </page-container>
</template>

```

### 8.7 后台管理 - 内容页（帖子分类 - 渲染+弹窗+添加编辑删除操作）

### 8.8 后台管理 - 内容页（美食帖子 - 中文化+下拉框提取+分页渲染+搜索重置按钮）
下拉框是父组件和子组件之前的传递，重点了解一下v-model和父子传参  