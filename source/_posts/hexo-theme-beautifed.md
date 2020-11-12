---
title: Hexo+NexT(v7.0+) 搭建博客：主题美化
tags:
  - Hexo
  - NexT
categories: 博客
date: 2019-05-17 10:07:53
copyright: true
related_posts: true
abbrlink: c7372a12
---

本文介绍了在 NexT(v7.0+) 主题的基础上整体布局美化的方案，主要包括:

+ Font：字体、动态背景canvas_ribbon
+ Top：进度条、Follow me、菜单背景收缩
+ Footer：备案显示、运行时间、访客统计
+ Sidebar：近期文章、浏览进度、左边显示、目录展开

大家可以选择喜欢的方案美化自己的博客，当然最好可以再加入点自己的想法改造。
<!--more-->

## Font

前端的美化，可以做的比较多，比如修改字体、修改背景等等。
首先在 <span id="inline-purple">主题配置文件</span> _config.yml 中修改配置`font`：

``` yaml themes/next/_config.yml
font:
  enable: true  # 开启前端设置
```

### 修改字体

默认的字体是微软雅黑，有点审美疲劳了 (lll￢ω￢)，在浏览其他人网站的时候看到了一种很有科技感的字体 『**Monda**』，感觉还是不错的。如何应用到自己的博客呢？
首先，可以从 [frontyukle](https://www.fontyukle.net/cn/1,MONDA) 下载字体文件，在服务器上安装。
然后在 <span id="inline-purple">主题配置文件</span> _config.yml 中的修改全局字体配置：

``` diff themes/next/_config.yml
global:
- family: Lato
+ family: Monda   # 设置Monda字体
```

### 设置三角丝带背景

{% note info %}
该功能由 Vue 作者 [尤雨溪](http://evanyou.me/) 首创。注意：添加动态背景会极大增加页面内存占用及 CPU 消耗。
{% endnote %}

首先，在根目录下执行以下命令安装相关依赖：

``` bash
$ git clone https://github.com/theme-next/theme-next-canvas-ribbon themes/next/source/lib/canvas_ribbon
```

然后，在 <span id="inline-purple">主题配置文件</span> _config.yml 中的修改配置`canvas_ribbon`：

``` yaml themes/next/_config.yml
canvas_ribbon:
  enable: true	# 开启随机三角丝带背景
  size: 90	# 设置丝带宽度
```

## Top

顶部我们保持简洁为主，设置主要包括：进度条、Fork、菜单等等。

### 加载进度条

当网络不好的时候会出现白屏等待，此时如果能有加载进度提示将会提高用户操作体验。
首先，在根目录下执行以下命令安装相关依赖：

``` bash
$ git clone https://github.com/theme-next/theme-next-pace themes/next/source/lib/pace
```

然后，修改 <span id="inline-purple">主题配置文件</span> _config.yml 中的配置`pace` 改为 `true`，并从上面提供的样式中选择一种填入`pace_theme`中就可以了。

### 右上角的Follow-Github

如果你想大家在看博客的时候能快速链接到你的Github，不妨设置**Github_banner**，效果图如下：
![github_banner](https://image.chingow.cn/images/23aaa7f0-a7e7-b170-deef-a8ffb649a473.png?500x "github_banner")

修改 <span id="inline-purple">主题配置文件</span> _config.yml 中的配置`github_banner`，在`permalink`里配置自己的github地址：

``` yaml themes/next/_config.yml
github_banner:
  enable: true
  permalink: https://github.com/yourname
```

### 菜单背景收缩

在 Muse主题方案中 Header 和 Footer 是没有背景色的，的颜色与内容一致不易区分，而且网站标题的字体不够明显，我做了几个修改

+ 头部和底部颜色修改
+ 网站标题加粗和颜色修改
+ 菜单字体增大显示

在主题自定义样式文件中添加以下样式：

``` css themes/next/sources/_custom/custom.styl
//窗口效果相关样式
.sidebar {
  box-shadow: none;
}

// 为Header和Footer添加背景色
#header, #footer {
  background-color: rgb(245, 245, 245);
}

//防止sidebar和footer同时开启动效时堆叠异常
#sidebar, header {
  z-index: 1 !important;
}

//防止挡住页末文章的阅读全文按钮
.main {
  padding-bottom: 30px;
}

// 加宽菜单间距，放大菜单图标
#menu .menu-item {
  margin: 0px 14px;

  .fa {
    font-size: 16px;
  }
}

// Muse主题下自定义样式
if hexo-config('scheme') == "Muse" {
  .site-meta {
    .brand {
      color: rgb(34, 34, 34);
      background: none;
    }

    .site-title {
      font-size: 24px;
      font-weight: bold;
    }
  }
}
```

改完才发现顶部菜单的空间太大了，于是调节菜单高度，在自定义布局文件中添加以下代码：

``` html themes/next/layout/_custom/custom.swig

{# 页面加载时header高度收缩动效 #}
<script>
  $(document).ready(function () {
    $(".header-inner").animate({padding: "25px 0 25px"}, 1000);
  });
</script>
```

如果 custom.swig 文件不存在，需要手动新建并在布局页面中 body 末尾引入：

``` diff themes/next/layout/_layout.swig
    ...
  {% include '_third-party/copy-code.swig' %}
  {% include '_third-party/chatra.swig' %}
  {% include '_third-party/tidio.swig' %}

+  {% include '_custom/custom.swig' %}
</body>
```

调节移动端显示的高度

``` diff themes/next/source/css/_schemes/Muse/_menu.styl
mobile() {
  position: absolute;
  left: 0;
-  top: 52px;
+  top: 95px;
  margin: 0;
  ...
}
```

## Footer

底部设置的可完性就比较多了，建站信息、备案信息、以及访客和统计信息等信息都可以个性化定制。

### ICP备案和公安备案

我们的网站已经有备案号了，但是应该如何添加备案号到博客最下面呢？
首先，在 <span id="inline-purple">主题配置文件</span> _config.yml 中的配置添加备案配置：

``` diff themes/next/_config.yml
 # 这里打开备案信息，填写ICP备案号
   beian:
    enable: true
    icp: 鄂ICP证18014719号

 # 这里是新加的内容，填写公安备案信息
+  gongan:
+    enable: true
+    local: 鄂
+    num: 42011102003178
```

然后，在语言配置文件里，添加表述文案`beian`：

``` diff themes/next/languages/zh-CN.yml
footer:
+   beian: 
+    gongan: 公网安备%s号
```

接下来，删除原有的布局文件中备案样式的代码：

``` diff  themes/next/layout/_partials/footer.swig
  #}{% set current = date(Date.now(), "YYYY") %}{#
- #}{% if theme.footer.beian.enable %}{#
- #}  {{ next_url('http://www.beian.miit.gov.cn', theme.footer.beian.icp + ' ') }}{#
- #}{% endif %}{#
  #}&copy; {% if theme.footer.since and theme.footer.since != current %}{{ theme.footer.since }} – {% endif %}{#
  ...
  <span class="author" itemprop="copyrightHolder">{{ theme.footer.copyright || author }}</span>

+  <span class="post-meta-divider footer-ages-icon"> | </span>

  {% if config.symbols_count_time.total_symbols %}
```

最后，在主题布局文件中添加我们自定义的备案样式：

``` diff  themes/next/layout/_layout.swig
  <div class="footer-inner">
    {{ partial('_partials/footer.swig', {}, {cache: theme.cache.enable}) }}
    {% include '_third-party/analytics/analytics-with-widget.swig' %}
+        <div class="footer-custom">
+            {% if theme.footer.beian.enable %}{#
+          #}{{ next_url('http://www.beian.miit.gov.cn', theme.footer.beian.icp + ' ') }}{#
+          #}{% endif %}
+          {% if theme.footer.beian.enable and theme.footer.gongan.enable %}
+            <span class="post-meta-divider">|</span>
+          {% endif %}
+          {% if theme.footer.gongan.enable %}{#
+          #} <span style="padding-left:25px;background:url(/images/beian.png) no-repeat left center" rel="nofollow">{#
+          #} {{ theme.footer.gongan.local}}{{__('footer.beian.gongan', next_url('http://www.beian.gov.cn/portal/registerSystemInfo?recordcode='+theme.footer.gongan.num, theme.footer.gongan.num )) }}{#
+          #}{% endif %}
+        </div>
    {% block footer %}{% endblock %}
  </div>
```

### 网站的建站时间

NexT主题已经内置了添加建站时间。在 <span id="inline-purple">主题配置文件</span> _config.yml 中修改配置`since` ：

``` yaml themes/next/_config.yml
footer:
  # Specify the date when the site was setup. If not defined, current year will be used.
  since: 2018  # 修改建站时间

  # Icon between year and copyright info.
  icon:
    # `heart` is recommended with animation in red (#ff0000).
    name: heart  # 改成心型图标
    # Change the color of icon, using Hex Code.
    color: "#ff0000" # 改成红色图标

```

### 网站的运行时间

{% note info %}
本章节受 [Yearito's Blog | 站点运行时间统计 ](http://yearito.cn/posts/hexo-advanced-settings.html) 的启发，自行重构了代码逻辑。
{% endnote %}

接下来我们显示网站的运行时间，在这里我做了一些个性化文案，可供大家参考：
![网站建站运行时间](https://image.chingow.cn/images/00c3834a-cf72-22a3-cd7c-6486649fd648.png?550x "网站建站运行时间")

在 <span id="inline-purple">主题配置文件</span> _config.yml 中的添加 `ages`配置：

``` yaml themes/next/_config.yml
  ages: # site running time
    enable: true
    birthday: 20190419    # 网站运行时间
    color: "#1094e8"  
```

在语言配置文件里，添加表述文案`age`：

``` diff themes/next/languages/zh-CN.yml
footer:
+   age: 我已在此等候你
```

在主题自定义布局文件中添加以下代码：

``` html themes/next/layout/_custom/custom.swig
{# 页脚站点运行时间统计 #}
  {% if theme.footer.ages.enable %}
    <script src="https://cdn.jsdelivr.net/npm/moment@2.22.2/moment.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/moment-precise-range-plugin@1.3.0/moment-precise-range.min.js"></script>
    <script>
      function timer() {
        var ages = moment.preciseDiff(moment(),moment({{ theme.footer.ages.birthday }},"YYYYMMDD"));
        //去除时分秒信息
        ages = ages.replace(/\s?\d{0,2}\s+hours?/, "");
        ages = ages.replace(/\s?\d{0,2}\s+minutes?/, "");
        ages = ages.replace(/\s?\d{0,2}\s+seconds?/, "");
        //将年月日转换为中文
        ages = ages.replace(/years?/, "年");
        ages = ages.replace(/months?/, "月");
        ages = ages.replace(/days?/, "天");
        ages = ages.replace(/\d+/g, '<span style="color:{{ theme.footer.ages.color }}">$&</span>');
        span.innerHTML = `{{ __('footer.age')}} ${ages}`;
      }
      var span = document.createElement("span");
      //插入到agesicon之后
      var agesicon = document.querySelector(".footer-ages-icon");
      document.querySelector(".copyright").insertBefore(span, agesicon.nextSibling);
      timer();
    </script>
  {% endif %}
```

### 添加访客统计

该功能由 [不蒜子](http://ibruce.info/2015/04/04/busuanzi/) 提供。UV：独立访客数，PV：网站浏览量，访客数和浏览量的区别在于一个用户连续点击n篇文章，会记录n次浏览量，但只记录一次访客数，效果如图：
![网站访客统计](https://image.chingow.cn/images/0298aada-faa2-1f09-f68a-e0382168a5ed.png?550x "网站访客统计")

{% note info %}
由于不蒜子是基于域名来进行统计计算的，所以通过 localhost:4000 端口访问的时候统计数据UV和PV都会异常的大，属于正常现象。
{% endnote %}

首先，在 <span id="inline-purple">主题配置文件</span> _config.yml 中打开不蒜子功能：

``` yaml themes\next\_config.yml
busuanzi_count:
  enable: true
  total_visitors: true   # 访客数
  total_visitors_icon: user
  total_views: true   # 访问量
```

然后，在语言配置文件里，修改个性化表述文案：

``` diff themes/next/languages/zh-CN.yml
footer:
- total_views: 总访问量
- total_visitors: 总访客量
+ total_views: "历经 %s 次回眸才与你相遇"
+ total_visitors: "我的第 %s 位朋友，"
```

最后，修改不蒜子模板文件：

``` diff themes/next/layout/_third-party/analytics/busuanzi-counter.swig
<div class="busuanzi-count">
  <script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>

+  {% if theme.busuanzi_count.enable %}
+  <script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>
+
    {% if theme.busuanzi_count.total_visitors %}
-    <span class="post-meta-item-icon">
-      <i class="fa fa-{{ theme.busuanzi_count.total_visitors_icon }}"></i>
-    </span>
-    <span class="site-uv" title="{{ __('footer.total_visitors') }}">
-      <span class="busuanzi-value" id="busuanzi_value_site_uv"></span>
+       <span class="site-uv">
+       {{ __('footer.total_visitors', '<span class="busuanzi-value" id="busuanzi_value_site_uv"></span>') }}
    </span>
    {% endif %}

-   {% if theme.busuanzi_count.total_visitors and theme.busuanzi_count.total_views %}
-     <span class="post-meta-divider">|</span>
-   {% endif %}
-
    {% if theme.busuanzi_count.total_views %}
-     <span class="post-meta-item-icon">
-       <i class="fa fa-{{ theme.busuanzi_count.total_views_icon }}"></i>
-     </span>
-     <span class="site-pv" title="{{ __('footer.total_views') }}">
-       <span class="busuanzi-value" id="busuanzi_value_site_pv"></span>
+     <span class="site-pv">
+       {{ __('footer.total_views', '<span class="busuanzi-value" id="busuanzi_value_site_pv"></span>') }}
      </span>
    {% endif %}
+ {% endif %}
</div>
```

在自定义样式文件中添加如下样式：

``` css themes/next/source/css/_custom/custom.styl
//修改不蒜子数据颜色
.busuanzi-value {
  color: #1890ff;
}
```

## Sidebar

侧边栏里的信息有很多，比如站点信息、RSS、链接信息、目录信息等等，我们要有选择性的显示。尽量保持Next主题的宗旨 -- <span id="inline-green">简洁</span>

### 显示近期文章

当你新写了一些好文章，想尽快告诉大家要怎么办呢？放在侧边栏里那是最合适不过了～(￣▽￣～)(～￣▽￣)～ 比如这样：
![近期文章](https://image.chingow.cn/images/f2840b83-ff46-dd56-0aa1-1af57c5e5dad.png "近期文章")

首先配置开关和文案，便于随时修改。
在 <span id="inline-purple">主题配置文件</span> _config.yml 中添加近期文章`recent_posts`的配置：

``` yaml  themes/next/_config.yml
# 近期文章
recent_posts: true
recent_posts_layout: block
```

在语言配置文件里，添加表述文案`recent_posts`：

``` diff themes/next/languages/zh-CN.yml
sidebar:
+   recent_posts: 近期文章
```

在侧边栏原有布局文件中的合适位置添加近期文章显示代码：

``` diff themes/next/layout/_macro/sidebar.swig
  {% if theme.social %}
  ...
  {% endif %}

+ <!-- 添加近期文章 -->
+ {% if theme.recent_posts %}
+   <div class="links-of-blogroll motion-element {{ "links-of-blogroll-" + theme.recent_posts_layout  }}">
+     <div class="links-of-blogroll-title">
+       <!-- modify icon to fire by szw -->
+       <i class="fa fa-history fa-{{ theme.recent_posts_icon | lower }}" aria-hidden="true"></i>
+       {{ __('sidebar.recent_posts') }}
+     </div>
+     <ul class="links-of-blogroll-list">
+       {% set posts = site.posts.sort('-date') %}
+       {% for post in posts.slice('0', '5') %}
+         <li>
+           <a href="{{ url_for(post.path) }}" title="{{ post.title }}" target="_blank">{{ post.title }}</a>
+         </li>
+       {% endfor %}
+     </ul>
+   </div>
+ {% endif %}
```

### 显示当前浏览进度

在 <span id="inline-purple">主题配置文件</span> _config.yml 中修改`back2top`的`scrollpercent`：

``` yaml themes\next\_config.yml
back2top:
  enable: true
  sidebar: false
  scrollpercent: true  #  浏览页面的时候显示当前浏览进度  
```

配置完之后，发现这个按钮颜色有点单调，让它炫酷点的吧 <(￣ˇ￣)/ ，每次刷新可以生成不同的颜色，在自定义样式文件中添加如下样式：

``` css themes/next/source/css/_custom/custom.styl
// 回到顶部样式
.back-to-top .fa-arrow-up:before{
  color: rgb(random-color(0, 255) - 50%, random-color(0, 255) - 50%, random-color(0, 255) - 50%);
}
```

### 侧边栏移到左边

用惯了**Ofice**的人习惯导航菜单在左边的格式，如何把侧边栏放在左侧呢？
NexT 主题中 Pisces 和 Gemini 支持通过主题配置文件来将侧边栏置于左侧或右侧。
在 <span id="inline-purple">主题配置文件</span> _config.yml 中修改配置`sidebar` ：

``` yaml  themes/next/_config.yml
sidebar:
# Sidebar Position, available values: left | right (only for Pisces | Gemini).
  position: left   # 调整侧边栏显示位置，仅支持 Pisces 和 Gemini 主题
```

而 Muse 和 Mist 则需要深度修改源码才能实现改变侧边栏位置：

``` css themes/next/source/css/_custom/custom.styl
.sidebar-toggle {
  left: 30px;
}

.sidebar {
  left: 0px;
}
```

修改动效脚本代码：

``` diff themes/next/source/js/src/motion.js
$(document)
  .on('sidebar.isShowing', function() {
    NexT.utils.isDesktop() && $('body').velocity('stop').velocity(
-     {paddingRight: SIDEBAR_WIDTH},
+     {paddingLeft: SIDEBAR_WIDTH},
      SIDEBAR_DISPLAY_DURATION
    );
  })
  ...
  hideSidebar: function() {
-   NexT.utils.isDesktop() && $('body').velocity('stop').velocity({paddingRight: 0});
+   NexT.utils.isDesktop() && $('body').velocity('stop').velocity({paddingLeft: 0});
    this.sidebarEl.find('.motion-element').velocity('stop').css('display', 'none');
    this.sidebarEl.velocity('stop').velocity({width: 0}, {display: 'none'});

    sidebarToggleLines.init();
    ...
}
```

如此以来就可以将侧边栏放置在左边了，但当窗口宽度缩小到991px之后会出现样式错误：侧边栏收缩消失但是页面左侧仍留有空白间距，此时修改如下代码即可：

``` diff themes/next/source/css/_common/scaffolding/base.styl
body {
  position: relative; // Required by scrollspy
  font-family: $font-family-base;
  font-size: $font-size-base;
  line-height: $line-height-base;
  color: $text-color;
  background: $body-bg-color;

- +mobile() { padding-left: 0 !important; }
- +tablet() { padding-left: 0 !important; }  
+ +mobile() { padding-right: 0 !important; }
+ +tablet() { padding-right: 0 !important; }
  +desktop-large() { font-size: $font-size-large; }
}
```

### 侧边栏目录展开

侧边栏的TOC目录每次滑到哪儿，目录展开到哪。这样查看文章结构就很不方便，如何设置才能让文章目录默认全部都展开？
在 <span id="inline-purple">主题配置文件</span> _config.yml 中修改配置`toc` ：

``` yaml  themes/next/_config.yml
toc:
  wrap: false  ## 如果标题超长，是否换行
  expand_all: true  ## 侧边栏是否完全展开
```
