---
layout:     post
title:      "解决Flask-CAS接入SSO的报KeyError问题"
subtitle:   "Solve Flask-CAS Thrown KeyError Issue"
date:       2019-01-30
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Flask
    - TroubleShooting
---
    
> 最近在使用Flask开发IM中台的运营平台，在解决身份为题时接入了公司的SSO服务，Flask的CSA的实现是插件叫flask-cas，但是这个插件的源码存在bug导致在认证的时候会有异常，本文主要解决flask-cas中源码的异常问题并提供解决方案。

## 一分钟介绍CAS
CAS是一个企业SSO系统，支持CAS/OpenId/Oauth/SAML等协议，用于企业内部Web系统的SSO。

### CAS是什么
Web 认证系统，企业内部passport

### CAS不是什么
- CAS不是session管理，session需要应用自己管理
- 权限管理： CAS不负责应用内部的权限管理。应用需要负责管理哪些用户可以登录，哪些不可以；并且需要负责应用内部权限控制。

### 概念定义：
- 下游系统： 待接入CAS认证的Web系统，如git，jira，wiki等

### 协议文档
- 架构文档： https://apereo.github.io/cas/5.0.x/planning/Architecture.html
- CAS协议：https://apereo.github.io/cas/5.0.x/protocol/CAS-Protocol.html

### 官方Python实现
- Flask-CAS client：https://github.com/cameronbwhite/Flask-CAS
- 客户端示例程序：https://github.com/cas-projects/cas-sample-python-webapp

## 使用问题：

### 1、flask_cas在解析sso返回的xml数据的时候，有个bug
一般的xml解析在解析相同的组数据会有两种情况，只有一条数据的时候返回字符串，多条数据的时候返回数组。但是在下面取验证信息的时候，flask_cas没有对这两种情况同时处理。需要对源码进行修改一下，同时兼容两种数据类型。

最简单的修改方式，在routing.py中的128行

>attributes["cas:memberOf"] = attributes["cas:memberOf"].lstrip('[').rstrip(']').split(',')
>
> 修改为
>
>attributes["cas:memberOf"] = str(attributes["cas:memberOf"]).lstrip('[').rstrip(']').split(',')

如果这方法仍然解决不了问题，那么请按照这个[commit](https://github.com/cameronbwhite/Flask-CAS/pull/36/files)修改

### 2、遇到KeyError: 'cas:attribus a tes'报错

原因：使用pip安装flask-CAS时默认安装的时1.0.0，bug在1.0.1版本中被修复

解决：克隆git中的源码手动安装flask-CAS


> 本文首次发布于 [S.L's Blog](http://elsef.com), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
