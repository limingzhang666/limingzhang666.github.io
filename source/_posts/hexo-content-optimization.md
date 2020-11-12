---
title: Hexo+NexT(v7.0+) 搭建博客：内容优化
copyright: true
related_posts: true
tags:
  - Hexo
  - NexT
categories: 博客
abbrlink: bd723aed
date: 2019-05-18 16:08:13
---

NexT主题内提供了很多功能来让内容更加丰富，本文介绍了如何开启和定制这些功能，主要包括:

+ 模板设置
+ 文章发布修改时间、字数统计
+ 文章版权声明
+ 链接样式、底部标签样式
+ 图片尺寸处理
+ 代码块复制、显示和隐藏
+ 草稿和发布
<!--more-->

## 模板设置

为了便于创建新文章时更加便利，可以在hexo的`scaffolds`文件夹内创建模板文件，比如我创建的草稿模板

``` markdown scaffolds/draft.md
---
title: {{ title }}
categories: 
tags: 
date: {{ date }}
---
```

## 文章发布修改时间

在 <span id="inline-purple">主题配置文件</span> _config.yml 中修改`post_meta`，可用于控制信息的显示：

``` yaml themes/next/_config.yml
post_meta:
  item_text: true  # 显示文字说明
  created_at: true  # 显示文章创建时间
  updated_at:
    enable: false  # 文章修改时间
    another_day: false  # 只有当修改时间和创建时间不是同一天的时候才显示
  categories: true  # 分类信息
```

## 文章字数统计

