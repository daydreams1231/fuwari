---
title: Cloudflare优选
published: 2026-02-23
description: 'CF优选知识入门'
image: ''
tags: [CDN]
category: '教程'
draft: false 
lang: 'zh_CN'
---

# CF优选原理
see [here](https://2x.nz/posts/cf-fastip/)

## 什么是优选IP / 优选域名
Cloudflare作为一家CDN提供商, 其拥有许多用于CDN的IP, 但大部分都不适合国内访问. <br>
你访问CDN时, DNS会把域名解析成CDN IP, 你向这个IP发起HTTP请求, 其中包含你访问的域名的信息, CDN会根据该信息, 将请求转发到目标源站, 完成一次访问. <br>
优选IP就是从许多CDN IP中筛选出适合国内的IP, 这些IP就叫优选IP <br>
而优选域名就是把这些IP绑定到一个域名, 这样别人使用时就不需要手动填IP了, 也方便更新优选IP, 这个域名就是优选域名 <br>

## 确定运营商及优选域名
不同的运营商适合不同的优选域名, 不过大部分优选域名都提供了三网优选解析, 所以这一步可要可不要 <br>

优选域名优先考虑: `cf.090227.xyz` <br>
原因很简单, 这个页面除了本身是个优选域名外, 也提供其他优选域名供你选择, 还有三网TCPing记录 <br>

## Cloudflare Worker/Pages 项目
如果你自己的网站是部署在Cloudflare Worker/Pages上的, 首先需要将Pages项目转为Worker <br>
如果你的Worker是通过 `xxx.workers.dev` 这个默认域名访问的, 请将其禁用, 转而新建一个路由, 内容: `you_domain.com/*` <br>
如果你的Worker通过自定义域名访问, 那么你的DNS面板那里应该有一条类型为Worker, 内容是你的Worker的DNS记录, 请在Worker设置里删除这个自定义域 <br>
如果没有该DNS记录, 反而是DNS那里有一个*目标为任意IP, 并开启了CF小黄云* 的DNS记录, 请删除这条DNS记录 <br>

准备完成后, 再去DNS解析那里添加一条CNAME解析, `名称（必需）`就是你的域名, `目标（必需）`即你选择的优选域名, 注意**不要开启CF小黄云** <br>

# 为什么不用BYOIP (Bring Your Own IP)
需要自己设置定时清理无用的IP, 并且用作BYOIP的IP一般不是自己的, 有可能会被别人强制重定向到别人自己的网站, 即免费引流. <br>
更重要的是, 这些IP和CF优选IP的延迟未必差别很大 <br>
图个省心, 建议不用BYOIP <br>