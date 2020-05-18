# 1.vue-cli

Vue CLI 是一个基于 Vue.js 进行快速开发的脚手架，原先定义好目录结构和基础代码，就像创建Maven项目时可以选择创建一个骨架项目，这个骨架项目就是一个脚手架，帮助我们更快速的开发。

主要功能如下：

- 统一的目录结构
- 本地调试
- 热部署
- 单元测试
- 集成打包上线

需要准备Node.js，官网链接：http://nodejs.cn/download/，选择对应版本下载安装即可，以我下载的windows为例，可以一路默认下一步，环境变量也能自动添加。

检查是否安装成功：

- cmd下 输入node - v
- cmd下 输入npm - v

能够正确打印版本号，说明安装成功

安装Node.js淘宝镜像加速器

```she
npm install cnpm -g
```

默认安装位置在`C:\Users\Administrator\AppData\Roaming\npm`目录下

安装vue-cli

```shell
cnpm install vue-cli -g
```

查看可以基于哪些模板创建vue程序，通常使用webpack

```shell
vue list
```

## 创建第一个vue-cli应用程序

创建一个vue项目，直接新建一空文件夹即可，在对应目录下，cmd输入，其中，myvue为我自定义的项目名，

```
vue init webpack myvue
```

回车确认后，会有一些配置项需要手动配置，现说明如下：

1.项目名称，如果不需要就直接回车。注：`此处项目名不能使用大写`。

```c
Project name 
```

2.项目描述，如果不需要就直接回车。

```c
Project description:
```

3.项目作者，默认计算机用户名

```c
Author (xxx)：
```

4.构建方式（暂且这么解释）

> 两个选择（上下箭头选择，回车即为选定）建议选择 : `Runtime + Compiler:recommended for most users`
> **这里推荐使用1选项，适合大多数用户的**

```c
vue build (Use arrow keys)
// 1. (译：运行+编译：被推荐给大多数用户)
> Runtime + Compiler:recommended for most users

// 2.(译：只运行大约6KB比较轻量的压缩文件，但只允许模板（或任何VUE特定HTML）。
//	VUE文件需要在其他地方呈现函数。翻译不精准，意思大概是选择该构建方式对文件大小有要求)
> Runtime-only:about 6KB lighter min+gzip,but templates (or any Vue-specific HTML) are ONLY 
allowed in .vue files-render functions are required elsewhere
12345678
```

5.安装vue的路由插件，需要就选y，否则就n

> 建议 : `Y`

```c
install vue-router?
```

6.是否使用ESLint检测你的代码？

> `ESLint` 是一个语法规则和代码风格的检查工具，可以用来保证写出语法正确、风格统一的代码。
> 建议选择 ‘`N`’ 因为选择 ‘`Y`’ 在做调试项目时,控制台会有很多 黄色警告 提示格式不规范,但其实并不影响项目

```c
Use ESLint to lint your code?
```

7.是否安装单元测试(暂不详细介绍)

> 建议 : `N`

```c
Setup unit tests? 
```

8.是否安装E2E测试框架NightWatch（E2E，也就是End To End，就是所谓的“用户真实场景”。）

> 建议 : `N`

```c
Setup e2e tests with Nightwatch(Y/n)?
```

9.项目创建后是否要为你运行“npm install”?这里选择包管理工具

> 选项有三个（上下箭头选择，回车即为选定）建议 : `yes use npm`

```c
Should we run 'npm install' for you after the project has been created?

// 使用npm
yes,use npm

// 使用yarn
yes,use yarn

// 自己操作
no,I will handle that myself
```

## 初始化并运行

```shell
cd myvue
#下载依赖
npm install
```

可能会有警告，根据提示输入修复即可

```shell
npm audit fix
```

启动项目

```shell
npm run dev
```

可以看到提示如下

![1589255040467](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589255040467.png)

访问http://localhost:8080/，启动成功

![1589255116742](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589255116742.png)

# 2.webpack

本质上，webpack 是一个现代 JavaScript 应用程序的**静态模块打包器(module bundler)**。当 webpack 处理应用程序时，它会递归地构建一个依赖关系图(dependency graph)，其中包含应用程序需要的每个模块，然后将所有这些模块打包成一个或多个 `bundle`。

## 模块化的演进

在应用之前，可以先看下这篇文章，对前端模块化的演化做了介绍，便于我们后面的理解：

