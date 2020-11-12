---
title: MacOS VPN(Cisco IPSec)断线解决办法
copyright: true
related_posts: true
tags:
  - Mac
  - VPN
categories: Mac
abbrlink: a02e5e7a
date: 2020-05-19 20:12:02
---


现在很多VPN客户端都是不支持Mac的，所以更多的时候我们需要使用Mac系统内置的`IPsec VPN `客户端工具去连接。

但是在链接 Cisco VPN 服务器后，一般每隔45分钟到一小时就会自动断线，这无疑是令人沮丧的。虽然说这是Mac系统为了电脑的安全性所做的考虑，但是实际使用时可能会很不方便。

本文章将介绍一种的解决方法来让 VPN 客户端正常连接不断线。

<!--more-->

## 方案参考

本解决方案来自苹果的官方支持社区 [Apple Support Communities](https://discussions.apple.com/thread/3275811?start=0&tstart=0)

## 解决步骤

1. 连接 VPN(Cisco IPSec) ，让系统生成配置文件
2. 执行拷贝命令，将配置文件拷贝到/etc/racoon（xxx.xxx 即VPN服务器ip地址）：

    ``` bash
    $ sudo cp /var/run/racoon/xxx.xxx.xxx.xxx.conf /etc/racoon
    ```

3. 修改 racoon 配置文件，使用定制的配置文件：
   将最后一行注释掉（目的是不使用系统生成配置），修改为使用定制的配置文件，注意行尾的分号：

    ``` diff /etc/racoon/racoon.conf
    - # include "/var/run/racoon/*.conf” ;  
    + include “/etc/racoon/xxx.xxx.xxx.xxx.conf” ;
    ```
4. 修改 IPSec 配置文件（注意使用 sudo 命令）:

   + 取消 dead peer 检测（原来为 20）: 

    ``` diff /etc/racoon/xxx.xxx.xxx.xxx.conf
    -  dpd_delay 20; 
    +  dpd_delay 0;
    ```

   + 修改请求方式为 claim（重要，原来为 obey）:
  
    ``` diff /etc/racoon/xxx.xxx.xxx.xxx.conf
    -  proposal_check obey;
    +  proposal_check claim;
    ```

   + 修改请求周期（所有 proposal 中的值，原来为 3600 sec）:

    ``` diff /etc/racoon/xxx.xxx.xxx.xxx.conf
    - lifetime time 3600 sec; 
    + lifetime time 24 hours;
    ```

5. 重新链接 VPN(Cisco IPSec)，即可永不断线。

{% note danger %}
注意：当原来的ip地址发生变化时，会拨号失败。 
解决方法：将xxx.xxx.xxx.xxx.conf 文件名改为新的ip地址，再执行一次步骤3即可。
{% endnote %}