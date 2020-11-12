---
title: Hexo+NexT(v7.0+) 搭建博客：性能优化
copyright: true
related_posts: true
tags:
  - Hexo
  - NexT
  - 阿里云
  - 七牛云
categories: 博客
abbrlink: e824570
date: 2019-05-20 17:15:23
---
在访问很多博客的时候，页面加载和响应速度往往都要上十秒，严重影响用户的体验。
本文将探究如何利用常用的方案来进行性能优化，主要包括:

- CDN加速
- Nginx压缩、缓存
- 图床

<!--more-->

首先，可利用 [Google PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights/) 帮助分析网页加载速度，根据报告结果和优化建议进行针对性的优化。
常见的网站提速方案有：cdn加速，压缩源文件，nginx gzip压缩，减少网站一些不必要的引入，图片大小等。

### CDN加速

在阅读下文之前，如果你还不知道 CDN 是什么，请先移步[百度百科：CDN词条](https://baike.baidu.com/item/CDN) 进行一些了解。
在所有静态资源中，对加载速度影响较大且存在大幅优化空间的主要还是「JavaScript 第三方库」脚本，设定成合适的 CDN 地址，此特性可以加速静态资源的加载。
对于我 Hexo 博客来说，NexT 主题已经做好了配置，只需添加 CDN 加载源，将其改为从公共 CDN 加载即可。
在 <span id="inline-purple">主题配置文件</span> _config.yml 中修改`vendors`：

``` yaml themes/next/_config.yml
  # Example:
  # jquery: //cdn.jsdelivr.net/npm/jquery@2/dist/jquery.min.js
  # jquery: //cdnjs.cloudflare.com/ajax/libs/jquery/2.1.3/jquery.min.js
  jquery: //cdn.jsdelivr.net/npm/jquery@2.1.3/dist/jquery.min.js

  ...  
```

比较常用的开源项目 CDN 服务商主要有 unpkg、bootcdn、 cdnjs、jsdelivr 和 cloudflare，本站主要使用 jsdelivr 提供的 CDN 加速服务。

### 上云

{% note info %}
国内的 CDN服务 要求网站必须备案，但是有些服务商是不支持备案的，于是云主机就是我们需要的了，可以一键备案直接上云。
{% endnote %}

研究了一下各个云服务的价格，1核1G的云主机一年大概都是500+，不过阿里云针对新用户都有很给力的活动：

| 产品名称 | 性能 | 配置 | 时长 | 原价 | 现价 | 折扣 |
| :---: | :---: | :-------------: | :---: | :----: | ------ | ------ |
| 阿里云t5 | **+20%突发性能** | 1核2G内存1M带宽 | 一年 | 992 | 89 | 9% |
| 阿里云t6 | **+10%突发性能** | 2核1G内存1M带宽 | 一年 | 745 | 99 | 13% |
| 阿里云t5 | **+20%突发性能** | 1核2G内存1M带宽 | 三年 | 2977 | 229 | 7% |
| 阿里云n4 | 100%性能 | 2核4G内存3M带宽 | 一年 | 3389 | 399 | 12% |
| 阿里云n4 | 100%性能 | 2核4G内存3M带宽 | 两年 | 6766 | 469 | 7% |
| 阿里云n4 | 100%性能 | 2核4G内存3M带宽 | 三年 | 10148 | 799 | 8% |
| 阿里云t5 | **+20%突发性能** | 2核4G内存1M带宽 | 三年 | 7236 | 639 | 9% |
| 阿里云t5 | **+20%突发性能** | 1核1G内存1M带宽（香港） | 一年 | 972 | 119 | 12% |

看起来的话 [阿里云1核2G的云主机](https://www.aliyun.com/minisite/goods?userCode=wdpvvh4p&share_source=copy_link) 三年只要229 ，简直太白菜价了。建议一次性买三年的，新用户优惠可是只有这一次。

Tips：香港主机的优势在于无需备案，且可以访问墙外的网络，要注意正规建站用途。

### Nginx压缩、缓存

{% note info %}
Nginx 是一个高性能的 Web 服务器，可以适当地分配流量（负载均衡器）、流媒体、动态调整图像大小、缓存内容等等，合理配置可以有效提高网站的响应速度。
{% endnote %}

#### 开启gzip

gzip压缩页面需要浏览器和服务器双方都支持，实际上就是服务器端压缩，传到浏览器后浏览器解压并解析。
修改nginx.conf，在http模块中增加gzip配置：

``` html
  #开启gzip压缩;
  gzip  on;

  #设置允许压缩的页面最小字节数;
  gzip_min_length 1k;

  #设置压缩缓冲区大小，此处设置为4个16K内存作为压缩结果流缓存
  gzip_buffers 4 16k;

  #压缩版本
  gzip_http_version 1.1;

  #设置压缩比率，最小为1，处理速度快，传输速度慢；9为最大压缩比，处理速度慢，传输速度快;级别越高，压缩就越小
  gzip_comp_level 6;

  #制定压缩的类型
  gzip_types text/plain application/x-javascript text/css application/xml text/javascript application/javascript application/json image/svg+xml application/x-font-ttf font/opentype image/x-icon;

  #配置禁用gzip条件，支持正则。此处表示ie6及以下不启用gzip（因为ie低版本不支持）
  gzip_disable "MSIE [1-6]\.";

  #选择支持vary header；改选项可以让前端的缓存服务器缓存经过gzip压缩的页面; 这个可以不写
  gzip_vary on;

```

#### 开启缓存

修改nginx.conf，在server中配置缓存和失效时间：

``` html
location ~* ^.+\.(ico|gif|jpg|jpeg|png)$ {
    access_log off;
    expires 30d;
}

location ~* ^.+\.(css|js|txt|xml|swf|wav)$ {
    access_log off;
    expires 24h;
}

location ~* ^.+\.(html|htm)$ {
     expires 1h;
}
```

### 图床

{% note info %}
目前各大云服务商都提供了对象存储服务，如七牛云 QINIU、又拍云 USS、腾讯云 COS、阿里云 OSS 等。我们可以使用这些服务器来存储图片信息，并将其称为图床。
{% endnote %}

使用图床的好处：
- 可以减轻服务器的存储压力；
- 减轻应为图片带来的额外的流量消耗；
- 图床一般都是具有cdn加速的，可以让你的网页变得更快。

我主要是看中了cdn加速这点，这个对网站的性能提升太重要了。

常用的云存储服务费用对比：

| 限定符 | 免费存储空间 | 免费下载流量 | 免费请求 |免费时间 | HTTPS  | CDN |
| :---: | :---: | :---: | :-----: |:----: | :--: | :--: |
| 微博图床 | 无限	| 无限 | 无限 | 永久 |<i class="fa fa-close"/> | <i class="fa fa-check"/>|
| 七牛云 | 10G	| 10G | PUT: 10万次 <br/>GET: 100万次 | 永久 | <i class="fa fa-check"/> | <i class="fa fa-check"/> |
| 青云QingStor | 30G	| 11G | PUT: 10万次 <br/>GET: 100万次 | 12个月 |<i class="fa fa-check"/> | <i class="fa fa-check"/> |
| 又拍云USS | 10G	| 15G | 无限 | 12个月 | <i class="fa fa-check"/> | <i class="fa fa-check"/> |
| 阿里云OSS | 无	| 无 | 无 | 无 | <i class="fa fa-check"/> | <i class="fa fa-check"/> |
| 腾讯云COS | 50G	| 无 | 无 | 6个月 | <i class="fa fa-check"/> | <i class="fa fa-check"/> |
| Github |100G | 无限 | 无限 | 永久 |<i class="fa fa-check"/> | <i class="fa fa-close"/> |

- 七牛云是专业云服务商，提供比较完备的服务，且免费额度足够个人博客使用。
- 七牛云的定位就是 CDN，让你在浏览网页的时候最快的接收到页面中的图片、音频等文件，所以非常适合个人、企业用户用来储存站点资源，且CDN加速也不会产生太多的费用。
- 微博图床是匿名图床，如果有一天禁止外链访问的话，图片将全部丢失。想着辛辛苦苦制作的图片有丢失的风险，马上就放弃了。【2019年4月微博图床开启了防盗链，对图片 CDN 添加了引用来源`Referer`检测，对于非微博站内引用的请求统统拒绝访问】
- GitHub 看起来是个不错的选择，但是网络访问速度不是很理想，随即放弃了。
- 阿里云OSS也是个不错的选择，有个9元包年40G存储空间，无限流量。

### 七牛云

综合比较之后：我选择了七牛云的对象存储作为图床(高效、快速、有保障)。
![七牛云对象存储](https://image.chingow.cn/images/20190610215145_FVk4s5_Screenshot.jpeg "七牛云对象存储")

#### 注册账号并实名认证 
注册 [七牛开发者平台](https://portal.qiniu.com/signup?code=1hjtnnywndb9u) 账号，并前往 **个人中心**  ->  **个人信息** 实名认证。

####  新建存储空间
- 进入控制台，打开 **对象存储**  -> **新建存储空间**， 即可创建新的Bucket。
【存储区域】：建议选择一个离你较近的CDN
【访问控制】：这里必须选择“公开空间”，因为设置为私有空间，图片的外链是无法访问的。

- 进入新创建的存储空间，在 **空间概览**里点击 **自定义域名** 为空间绑定融合cdn加速域名。详细的参数解释可以参考 [官方域名接入文档](https://developer.qiniu.com/fusion/manual/4939/the-domain-name-to-access) 。
![自定义域名](https://image.chingow.cn/images/20190610224405_2DZajr_Screenshot.jpeg "自定义域名")
【域名类型】：如果没有特殊需求，选择普通域名即可。
【加速域名】：建议填写的是，您未在使用的二级或三级域名等，请勿轻易绑定www域名避免影响您的源站服务。
【源站配置】：当您为存储空间绑定自定义域名的时候，源站配置默认为七牛云存储空间即可。

- 配置CNAME
创建加速域名成功后，七牛云会提供CNAME地址，需要在域名服务提供商处将加速域名指向分配的CNAME地址，配置生效后，即可享受CDN加速服务。根据控制台的引导文档并参考 [官方配置域名CNAME文档](https://developer.qiniu.com/fusion/kb/1322/how-to-configure-cname-domain-name) 。

#### 上传文件
进入新创建的存储空间，在 **内容管理** 中上传、下载、访问、修改资源，这样就可以使用资源的外链了。
上传图片文件以后，复制外链连接就可以利用这个链接访问这个图片了。
![使用资源外链](https://image.chingow.cn/images/20190610224604_5uT2oa_Screenshot.jpeg "使用资源外链")

### 上传工具
如果每次都需要在web端点击上传图片，然后复制外链的操作就比较麻烦了，使用工具可以让我们更加方便地上传资源。
Mac平台上有多款图床工具，找到了几个优秀的工具，做了个对比：

<style>
table th:nth-of-type(2){
width: 15%;;
}
table th:nth-of-type(5){
width: 15%;
}

</style>

| 名称 | 收费标准 | 优点 | 缺点 | 推荐指数 | 下载链接 |
| :---: | :---: | :-------: | :-------: |:-----: | :----: |
| ipic | 60元/年	| 剪贴板、压缩、拖拽上传，功能强大，支持多种云服务 | 免费版只支持微博图床 |  <i class="fa fa-star"/>  | [Mac App Store](https://itunes.apple.com/cn/app/ipic-markdown-%E5%9B%BE%E5%BA%8A-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0%E5%B7%A5%E5%85%B7/id1101244278?mt=12) |
| PicGo | 免费 | 链接上传，支持相册管理 | 不支持清除上传历史 | <i class="fa fa-star"/> <i class="fa fa-star"/> <i class="fa fa-star-half-o"/> | [PicGo.dmg](https://github.com/Molunerfinn/PicGo/releases) |
| PicUploader | 免费 | 压缩上传，多文件、文件夹同时上传 | 不支持顶部菜单 | <i class="fa fa-star"/> <i class="fa fa-star"/> | [PicUploader.zip](https://github.com/xiebruce/PicUploader/releases) |
| 云存储管理 | 免费 | 链接上传，可视化相册管理 | 上传速度太慢，会卡死（不能忍受(°⌓°;） | <i class="fa fa-star"/> <i class="fa fa-star"/> <i class="fa fa-star"/> | [云存储管理客户端](https://github.com/willnewii/qiniuClient) |
| cuImage | 免费 | 剪贴板、压缩、拖拽上传，与ipic类似 | 仅支持七牛云<br/>不支持链接上传 | <i class="fa fa-star"/> <i class="fa fa-star"/> <i class="fa fa-star"/> <i class="fa fa-star"/>  <i class="fa fa-star-half-o"/> | [Mac App Store](https://github.com/hulizhen/cuImage/releases) |

如果是使用七牛云图床我推荐cuImage，它的功能完善，使用剪贴板、拖曳、甚至是快捷键都可以直接将图片上传到云存储，并直接生成Markdown外链，操作十分简便。
