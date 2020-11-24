---
title: gitstats-java-01-命令介绍
copyright: true
related_posts: true
date: 2020-11-24 23:58:24
tags: gitstats
categories: 
	- git
	- gitstats
---

## gitstats java版

### 前言

年终总结总是要统计项目组 一年的代码提交情况，最早是使用gitstats -python版，github地址是：
[gitstats]: https://github.com/hoxu/gitstats

这个使用了python2.6 ，需要手动download项目下来，然后在本地配置执行，然后会生成一系列可视化的静态页面。这个过程是漫长的，并且页面展示的数据也有可能不是我们想要的。但是给我们提供了一个思路，我们只需要搞清楚 里面的py脚本的原理，便可用java实现 ，定制自己想要的效果 。

本文参考 ：文章[使用Git工具统计代码]: https://blog.cyeam.com/kaleidoscope/2015/01/17/gitstats#1-git-shortlog--s-since2013-12-01-before2015-12-10-head-no-merges	" "



原始脚本主要用到了8个 git 命令：

```bash
git shortlog -s --since=2013-12-01 --before=2015-12-10 HEAD --no-merges

git show-ref --tags

git rev-list --pretty=format:"%at %ai %aN (%aE)" --since=2013-12-01 --before=2015-12-10 HEAD | grep -v ^commit

git rev-list --pretty=format:"%at %T" --since=2013-12-01 --before=2015-12-10 HEAD | grep -v ^commit

git ls-tree -r -l -z HEAD

git log --shortstat --first-parent -m --pretty=format:"%at %aN (%aE)" --since=2013-12-01 --before=2015-12-10 HEAD

git log --shortstat --date-order --pretty=format:"%at %aN (%aE)" --since=2013-12-01 --before=2015-12-10 HEAD

git --git-dir=.git --work-tree=./ rev-parse --short HEAD
```

具体的命令含义，可以参考原文，

我这边的思路，就是把所有的提交历史，影响行数，全部保存到数据库里，

使用 

```bash
 git log --shortstat --date-order  --no-merges --pretty=format:"%H %at %an %s "   HEAD
 （ --no-merges 代表： 只输出修改代码的 commit 的数据，开发分支merge到主分支的请求的commit 是不会被统计的）
 输出示例： 
7847dd178ffe1c50f7ae2ecc8dce4f671b6b0df4 1571158821 jackfrued 更新了第91天的文档和资源
 2 files changed, 108 insertions(+), 28 deletions(-)

662318eadb8efaf86f0bc9ac4623d8c5b45c1657 1571146991 jackfrued 更新了第91天的内容
 7 files changed, 148 insertions(+), 42 deletions(-)

24048b2e1be968750e18d8c2cc5031d6d0ad265e 1570969223 jackfrued 更新了README.md文件
 1 file changed, 1 insertion(+), 2 deletions(-)

cef5d95041b1c1a6e03466325258d627386afbd2 1570969065 jackfrued 更新了最后10天的文档
 17 files changed, 1045 insertions(+), 254 deletions(-)

 
```

将所有的提交记录输出到txt文件上，再去读txt文件，写入数据库，后续展示 只需要看如何写sql就行，可以实现定制化

### 后续思路（已实现）：

- 将需要统计的项目信息配置在数据库中
- 使用egit 自动git clone代码或者 git checkout  and pull最新分支的代码
- 自动将git log 信息写入文件，再读取并解析文件信息写入数据库中 
###  todo （前端echart  图标可视化展示 ）