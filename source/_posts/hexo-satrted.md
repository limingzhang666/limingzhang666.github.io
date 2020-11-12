---
title: Hexo+NexT(v7.0+) 搭建博客：基础安装
copyright: true
related_posts: true
date: 2019-04-29 22:16:23
abbrlink: ae8c6a29
categories: 博客
tags: 
    - Hexo
---
![Hexo](https://image.chingow.cn/background/006tNc79gy1g37jxk0kq5j327a0ki0th.jpg "Hexo")

关于如何搭建Hexo博客的文章已经有很多人写过了，并且有很多人已经写的很深刻很到位了，为什么还要重复写一遍呢？直到我看到了这位同学的博客 [yearito](yearito.cn) **（ ps：本站的建站优化大都参考自这里）** ，我有了说服自己的理由：

* 你可以参考别人的技术方案，集众所长，亲自实践，然后融入自己的思考写出一篇新文章
* 即使并没有做出创新性的贡献，自己重新归纳一遍也有助于梳理流程，深化理解

<!--more-->
<p id="div-border-left-red">现在百度 Google 很方便，动动手指就可以搜索到想要的答案，但是太多人都是**『顺手拈来、过目就忘』**，下次遇到同样的问题再搜索一遍。
为什么会这样呢？不善于总结，不情愿动手思考，时而久之就会变成所谓的 “代码搬运工” ！<p>

闲话不多说了，我们开始吧！

## 安装node.js

在 [官方下载网站](https://nodejs.org/en/download/) 下载源代码，选择最后一项 `Source Code`
解压到某一目录, 然后进入此目录,依次执行以下 3 条命令

``` bash
$ ./configure
$ make
$ sudo make install
```

安装完后查看`node.js`版本，检验是否安装成功

``` bash
$ node -v
```

## 安装hexo

在命令行中通过 **npm** 来安装 hexo：

``` bash
$ npm install -g hexo-cli
```

### 本地启动hexo

创建一个博客目录（例如 `/my-blog`），在此目录下，执行初始化命令

``` bash
$ mkdir -p my-blog
$ cd my-blog
$ hexo init
```

执行完毕后，将会生成以下文件结构：

``` tree
.
|-- node_modules       //依赖安装目录
|-- scaffolds          //模板文件夹，新建的文章将会从此目录下的文件中继承格式
|-- source             //资源文件夹，用于放置图片、数据、文章等资源
|   |-- _posts          //文章目录
|-- themes             //主题文件夹
|   |-- landscape      //默认主题
|-- .gitignore         //指定不纳入git版本控制的文件
|-- _config.yml        //站点配置文件
|-- db.json
|-- package.json
`-- package-lock.json
```

在根目录下执行如下命令启动**hexo**内置的web容器

``` bash
$ hexo generate     # 生成静态文件
$ hexo server       # 在本地服务器运行
```

在浏览器输入IP地址 http://localhost:4000  就可以看到我们熟悉的** Hello Word **了。

![Hello Word](https://image.chingow.cn/images/d7cced3b-950e-6d7b-6edc-dc3058646ddb.png "Hello Word")

### 常用命令简化和组合

``` bash
$ hexo g    # 等同于hexo generate
$ hexo s    # 等同于hexo server
$ hexo p    # 等同于hexo port 
$ hexo d    # 等同于hexo deploy 
```

当本地不想使用默认的4000端口时（比如在服务器上，默认使用80端口），可以使用 port 命令更改启动端口
另外，**hexo**支持命令合并，比方说 生成静态文件 → 本地启动80端口，我们可以执行

``` bash
$ hexo s -g -p 80
```

## 安装NexT主题

hexo 安装主题的方式非常简单, 只需几个简单的命令即可。
将NexT主题文件拷贝至**themes**目录下，然后修改 <span id="inline-blue">站点配置文件</span> _config.yml 中的 `theme`字段为`next`即可。

cd 到博客的根目录下执行以下命令下载主题文件：

``` bash
$ cd my-blog
$ git clone https://github.com/theme-next/hexo-theme-next.git themes/next

$ vim _config.yml
theme: next
```

清除 **hexo**缓存，重启服务

``` bash
$ hexo clean
$ hexo s -g
```

大部分的设定都能在 [NexT官方文档](http://theme-next.iissnan.com/getting-started.html) 里找到, 如主题设定、侧栏、头像、友情链接、打赏等等，在此就不多讲了，照着文档走就行了。
