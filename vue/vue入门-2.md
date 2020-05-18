# 1. 组件基础

组件是可复用的 Vue 实例，且带有一个名字，说白了就是一组可以重复使用的模板。

## 基本示例

官网例子如下：

```javascript
// 定义一个名为 button-counter 的新组件
Vue.component('button-counter', {
  data: function () {
    return {
      count: 0
    }
  },
  template: '<button v-on:click="count++">You clicked me {{ count }} times.</button>'
})
```

然后就可以把这个组件当做自定义元素来使用

```html
<div id="app">
  <button-counter></button-counter>
</div>
<script>
    var vm = new Vue({
        el:'#app'
    });
</script>
```

可以看到，利用`Vue.component`注册组件，`button-counter`就是自定义的组件名字，`data`**选项必须是一个函数**，每个实例可以维护一份被返回对象的独立的拷贝，现举例说明：

我们复用三次该组件

```html
<div id="components-demo">
  <button-counter></button-counter>
  <button-counter></button-counter>
  <button-counter></button-counter>
</div>
<script>
    var vm = new Vue({
        el:'#components-demo'
    });
</script>
```

可以看到，点击按钮时，每个组件都会各自独立维护它的 `count`。因为你每用一次组件，就会有一个它的新**实例**被创建。

![1589197680780](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589197680780.png)

## 通过props向子组件传递值

我们还可以把参数传递给组件，此时就需要`props`属性，例子如下：

```html
<div id="app">
    <test v-for="item in items" v-bind:prop="item"></test>
</div>
<script>
    Vue.component("test",{
        props:['prop'],
        template:'<li>{{item}}</li>'
    });
    var vm = new Vue({
        el:'#app',
        data:{
            items:["Java","C#","Go"]
        }
    });
</script>
```

`v-for="item in items"`:遍历vue实例中定义的`items`数组，并创建同等数量的组件

`v-bind:prop="item"`：将上一步`v-for`遍历的`item`项绑定到组件`props`定义的名为`prop`的属性上。

# 2. 异步通信 Axios

由于Vue.js是一个视图层框架且严格遵守SOC（关注度分离原则），所以Vue.js并不支持AJAX的通信功能，作者尤雨溪老师曾开发过一名为`vue-resource`的插件，但在2.0版本之后停止维护并推荐使用`Axios`框架。

Axios是一个开源的可以用在浏览器端和NodeJS的异步通信框架，主要作用就是实现AJAX异步通信。



## vue生命周期

每个 Vue 实例在被创建时都要经过一系列的初始化过程——例如，需要设置数据监听、编译模板、将实例挂载到 DOM 并在数据变化时更新 DOM 等。同时在这个过程中也会运行一些叫做**生命周期钩子**的函数，这给了用户在不同阶段添加自己的代码的机会，如下所示，比如`created`钩子可以用来在一个实例被创建之后执行代码：