[前端模块化演化阶段](https://www.jianshu.com/p/f2cf62c66493)

##   安装webpack

```shell
npm install webpack -g
npm install webpack-cli -g
```

测试安装成功

```shell
webpack -v
webpack-cli -v
```

## 配置

创建webpack.config.js配置文件

- entry：入口文件，指定webpack把哪个文件作为项目的入口
- output：输出，指定webpack把处理完成的文件放置到指定路径
- module：模块，用于处理各种类型的文件
- plugins：插件，如热更新等
- resolve：设置路径指向
- watch：监听，用于设置文件改动后直接打包

```javascript

module.exports = {
  entry: '',
  output: {
    path: '''',
    filename: 'foo.bundle.js'
  },
  module:{
      loaders:[
          {}
      ]
  },
  plugins:{},
  resolve:{},
  watch:true
};
```

## 使用webpack

1. 创建项目，即创建一空白文件夹即可，例如一名为`webpack-study`的文件夹

2. 创建一个名为modelus的目录

3. 在modelus目录下创建模块文件，如hello.js，用于编写JS模块相关代码

   ```javascript
   //暴露一个方法
   exports.sayHi = function () {
       document.write("<h1>学习Java</h1>");
   };
   ```

4. 在modelus目录下创建名为main.js的入口文件，用于打包时设置entry属性

   ```javascript
   var hello = require("./hello");
   hello.sayHi();
   ```

5. 在项目目录下创建名为webpack.config.js配置文件，使用webpack命令打包

   ```javascript
   module.exports={
       entry:'./modules/main.js',
       output:{
           filename:"./js/boundle.js"
       }
   }
   ```

6. 控制台输入命令`webpack`打包，可以看到生成对应的`boundle.js`文件

7. 此时，新建`index.html`文件，引入刚生成的`boundle.js`文件

   ```html
   <script src="dist/js/boundle.js"></script>
   ```

8. 浏览器打开`index.html`文件，显示正确

![1589272176691](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589272176691.png)

项目整体目录如下所示

![1589272211489](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589272211489.png)

# 3.vue-router路由

## 简介

以下介绍摘抄自官网：https://router.vuejs.org/zh/

Vue Router 是 [Vue.js](http://cn.vuejs.org/) 官方的路由管理器。它和 Vue.js 的核心深度集成，让构建单页面应用变得易如反掌。包含的功能有：

- 嵌套的路由/视图表
- 模块化的、基于组件的路由配置
- 路由参数、查询、通配符
- 基于 Vue.js 过渡系统的视图过渡效果
- 细粒度的导航控制
- 带有自动激活的 CSS class 的链接
- HTML5 历史模式或 hash 模式，在 IE9 中自动降级
- 自定义的滚动条行为

## 应用

以在vue-cli章节中创建的myvue项目为例。

### 删除不需要的默认内容

先删除一些不需要的默认项，比如src目录下如下红框内的文件

![1589272844911](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589272844911.png)

然后，如下所示，在App.vue中删除与上述文件相关的内容

![1589272877242](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589272877242.png)

### 安装vue-router

老规矩，利用npm/cnpm进行安装

```shell
npm install vue-router --save-dev
```

如果要在一个模块化工程中使用，通过Vue.use()明确安装即可

```vue
import VueRouter from 'vue-router'
import Vue from 'vue'

Vue.use(VueRouter);
```



### 编写测试用例

- 在`components`目录下存放我们自己编写的组件`Main.vue`，`Content.vue`,视为首页和内容页，包含的信息也很简单

Main.vue

```vue
<template>
  <h1>首页</h1>
</template>

<script>
    export default {
        name: "main"
    }
</script>

<style scoped>

</style>
```

Content.vue

```vue
<template>
  <h1>内容页</h1>
</template>

<script>
    export default {
        name: "Content"
    }
</script>

<style scoped>

</style>
```

- 新建文件夹`router`，专门存放路由

  ```javascript
  import Vue from 'vue'
  import VueRouter from 'vue-router'
  //导入上面定义的组件
  import Content from '../components/Content'
  import Main from '../components/Main'
  
  Vue.use(VueRouter);
  
  //配置导出路由
  export default new VueRouter({
    routes:[
      {
          //路由路径
        path:'/content',
          //路由名称
        name:'content',
          //跳转到对应组件
        component:Content
      },
      {
        path:'/main',
        name:'main',
        component:Main
      }
    ]
  });
  ```

- 在`main.js`中配置路由

  ```javascript
  import Vue from 'vue'
  import App from './App'
  //也可以不指定index，会自动扫描
  import router from './router/index'
  //关闭生产模式下的提示
  Vue.config.productionTip = false
  
  new Vue({
    el: '#app',
    //配置路由
    router,
    components: { App },
    template: '<App/>'
  })
  ```

- 在`App.vue`中使用路由，如下所示，添加`router-link`，`router-view`标签，

  ```vue
  <template>
    <div id="app">
      <router-link to="/main">首页</router-link>
      <router-link to="/content">内容页</router-link>
      <!-- 路由出口 -->
      <!-- 路由匹配到的组件将渲染在这里 -->
      <router-view></router-view>
    </div>
  </template>
  
  <script>
  
  export default {
    name: 'App',
    components: {
    }
  }
  </script>
  
  <style>
  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
  }
  </style>
  
  ```

  最终效果如下

  ![1589277045040](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589277045040.png)

# 4.Vue+elementUI快速入门

elemeng官网地址：https://element.eleme.cn/#/zh-CN/component/installation

新建项目vue-elementui

```shell
vue init webpack vue-elementui
```

为了流程更清晰，配置一路选择了no选项，手动安装依赖

```shell
cd vue-elementui
#安装vue-router
npm install vue-router --save-dev
#安装elemengt-ui
npm i element-ui -S
#安装依赖
npm install
#安装SASS加载器,速度影响使用了阿里的cnpm
cnpm install sass-loader node-sass --save-dev
#启动测试
npm run dev
```

对npm命令做进一步解释

- `npm install moduleName`：安装模块到项目目录下
- `npm install -g moduleName`:-g 的意思是将模块安装到全局，具体安装位置，看npm config perfix的配置
- `npm install --save moduleName`:-- save的意思是将模块安装到项目目录下，并在paskage文件的dependencies节点写入依赖，-S为缩写
- `npm install --save-dev  moduleName`：--save-dev意思是将模块安装到项目目录下，并在paskage文件的devDependencies节点写入依赖，-D为缩写

老规矩，利用IDEA打开`vue-elementui`文件，删除默认的HelloWorld相关内容。我们新建文件夹`router`，`views`，用来存放路由的组件和视图的组件，原来的`components`继续存放功能性的组件

## 创建首页视图

在 `views` 目录下创建一个名为 `Main.vue` 的视图组件；该组件在当前章节无任何作用，主要用于登录后展示登录成功的跳转效果；

```vue
<template>
    <div>
      首页
    </div>
</template>
<script>
    export default {
        name: "Main"
    }
</script>
<style scoped>
</style>
```

## 创建登录页视图

在 `views` 目录下创建一个名为 `Login.vue` 的视图组件，其中 `el-*` 的元素为 ElementUI 组件；

```vue
<template>
  <div>
    <el-form ref="loginForm" :model="form" :rules="rules" label-width="80px" class="login-box">
      <h3 class="login-title">欢迎登录</h3>
      <el-form-item label="账号" prop="username">
        <el-input type="text" placeholder="请输入账号" v-model="form.username"/>
      </el-form-item>
      <el-form-item label="密码" prop="password">
        <el-input type="password" placeholder="请输入密码" v-model="form.password"/>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" v-on:click="onSubmit('loginForm')">登录</el-button>
      </el-form-item>
    </el-form>

    <el-dialog
      title="温馨提示"
      :visible.sync="dialogVisible"
      width="30%"
      :before-close="handleClose">
      <span>请输入账号和密码</span>
      <span slot="footer" class="dialog-footer">
        <el-button type="primary" @click="dialogVisible = false">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
  export default {
    name: "Login",
    data() {
      return {
        form: {
          username: '',
          password: ''
        },
        // 表单验证，需要在 el-form-item 元素中增加 prop 属性
        rules: {
          username: [
            {required: true, message: '账号不可为空', trigger: 'blur'}
          ],
          password: [
            {required: true, message: '密码不可为空', trigger: 'blur'}
          ]
        },
        // 对话框显示和隐藏
        dialogVisible: false
      }
    },
    methods: {
      onSubmit(formName) {
        // 为表单绑定验证功能
        this.$refs[formName].validate((valid) => {
          if (valid) {
            // 使用 vue-router 路由到指定页面，该方式称之为编程式导航
            this.$router.push("/main");
          } else {
            this.dialogVisible = true;
            return false;
          }
        });
      }
    }
  }
</script>

<style lang="scss" scoped>
  .login-box {
    border: 1px solid #DCDFE6;
    width: 350px;
    margin: 180px auto;
    padding: 35px 35px 15px 35px;
    border-radius: 5px;
    -webkit-border-radius: 5px;
    -moz-border-radius: 5px;
    box-shadow: 0 0 25px #909399;
  }

  .login-title {
    text-align: center;
    margin: 0 auto 40px auto;
    color: #303133;
  }
</style>
```

## 创建路由

在 `router` 目录下创建一个名为 `index.js` 的 vue-router 路由配置文件

```javascript
import Vue from 'vue'
import Router from 'vue-router'

import Login from "../views/Login"
import Main from '../views/Main'

Vue.use(Router);

export default new Router({
  routes: [
    {
      // 登录页
      path: '/login',
      name: 'Login',
      component: Login
    },
    {
      // 首页
      path: '/main',
      name: 'Main',
      component: Main
    }
  ]
});
```

## 配置路由

修改 `main.js` 入口代码

```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import router from './router'

// 导入 ElementUI
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'

import App from './App'

// 安装路由
Vue.use(VueRouter);

// 安装 ElementUI
Vue.use(ElementUI);

new Vue({
  el: '#app',
  // 启用路由
  router,
  // 启用 ElementUI
  render: h => h(App)
});
```

修改 `App.vue` 组件代码

```vue
<template>
  <div id="app">
    <router-view/>
  </div>
</template>

<script>
  export default {
    name: 'App',
  }
</script>
```

输入`npm run dev`启动。需要说明的是，有可能你会发现启动报错

```
Module build failed: TypeError: this.getResolve is not a function
    at Object.loader (D:\Program Files\ideaProjects\vueProject\vue-elementui\node_modules\_sass-loader@8.0.2@sass-loader\dist\index.js:52:26)
```

原因就是默认的`sass-loader`版本过高，在`package.json`文件中降低版本即可，比如7.3.1版本，然后重新安装，重新启动()

```shell
npm install #如果不成功可以换用cnpm install
npm run dev
```

## 效果演示

在浏览器打开 http://localhost:8080/#/login 你会看到如下效果

![1589285078307](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589285078307.png)



# 5.嵌套路由

嵌套路由又称子路由，在实际应用中，通常由多层嵌套的组件组合而成，同样地，URL中各段动态路径也按某种结构对应嵌套的各种组件，如下所示，请求不同的URL，组件对应变化

![1589285270585](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589285270585.png)

## 创建子视图

继续修改上面的项目，我们在views文件夹下新建user文件夹，并在user文件夹下添加`List.vue`，`Profile.vue`,内容很简单，只写一个标题栏区分即可

List.vue

```vue
<template>
    <h1>用户列表</h1>
</template>

<script>
    export default {
        name: "UserList"
    }
</script>

<style scoped>

</style>
```

Profile.vue

```vue
<template>
    <h1>个人信息</h1>
</template>

<script>
    export default {
        name: "UserProfile"
    }
</script>

<style scoped>

</style>
```

## 配置路由

修改router文件夹下的`index.js`文件

```javascript
import Vue from 'vue'
import Router from 'vue-router'

import Login from "../views/Login"
import Main from '../views/Main'
import UserList from "../views/user/List"
import UserProfile from "../views/user/Profile"

Vue.use(Router);

export default new Router({
  routes: [
    {
      // 登录页
      path: '/login',
      name: 'Login',
      component: Login
    },
    {
      // 首页
      path: '/main',
      name: 'Main',
      component: Main,
      //嵌套路由
      children:[
        {path:'/user/profile',component:UserProfile},
        {path:'/user/list',component:UserList}
      ]
    }
  ]
});

```

## 修改主视图

根据ElementUI的[Container布局容器示例](https://element.eleme.cn/#/zh-CN/component/container)修改`Main.vue`

```vue
<template>
  <div>
    <el-container>
      <el-aside width="200px">
        <el-menu :default-openeds="['1']">
          <el-submenu index="1">
            <template slot="title"><i class="el-icon-caret-right"></i>用户管理</template>
            <el-menu-item-group>
              <el-menu-item index="1-1">
                <router-link to="/user/profile">个人信息</router-link>
              </el-menu-item>
              <el-menu-item index="1-2">
                <router-link to="/user/list">用户列表</router-link>
              </el-menu-item>
            </el-menu-item-group>
          </el-submenu>
          <el-submenu index="2">
            <template slot="title"><i class="el-icon-caret-right"></i>内容管理</template>
            <el-menu-item-group>
              <el-menu-item index="2-1">分类管理</el-menu-item>
              <el-menu-item index="2-2">内容列表</el-menu-item>
            </el-menu-item-group>
          </el-submenu>
        </el-menu>
      </el-aside>

      <el-container>
        <el-header style="text-align: right; font-size: 12px">
          <el-dropdown>
            <i class="el-icon-setting" style="margin-right: 15px"></i>
            <el-dropdown-menu slot="dropdown">
              <el-dropdown-item>个人信息</el-dropdown-item>
              <el-dropdown-item>退出登录</el-dropdown-item>
            </el-dropdown-menu>
          </el-dropdown>
        </el-header>

        <el-main>
          <router-view />
        </el-main>
      </el-container>
    </el-container>
  </div>
</template>

<script>
    export default {
        name: "Main"
    }
</script>
<style scoped lang="scss">
  .el-header {
    background-color: #2acaff;
    color: #333;
    line-height: 60px;
  }

  .el-aside {
    color: #333;
  }
</style>

```

## 效果演示

![1589288395355](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589288395355.png)

# 6.参数传递与重定向

## 参数传递

上一节内容中，我们解决了嵌套路由的问题，但理论上，点击个人信息应该显示每个用户单独的信息才对，而不是上一节所示的统一显示，这就涉及到了参数传递的问题。

<router-link>标签自己就可以指定参数，同时还需利用`v-bind`绑定，可以看到，我们用路劲名称代替了实际路径

```
<router-link v-bind:to="{name:'UserProfile',param:{id:1}}"></router-link>
```

同理，在路由的index.js里也要设置接收参数

```js
export default new Router({
  routes: [
    {
      // 登录页
      path: '/login',
      name: 'Login',
      component: Login
    },
    {
      // 首页
      path: '/main',
      name: 'Main',
      component: Main,
      //嵌套路由
      children:[
        {path:'/user/profile/:id',name:'UserProfile',component:UserProfile},
        {path:'/user/list',component:UserList}
      ]
    }
  ]
});
```

最后，在`Profile.vue`文件中引用`$route`

```vue
<template>
  <div>
    <h1>个人信息</h1>
    {{$route.params.id}}
  </div>
</template>

<script>
    export default {
        name: "UserProfile"
    }
</script>

<style scoped>

</style>

```

需要注意，元素不能直接写在根节点下，比如下面这样就会报错

```vue
<template>
    <h1>个人信息</h1>
    {{$route.params.id}}
</template>
```

官方更建议使用 **props 解耦**的方式来进行传参，

修改index.js，添加`props: true`

```js
export default new Router({
  routes: [
    {
      // 登录页
      path: '/login',
      name: 'Login',
      component: Login
    },
    {
      // 首页
      path: '/main',
      name: 'Main',
      component: Main,
      //嵌套路由
      children:[
        {path:'/user/profile/:id',component:UserProfile,props:true},
        {path:'/user/list',component:UserList}
      ]
    }
  ]
});
```

在`Profile.vue`中

```vue
<template>
  <div>
    <h1>个人信息</h1>
      {{id}}
  </div>
</template>

<script>
    export default {
        props:['id'],
        name: "UserProfile"
    }
</script>

<style scoped>

</style>

```

## 重定向

利用路由的redirect属性即可，如下所示

```json
export default new Router({
  routes: [
    {
      // 登录页
      path: '/login',
      name: 'Login',
      component: Login
    },
    {
      // 首页
      path: '/main',
      name: 'Main',
      component: Main,
      //嵌套路由
      children:[
        {path:'/user/profile/:id',name:'UserProfile',component:UserProfile},
        {path:'/user/list',component:UserList},
        {path:'/goHome',redirect:'/main'}
      ]
    }
  ]
});
```





