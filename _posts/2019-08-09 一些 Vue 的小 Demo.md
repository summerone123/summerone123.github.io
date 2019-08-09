---
layout:     post   				    # 使用的布局（不需要改）
title:      Some Vue Demos # 标题 
subtitle:   数据可视化    #副标题
date:       2019-09-21			# 时间
author:     summer				# 作者
header-img: img/post-bg-os-metro.jpg  	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								   #标签
    - Vue 学习
---

# 利用搭好的框架实践几个小实践

## 一个简单的不能再简单的vue例子
我们在已经搭建好的vue项目中，实现一个从项目已有的Hello World! 跳转至我们自己创建的Hello Vue组件页面的例子。

### 首先，在已经搭建好的环境的components下新建一个vue文件，作为我们自己的vue组件。 
![-w810](/img/blog_img/15650123700773.jpg)
### 然后在HelloVue.vue文件中添加以下代码，vue文件的格式分为三个模块，如下图所示，首先时template模板，然后是script标签及代码，最后是style样式。

```javascript
<template>
  <div id="vue">Hello Vue.js! {{ message }}</div>
</template>

<script type="text/javascript">
  export default { //这里需要将模块引出，可在其他地方使用
    name: "HelloVue",
    data (){ //注意：data即使不需要传数据，也必须return,否则会报错
      return {
        message: "啦啦啦啦啦"
      }
    }
  }
</script>

<style type="text/css">
  #vue{
    color: green;
    font-size: 28px;
  }
</style>

```

### 在项目搭建时生成的HelloWorld.vue文件中的template中添加一个链接，用于跳转至我们自己的组件内容。 
关于 router-link 的使用，请参考 vue-router文档。

```javascript
<h1>
 <router-link to="day01">跳转至HelloVue</router-link>
</h1>
```

### 接着，我们修改项目中的router.js文件，这是一个vue-router的简单应用。对于路由，我们一般会想到宽带安装时我们使用的路由器，这里的路由主要是为了定义页面之前的跳转。在routerjs文件中添加以下代码：

![-w1051](/img/blog_img/15650132329982.jpg)

### 结果展示
![-w775](/img/blog_img/15650132648748.jpg)
![-w518](/img/blog_img/15650132749640.jpg)