![](https://s1.ax1x.com/2020/05/11/YJHyaq.md.png)

我们以`mounted`为例，它是编辑好的HTML挂载到页面完成后执行的事件钩子，此钩子函数一般会做一些AJAX请求获取数据进行初始化，**`mounted`在整个实例中仅执行一次**。

我们先以获取本地的json文件为例，创建data.json文件

```json
{
  "name": "张三",
  "age": 25,
  "isMan": true,
   "url":"https://www.baidu.com/",
  "address":{
    "street": "火星街",
    "city": "成都"
  }
}
```

通过函数`mounted()`获取json数据，通过`data()`函数将数据绑定

```html
<div id="app">
    <div>{{info.name}}</div>
    <div>{{info.address.city}}</div>
    <a v-bind:href="info.url">链接</a>
</div>
<script type="text/javascript">
    var vm = new Vue({
        el:'#app',
        data(){
            return {
                info: null
            }
        },
        mounted(){
            axios.get('../static/json/data.json').then(response=>(this.info=response.data))
        }
    });
</script>
```

同理，当然也可以通过访问后端提供的接口来获取数据

```javascript
axios.get('/user', {
    params: {
      ID: 12345
    }
  })
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });
```

post方法一样

```javascript
axios.post('/user', {
    firstName: 'Fred',        // 参数 firstName
    lastName: 'Flintstone'    // 参数 lastName
  })
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });
```

还能执行多个并发请求

```javascript
function getUserAccount() {
  return axios.get('/user/12345');
}
 
function getUserPermissions() {
  return axios.get('/user/12345/permissions');
}
axios.all([getUserAccount(), getUserPermissions()])
  .then(axios.spread(function (acct, perms) {
    // 两个请求现在都执行完成
  }));
```

Axios的更多用法可以参考：https://www.kancloud.cn/yunye/axios/234845。

# 3. 插槽

在`Vue.js`中，将 `<slot>` 元素作为承载分发内容的出口，称之为插槽，可以应用在组合组件的场景中。

比如我们要创建如下一个代办事项组件（todo），该组件有标题和内容组成，但这三个组件又是相互独立的，该如何操作呢？

```html
<div>
    <div>代办事项</div>
    <ul>
        <li>学习Java</li>
        <li>学习Linux</li>
    </ul>
</div>
```

最终代码如下：

```html
<div id="app">
    <todo>
        <todo-title slot="todo-title" v-bind:name="todoTitle"></todo-title>
        <todo-items slot="todo-items" v-for="item in todoItems" v-bind:name="item"></todo-items>
    </todo>
</div>
<script type="text/javascript">
   Vue.component('todo',{
       template:
           '<div>\
                <slot name="todo-title"></slot>\
            <ul>\
                <slot name="todo-items"></slot>\
            </ul>\
            </div>'
   });
   Vue.component("todo-title",{
       props:['name'],
       template:'<div>{{name}}</div>'
   })
   Vue.component("todo-items",{
       props:['name'],
       template:'<li>{{name}}</li>'
   })
   new Vue({
       el:'#app',
       data:{
           todoTitle:'代办事项',
           todoItems:['学习Java','学习Linux']
       }
   });
</script>

```

分析上面的代码，可以看到，我们首先定义了三个组件`todo`，`todo-title`，`todo-items`，可以将`todo`视为我们的主组件，其他两个是从组件。并在主组件`todo`的`template`定义中制定了插槽，通过插槽的`name`上属性匹配相应的组件。同时，为了接受传递的参数值，在从组件中指定了`props`属性。

```javascript
   Vue.component('todo',{
       template:
           '<div>\
                <slot name="todo-title"></slot>\
            <ul>\
                <slot name="todo-items"></slot>\
            </ul>\
            </div>'
   });
   Vue.component("todo-title",{
       props:['name'],
       template:'<div>{{name}}</div>'
   })
   Vue.component("todo-items",{
       props:['name'],
       template:'<li>{{name}}</li>'
   })
```

之后，定义了如下的组件组合方式，通过`slot`属性匹配对应的组件。

```html
<div id="app">
    <todo>
        <todo-title slot="todo-title" v-bind:name="todoTitle"></todo-title>
        <todo-items slot="todo-items" v-for="item in todoItems" v-bind:name="item"></todo-items>
    </todo>
</div>
```

对于`todo-title`组件，利用`v-bind`将我们在对应组件中`props`指定的名为`name`的参数与Vue对象的`data`属性的`todoTitle`绑定。

对于`todo-items`组件，首先通过`v-for`遍历绑定的Vue对象的`data`属性的`todoItems`，然后利用`v-bind`将遍历获得的`item`与组件`props`指定的名为`name`参数绑定。

# 4.自定义事件

此时，我已经通过插槽构建了我们需要的界面，考虑一个问题，如果我们现在需要添加删除功能该怎么做呢？

首先，组件内部自己可以定义方法，我们修改上文例子中的`todo-title`插槽，为他添加自定义方法

```javascript
  Vue.component("todo-items",{
        props:['name'],
        template:'<li>{{name}}<button v-on:click="remove">删除</button></li>',
        methods:{
            remove:function () {
                alert("删除信息");
            }
        }
    })
```

测试没有问题，但这不符合我们想要的效果，我们期望的删除`Vue`对象中的`data`属性的数据，那么我们在Vue对象中定义删除方法，然后在组件中绑定，可以吗？如下所示，我们在组件`todo-items`中绑定Vue对象定义的方法`removeItem`

```javascript
      Vue.component("todo-items",{
        props:['name'],
        template:'<li>{{name}}<button v-on:click="removeItem">删除</button></li>',
        methods:{
            remove:function () {
                alert("删除信息");
            }
        }
    })
    
    new Vue({
        el:'#app',
        data:{
            todoTitle:'代办事项',
            todoItems:['学习Java','学习Linux'],
            methods:{
                removeItem:function (index) {
                    this.todoItems.splice(index,1);//从当前下标删除一个元素
                }
            }
        }
    });
```

你会发现，浏览器控制台报错，该方法未定义，可见是行不通的。

![1589249074349](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589249074349.png)

Vue为我们提供了自定义事件的功能就很好的为我们解决了这个问题，使用**this.$emit('自定义事件名'，参数)**，操作过程如下：

首先，通过上面的例子，我们知道，在组件内是不能直接绑定Vue对象内的方法的，但是用已经创建的组件标签是可以的，如下所示，我们利用`v-on:`把Vue对象内的`removeItem`方法绑定，其中，`remove-diy`是我们自定义的事件名称。

**注意：事件名请采用小写，如果采用驼峰命名，浏览器会把驼峰命名中的大写字母自动转化成小写，导致匹配不到事件**

```html
<todo-items slot="todo-items" v-for="(index，item) in todoItems"
v-bind:name="item" v-bind:index="index" v-on:remove-diy="removeItem(index)"></todo-items>
```

然后，在组件`todo-items`内就可以利用`this.$emit('自定义事件名'，参数)`进行操作

```javascript
    Vue.component("todo-items",{
        props:['name','index'],
        template:'<li>{{name}}<button v-on:click="remove">删除</button></li>',
        methods:{
            remove:function (index) {
               this.$emit('remove-diy',index)
            }
        }
    })
```

完整代码如下

```html
<div id="app">
    <todo>
        <todo-title slot="todo-title" v-bind:name="todoTitle"></todo-title>
        <todo-items slot="todo-items" v-for="(item,index) in todoItems"
                    v-bind:name="item" v-bind:index="index" 
                    v-on:remove-diy="removeItem(index)"></todo-items>
    </todo>
</div>
<script type="text/javascript">
    Vue.component('todo',{
        template:
            '<div>\
                 <slot name="todo-title"></slot>\
             <ul>\
                 <slot name="todo-items"></slot>\
             </ul>\
             </div>'
    });
    Vue.component("todo-title",{
        props:['name'],
        template:'<div>{{name}}</div>'
    });
    Vue.component("todo-items",{
        props:['name','index'],
        template:'<li>{{name}}<button v-on:click="remove">删除</button></li>',
        methods:{
            remove:function (index) {
               this.$emit('remove-diy',index);
            }
        }
    });
    var vm = new Vue({
        el:'#app',
        data:{
            todoTitle:'代办事项',
            todoItems:['学习Java','学习Linux'],
        },
        methods:{
            removeItem:function (index) {
                this.todoItems.splice(index,1);//从当前下标删除一个元素
            }
        }
    });
</script>
```







