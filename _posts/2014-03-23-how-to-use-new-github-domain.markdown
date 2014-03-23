---
layout: post
title:  "如何使用新的github域名管理方式托管blog!"
date:   2014-03-23 17:06:25
categories: github jekyll domain
---

2014年2月份，github开始使用新的pages托管方式，将原来的`username.github.com`改为了`username.github.io`，同时也推荐我们使用二级域名的方式来自定义我们的blog，这样可以依靠github的CDN提供更好的服务。

使用的方式也很简便，以我所使用的DNSPOD为例，只需要在域名管理里面添加一条www的CNAME记录，并将地址指向`xiongbo.github.io`即可。

而对于我的blog，则将CNAME中的域名修改为`www.xiongbo.me`即可。