该功能由 [hexo-symbols-count-time](https://github.com/theme-next/hexo-symbols-count-time) 提供，效果如图：
![文章统计](https://image.chingow.cn/images/20190602020607_IyueIG_Screenshot.jpeg?420x "文章统计")

在根目录下执行如下命令安装相关依赖：

```
$ npm install hexo-symbols-count-time --save
```

在 <span id="inline-blue">站点配置文件</span> _config.yml 中添加`symbols_count_time`配置，这些配置项主要用于控制每项统计信息是否显示：

``` yaml _config.yml
symbols_count_time:
  symbols: true         # 统计单篇文章字数
  time: true            # 估算单篇文章阅读时间
  total_symbols: false  # 统计站点总字数
  total_time: false     # 估算站点总阅读时间
```

在 <span id="inline-purple">主题配置文件</span> _config.yml 中做如下修改，这些配置项主要用于控制统计信息的显示样式：

``` yaml themes/next/_config.yml
symbols_count_time:
  separated_meta: true  # 是否换行显示 统计信息
  item_text_post: true  # 文章统计信息中是否显示“本文字数/阅读时长”等描述文字
  item_text_total: false   # 站点统计信息中是否显示“本文字数/阅读时长”等描述文字
  awl: 4  # Average Word Length：平均字符长度
  wpm: 275  # Words Per Minute：阅读速度
```

## 文末版权声明

NexT主题已经内置了版权声明功能，只需开启配置即可，效果如下：
![文末版权声明](https://image.chingow.cn/images/20190602011504_NtvIUD_Screenshot.jpeg?600x "文末版权声明")

在 <span id="inline-purple">主题配置文件</span> _config.yml 中开启文章底部的版权声明，版权声明默认使用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议，用户可以根据自身需要修改 `licence` 字段变更协议：

``` yaml themes/next/_config.yml
creative_commons:
  license: by-nc-sa  # 开启版权声明
  sidebar: true # 侧边栏
  post: true # post文章
  language: zh-CN
```

默认版权声明中只有 **本文作者**、**本文链接**、**版权声明** 三项，如果你想添加更多内容，如 **文章标题** 等，需要先在语言配置文件里补全版权信息文案字段：

``` diff themes/next/languages/zh-CN.yml
copyright:
+ title : 本文标题
  author: 文章作者
  link: 原始链接
  license_title: 许可协议
  license_content: "本文章采用 %s 许可协议，转载请保留原文链接及作者。"
```

再修改版权声明布局的相关代码：

``` html themes/next/layout/_partials/post/post-copyright.swig
<ul class="post-copyright">
  <li class="post-copyright-title">
    <strong>{{ __('post.copyright.title') + __('symbol.colon') }}</strong>{#
    #}{{ post.title | default(config.title) }}{#
  #}</li>
  <li class="post-copyright-author">
    <strong>{{ __('post.copyright.author') + __('symbol.colon') }} </strong>{#
  #}{{ post.author || author }}{#

```

在版权样式文件中添加如下样式：

``` css themes\next\source\css\_common\components\post\post-copyright.styl
.swal-overlay {
  background-color: transparent;
}

.copy-success-message {
  box-shadow: 0px 4px 12px rgba(0,0,0,0.15);
  border-radius: 4px;
  width: auto;  
  margin: 16x 0px;
  vertical-align: top;
}

.copy-success-message .swal-content {
  margin: 0px 0px !important;
  padding: 10px 16px;  
  line-height: 1em;
}

.copy-success-message .message-icon {
  color: #52c41a;
  margin-right: 8px;
}

.copy-success-message .message-content {
  font-size: 14px;
}
```

在实际使用过程中，有些文章是转载别人的文章，文末再出现个人版权声明就不太合适。此时可在Front-Matter中设定变量 `copyright` 用于控制是否显示版权信息。
修改文章布局模板中相关代码，使得只有当主题配置文件中 `post_copyright.enable` 字段和 `page.copyright` 字段同时为 `true` 时才会插入版权声明：

``` diff themes/next/layout/_macro/post.swig
- {% if theme.post_copyright.enable and not is_index %}
+ {% if theme.post_copyright.enable and page.copyright and not is_index %}
    <div>
      {% include 'post-copyright.swig' with { post: post } %}
    </div>
  {% endif %}
```

为了批量为每篇新文章设定该变量并赋默认值，可以修改草稿模板内容，这样每篇草稿发布为正文后都会默认显示底部版权信息：

``` diff scaffolds\draft.md
  title: {{ title }}
  tags:
  categories:
+ copyright: true
```

## 链接样式

主题自带的链接样式在hover时是灰色的，颜色不明显。在自定义样式文件中添加样式：

``` css themes/next/source/css/_custom/custom.styl

$link-color = #2780e3;
$link-hover-color = #1094e8;
$sidebar-link-hover-color = #0593d3;  

// 普通链接样式
a, span.exturl {
  &:hover {
    color: $link-hover-color;
    border-bottom-color: $link-hover-color;
  }
  // For spanned external links.
  cursor: pointer;
}

// 侧边栏链接样式
.sidebar a, .sidebar span.exturl{
  &:hover {
    color: $sidebar-link-hover-color;
    border-bottom-color: $sidebar-link-hover-color;
  }
}

// 侧边栏目录链接样式
.post-toc ol a {
  &:hover {
    color: $sidebar-link-hover-color;
    border-bottom-color: $sidebar-link-hover-color;
  }
}

//文章内链接文本样式
.post-body p a{
  color: $link-color;
  text-decoration: none;
  border-bottom: none;
  &:hover {
    color: $link-hover-color;
    text-decoration: underline;
    border-bottom-color: $link-hover-color;
  }
}

// 文章内上下一页链接样式
.post-nav-prev a , .post-nav-next a{
  &:hover {
    color: $link-hover-color;
  }
}
```

## 底部标签添加图标

默认情况下标签前缀是 `#` 字符，可以通过修改主题源码将标签的字符前缀改为图标前缀，效果如图：

![底部标签](https://image.chingow.cn/images/20190602012005_lHglf5_Screenshot.jpeg?140x "底部标签")

在文章布局模板中找到文末标签相关代码段，将 `#` 换成 `<i class="fa fa-tags"></i>` 即可：

``` diff themes/next/layout/_macro/post.swig
  <footer class="post-footer">
    {% if post.tags and post.tags.length and not is_index %}
      <div class="post-tags">
        {% for tag in post.tags %}
-          <a href="{{ url_for(tag.path) }}" rel="tag"># {{ tag.name }}</a>
+          <a href="{{ url_for(tag.path) }}" rel="tag"><i class="fa fa-tags"></i> {{ tag.name }}</a>
        {% endfor %}
      </div>
    {% endif %}
    ...
  </footer>
```

NexT中使用 [FontAwesome](https://fontawesome.com/v4.7.0/icons/) 作为图标库，用户可以在 FontAwesome 上找到心仪的图标来替换标签的字符前缀。

## 图片尺寸处理

{% note info %}
本章节受 bobcn 的[方案](https://github.com/bobcn/hexo_resize_image.js)，自行重构了代码逻辑。
{% endnote %}

有时候原始图片的尺寸不太合适，想指定图片在文章中的大小，但是 **Markdown** 原生的图片语法在**Hexo**中是无效的，这一点让人很困扰（可能是Hexo的Bug，希望以后的版本能够解决这个问题）。
现行的处理办法主要有两种方案，一种是使用html标签

``` html
<img width=200 src="/image/test.jpg" >
```

另一种是 [hexo官方文档](https://hexo.io/zh-cn/docs/tag-plugins.html) 推荐的方式

``` html
{% img [class names] /path/to/image [width] [height] [title text [alt text]] %}
```

但是习惯了 Markdown 的原生语法之后还是觉得这两种都不够简洁高效，用起来多有不便。于是尝试对 Next 主题进行了加强，变相扩展支持了 Markdown 的插图语法：

- 可指定像素
  方法是在 URL 后面添加 `?<width>x<height>`，也可以只指定一个参数，图片会等比例缩放。

   ```markdown
   ![指定像素](/image/test.jpg?200x200)
   ![仅指定width](/image/test.jpg?200x)
   ![仅指定height](/image/test.jpg?x200)
   ```

- 可指定缩放比例
  方法是在 URL 后面添加 `?<scale>`，等比例缩放图片大小至 %。

   ```markdown
   ![指定比例](/image/test.jpg?40)
   ```

如何实现这种效果的呢？首先在自定义脚本目录新建用于处理图片尺寸的 **JavaScript** 脚本

``` js themes/next/source/js/_custom/hexo_resize_image.js
function set_image_size(image, width, height) 
{
    image.setAttribute("width", width + "px");
    image.setAttribute("height", height + "px");
}

function hexo_resize_image()
{
    var imgs = document.getElementsByTagName('img');
    for (var i = imgs.length - 1; i >= 0; i--) 
    {
        var img = imgs[i];
        var src = img.getAttribute('src').toString();
        var fields = src.match(/\?(\d*x\d*)/);
        if (fields && fields.length > 1)
        {
            var values = fields[1].split("x");
            if (values.length == 2)
            {
                var width = values[0];
                var height = values[1];

                if (!(width.length && height.length))
                {
                    var n_width = img.naturalWidth;
                    var n_height = img.naturalHeight;
                    if (width.length > 0)
                    {
                        height = n_height*width/n_width;
                    }
                    if (height.length > 0)
                    {
                        width = n_width*height/n_height;
                    }
                }
                set_image_size(img, width, height);
            }
            continue;
        }

        fields = src.match(/\?(\d*)/);
        if (fields && fields.length > 1)
        {
            var scale = parseFloat(fields[1].toString());
            var width = scale/100.0*img.naturalWidth;
            var height = scale/100.0*img.naturalHeight;
            set_image_size(img, width, height);
        }
    }
}
window.onload = hexo_resize_image;

```

然后在自定义布局文件最后添加 **JavaScript** 声明

``` html  themes/next/layout/css/_custom/custom.swig
<script type="text/javascript" src="/js/custom/hexo_resize_image.js"></script>
```

## 代码复制

NexT主题已经内置了代码复制功能，只需开启配置即可，效果如下：
![代码复制](https://image.chingow.cn/images/20190602170547_O2y1Oe_Screenshot.jpeg?600x "代码复制")

在 <span id="inline-purple">主题配置文件</span> _config.yml 中开启代码复制功能：

``` yaml themes/next/_config.yml
copy_button:
  enable: true  # 开启代码复制功能
  show_result: true  # 显示复制结果
```

搜索的按钮有点移位，在自定义样式文件中调整样式：

``` css themes\next\source\css\_custom\custom.styl
// 复制按钮样式top调整
.highlight-wrap .copy-btn {
  padding: 1px 6px;
  top: 3px;
}
```

## 代码块显示和隐藏

--- 待完成 ---

## 草稿和发布

<p id="div-border-left-blue">一般我们使用` hexo new <title> `来建立文章，这种建立方法会将新文章建立在 **source/_posts** 目录下，当使用 hexo generate 编译文件时，会将其 HTML 结果编译在 public 目录下，之后` hexo server `将会把 public 目录下所有文章发布。</p>

{% note danger %}
这种建立文章方式是有缺点的！写文章的人都知道，一篇文章从创作到发布需要经过多次润色，若我们的文章还在创作润色中，尚未编辑完成，执行 **hexo server** 时也会随着一起发布，这样对读者是不友好的。
{% endnote %}
Hexo 另外提供 draft 机制，它的原理是新文章将建立在 **source/_drafts** 目录下，因此并不会将其编译到 public 目录下发布，而且提供了很友好的预览功能。

``` bash
$ hexo new draft <title>	# 新建草稿文章
$ hexo s --draft	        # 预览草稿文章
```

将草稿发布为正式文章：

``` bash
$ hexo P <filename>
```

其中 `<filename>` 为不包含 md 后缀的文章名称。它的原理只是将文章从 source/_drafts 移动到 source/_posts 而已。