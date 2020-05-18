由于工作需要，需要了解下Vue的内容，因此根据官方文档总结了这个系列的快速上手笔记，大家也可以直接查看官方文档，写的也很详细：https://cn.vuejs.org/v2/guide/

# 1.概述

前端主要有如下三要素构成：

- HTML（结构）

  超文本标记语言（HyperText Markup Language），决定网页的结构和内容

- CSS（表现）

  层叠样式表（Cascading Style Sheets），决定网页的表现形式

- JavaScript（行为）

  弱类型脚本语言，源代码无需编译，由浏览器解释执行，用于控制网页的行为

而Vue (读音 /vjuː/，类似于 view) 就是一套用于构建用户界面的**渐进式JavaScript**框架。与其它大型框架不同的是，Vue 被设计为可以自底向上逐层应用。Vue 的核心库只关注视图层，不仅易于上手，还便于与第三方库或既有项目整合。

Vue (读音 /vjuː/，类似于 **view**) 是一套用于构建用户界面的**渐进式JavaScript框架**。与其它大型框架不同的是，Vue 被设计为可以自底向上逐层应用。Vue 的核心库只关注视图层，不仅易于上手，还便于与第三方库或既有项目整合。

Vue是MVVM架构的实现者，MVVM架构如下所示：

![1589168527875](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589168527875.png)

主要是ViewModel层，链接View和Model层的中间件，**ViewModel能够观察到数据变化，并对视图的内容进行更新，也能够监听到视图的变化，并能够通知数据发生改变**，Vue就是MVVM架构中ViewModel的实现。



# 2.第一个Vue程序

## 安装与部署

