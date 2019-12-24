---
layout:     post
title:     Scala 语言入门
subtitle:   大数据处理
date:       2019-10-18
author:     Yi Xia
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - 大数据处理
---

# Scala 语言
## 配置环境
### “笨笨”的 vim
开始用 vim 编写脚本然后 scalac 生成class 文件，在运行这个 class 文件，最麻烦的我感觉就是要切出 vim 在进行命令行运行（当然也可以在.vimrc配置在 vim 命令模式下进行直接编译）虽 vim 为“编译器之神”，但是对于我这种 vim 命令不是太熟悉的还是不太友好的。
![-w882](/img/blog_img/15718138445659.jpg)

### idea 安装 scala
- idea搜索scala 插件并安装
![-w497](/img/blog_img/15717930679129.jpg)
- 重启完之后，新建一个工程</br>
![-w380](/img/blog_img/15717931126054.jpg)
- **默认就是这样，但是在这里要强调，Scala的class文件是动态类，所以不能执行main方法，我们只能创建一个Object（这是静态的，后续再讨论）。**

- 所以点击kind下拉选，选择Object

- 我们创建了一个HelloWorld.Object,，在里面输入如下代码

```scala
def main(args: Array[String]): Unit = {
    println("Hello World")
  }
```
![-w1438](/img/blog_img/15717932407668.jpg)

- 这样我们的第一个Scala工程就建好了。

**大师始于“Hello World”**