根据[官方文档](https://cn.vuejs.org/v2/guide/installation.html#Vue-Devtools)说明，新手上路直接利用<script>引入即可，直接下载需要的版本并用 <script> 标签引入，Vue 会被注册为一个全局变量。

也可以利用CDN的方式去引用，如下，可以这样使用最新版本：

```
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
```

新建一个html文件，引入vue.js，例子如下:

```html
<div id="app">
    {{message}}
</div>

<script>
    var vm = new Vue({
        el:'#app',
        data:{
            message:'Hello Vue'
        }
    });
</script>
```

可以看到，我们新建Vue对象，通过`el`属性绑定id为`app`的div，并通过属性data绑定数据`message`，div可以直接获取到`message`信息。

为了直观的体验Vue的数据绑定功能，可以通过浏览器测试一番：

浏览器打开创建的html文件，显示我们默认设置的`Hello Vue`

![1589172238892](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589172238892.png)

进入开发者工具，在控制台输入vm.message ='Hello World'，可以看到浏览器显示的内容自动更新

![1589172368980](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589172368980.png)



# 3.基本语法

## Class 与 Style 绑定 v-bind

在之前的第一个程序中，我们使用{{message}}来获取Vue对象中的属性，还可以使用`v-bind`来绑定元素特性

```html
<div id="app">
    <span v-bind:title="message">
        鼠标悬停几秒查看动态绑定的提示信息
    </span>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            message:'Hello Vue'
        }
    });
</script>
```

在Vue中，像v-bind这种指令带有前缀v-的，都被称为指令，表示他们是Vue提供的特殊特性，它们会在渲染的DOM上应用特殊的响应式行为。在这个例子中，该指令的意思就是：将这个元素节点的title属性与Vue实例的message属性保持一致。

## 条件渲染 v-if，v-else

判断语句，不用多说，直接看例子

```html
<div id="app">
    <h1 v-if="ok">Yes</h1>
    <h1 v-else>No</h1>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            ok:true
        }
    });
</script>
```

```html
<div id="app">
    <h1 v-if="type==='A'">A</h1>
    <h1 v-else-if="type==='B'">B</h1>
	<h1 v-else>C</h1>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            type:'A'
        }
    });
</script>
```

## 列表渲染 v-for

循环一样，直接看例子就明白了

```html
<div id="app">
    <li v-for="item in items">
        {{item.message}}
    </li>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            items:[
                {message:'吃苹果'},
                {message:'吃香蕉'}
            ]
        }
    });
</script>
```

## 事件处理 v-on

用 `v-on` 指令监听 DOM 事件，并在触发时运行一些 JavaScript 代码。

```html
<div id="app">
    <button v-on:click="sayHi()">单击</button>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            message:'方法运行'
        },
        methods:{ //方法必须定义在Vue的methods对象中
            sayHi:function (event) {
                alert(this.message);
                // `event` 是原生 DOM 事件
                if (event) {
                    alert(event.target.tagName)
                }
            }
        }
    });
</script>
```

## 双向绑定

可以用 `v-model` 指令在表单 `<input>`、`<textarea>` 及 `<select>` 元素上创建双向数据绑定。

需要注意，`v-model` 会忽略所有表单元素的 `value`、`checked`、`selected` attribute 的初始值而总是将 Vue 实例的数据作为数据来源。你应该通过 JavaScript 在组件的 `data` 选项中声明初始值。

### 文本

```html
<div id="app">
    <input v-model="message" placeholder="edit me">
    <p>Message is: {{ message }}</p>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            message:''
        }
    });
</script>
```

### 复选框

```html
<div id="app">
    <input type="checkbox" id="checkbox" v-model="checked">
    <label for="checkbox">{{ checked }}</label>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            checked:''
        }
    });
</script>
```

### 单选按钮

```html
<div id="app">
  <input type="radio" id="one" value="One" v-model="picked">
  <label for="one">One</label>
  <br>
  <input type="radio" id="two" value="Two" v-model="picked">
  <label for="two">Two</label>
  <br>
  <span>Picked: {{ picked }}</span>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            picked:''
        }
    });
</script>
```

### 下拉框

```html
<div id="app">
  <select v-model="selected">
    <option disabled value="">请选择</option>
    <option>A</option>
    <option>B</option>
    <option>C</option>
  </select>
  <span>Selected: {{ selected }}</span>
</div>
<script>
    var vm = new Vue({
        el:'#app',
        data:{
            selected:''
        }
    });
</script>
```

官网这里还有额外提示如下：

```
如果 v-model 表达式的初始值未能匹配任何选项，<select> 元素将被渲染为“未选中”状态。在 iOS 中，这会使用户无法选择第一个选项。因为这样的情况下，iOS 不会触发 change 事件。因此，更推荐像上面这样提供一个值为空的禁用选项。
```

### 修饰符:

`.lazy`

在默认情况下，`v-model` 在每次 `input` 事件触发后将输入框的值与数据进行同步 (除了[上述](https://cn.vuejs.org/v2/guide/forms.html#vmodel-ime-tip)输入法组合文字时)。你可以添加 `lazy` 修饰符，从而转为在 `change` 事件_之后_进行同步：

```html
<!-- 在“change”时而非“input”时更新 -->
<input v-model.lazy="msg">
```

`.number`

如果想自动将用户的输入值转为数值类型，可以给 `v-model` 添加 `number` 修饰符：

```html
<input v-model.number="age" type="number">
```

`.trim`

如果要自动过滤用户输入的首尾空白字符，可以给 `v-model` 添加 `trim` 修饰符：

```html
<input v-model.trim="msg">
```

## 计算属性

简单说，计算属性就是一个能将计算结果==**缓存**==起来的属性。对于任何的复杂逻辑，都建议使用**计算属性**。

Vue利用`computed`申明计算属性：

```html
<div id="app">
    <p>调用当前计算时间的方法：{{currtenTime1()}}</p>
    <p>调用当前计算时间的属性：{{currtenTime2}}</p>
</div>
<script type="text/javascript">

    var vm = new Vue({
        el:'#app',
        data:{
            "message":"Hello Vue"
        },
        methods:{
            currtenTime1:function(){
                return Date.now();
            }
        },
        computed:{
            currtenTime2:function(){
                return Date.now();
            }
        }
    });
</script>
```

观察这个例子，可以看到，我们不仅通过计算属性来获取当前时间戳，还利用了方法来获取时间戳

```html
<div id="app">
    <p>调用当前计算时间的方法：{{currtenTime1()}}</p>
    <p>调用当前计算时间的属性：{{currtenTime2}}</p>
</div>
```

我们打开浏览器的控制台，很明显能看到，计算属性的值被缓存了。

![1589203538886](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589203538886.png)





那么，什么情况下计算属性的值才能更新呢？**计算属性是基于它们的响应式依赖进行缓存的。只在相关响应式依赖发生改变时它们才会重新求值。**

我们修改计算属性，添加一个相关依赖

```html
<script type="text/javascript">
    var vm = new Vue({
        el:'#app',
        data:{
            "message":"Hello Vue"
        },
        methods:{
            currtenTime1:function(){
                return Date.now();
            }
        },
        computed:{
            currtenTime2:function(){
                this.message;
                return Date.now();
            }
        }
    });
</script>
```

再次打开浏览器的控制台，可以看到，在输入`vm.message='test'`之后，计算属性的值更新了，获取时间戳的方法再次执行。

![1589203750855](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589203750855.png)



