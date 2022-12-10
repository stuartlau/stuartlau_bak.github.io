---
layout:     post
title:      如何绑定你的独立域名到自己的GitHub Pages
subtitle:   以域名服务商Godaddy和域名解析商dnspod为例
date:       2018-09-18
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Site
    - Github
---

之前一直使用GitHub Pages的域名下的二级域名来管理博客，现在决定用自己的域名来进行管理。
由于我的域名托管在GoDaddy，对于国内的用户的DNS解析不是很友好，如可能被墙，所以我采用了国内的dnspod进行域名的解析，对于国内用户来说会友好很多。
# 步骤
- 注册dnspod账户获取NS记录的地址
- 修改GoDaddy中对应域名的NS记录地址
- 等待新记录生效
- 修改dnspod中域名的A记录地址
- 等待新记录生效
- 一切就绪，可以用自己的域名访问博客了

## 注册dnspod账户获取NS记录的地址
根据官网的要求一步一步注册即可，可以使用微信登录（dnspod现在是腾讯的产品）并关联到腾讯云。
注册成功后添加自己的域名，然后系统会自动帮你生成DNS解析相关的记录如A记录、NS记录等。
这里会提供两个dnspod的NS记录地址，`f1g1ns1.dnspod.net`和`f1g1ns2.dnspod.net`

## 修改GoDaddy中对应域名的NS记录地址
登录到GD的后台依次选择： 我的产品 -> 选择域名 -> DNS
在此处修改对应域名的服务器，默认是GD自己的，需要手工修改为上一步获取的两个NS记录地址，然后保存。
注意DNS的任何记录生效都需要一定时间，即缓存失效的时间，取决于记录的TTL值。
想要查看记录是否生效在linux环境下可以直接使用`dig`命令：

    $ dig your-domain
    
修改前

```
 ~ dig elsef.com

; <<>> DiG 9.9.7-P3 <<>> elsef.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13442
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;elsef.com.			IN	A

;; ANSWER SECTION:
elsef.com.		600	IN	A	50.63.202.41

;; AUTHORITY SECTION:
elsef.com.		1915	IN	NS	ns05.domaincontrol.com.
elsef.com.		1915	IN	NS	ns06.domaincontrol.com.

;; ADDITIONAL SECTION:
ns06.domaincontrol.com.	105171	IN	A	173.201.70.3
ns06.domaincontrol.com.	165028	IN	AAAA	2603:5:2260::3
ns05.domaincontrol.com.	105171	IN	A	216.69.185.3
ns05.domaincontrol.com.	165028	IN	AAAA	2607:f208:206::3

;; Query time: 182 msec
;; SERVER: 192.168.25.100#53(192.168.25.100)
;; WHEN: Tue Sep 18 11:38:10 CST 2018
;; MSG SIZE  rcvd: 194
```

修改后

```
 ~ dig elsef.com

; <<>> DiG 9.9.7-P3 <<>> elsef.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34559
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 10

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;elsef.com.			IN	A

;; ANSWER SECTION:
elsef.com.		600	IN	A	50.63.202.41

;; AUTHORITY SECTION:
elsef.com.		86400	IN	NS	f1g1ns1.dnspod.net.
elsef.com.		86400	IN	NS	f1g1ns2.dnspod.net.

;; ADDITIONAL SECTION:
f1g1ns2.dnspod.net.	101602	IN	A	182.140.167.188
f1g1ns2.dnspod.net.	101602	IN	A	61.129.8.159
f1g1ns2.dnspod.net.	101602	IN	A	101.226.220.16
f1g1ns2.dnspod.net.	101602	IN	A	121.51.128.164
f1g1ns1.dnspod.net.	101602	IN	A	58.247.212.36
f1g1ns1.dnspod.net.	101602	IN	A	61.151.180.44
f1g1ns1.dnspod.net.	101602	IN	A	180.163.19.15
f1g1ns1.dnspod.net.	101602	IN	A	182.140.167.166
f1g1ns1.dnspod.net.	101602	IN	A	14.215.150.17

;; Query time: 1099 msec
;; SERVER: 192.168.25.100#53(192.168.25.100)
;; WHEN: Tue Sep 18 13:04:11 CST 2018
;; MSG SIZE  rcvd: 252
```

可以看到NS记录已经生效，即DNS解析工作已经从GD交给了dnspod，下一步是修改A记录让dnspod的域名服务器解析域名到正确的服务器地址。

## 修改dnspod中域名的A记录地址
登录到dnspod的管理后台，给对应的域名的A记录增加几条GitHub Pages的主机地址，注意该地址列表可能会更新，所以最新请戳[here](https://help.github.com/articles/troubleshooting-custom-domains/)

这里我配置了两个做负载均衡，如果配置更多的A记录，需要购买dnspod的增值服务，两个对于我来说已经足够了。

![dnspod-a-record](http://stuartlau.github.io/img/in-post/dnspod-a-record.jpg)

等待一段时间后用dig查看是否生效：

```
 ~ dig elsef.com

; <<>> DiG 9.9.7-P3 <<>> elsef.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8268
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 10

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;elsef.com.			IN	A

;; ANSWER SECTION:
elsef.com.		600	IN	A	185.199.109.153
elsef.com.		600	IN	A	185.199.108.153

;; AUTHORITY SECTION:
elsef.com.		85094	IN	NS	f1g1ns2.dnspod.net.
elsef.com.		85094	IN	NS	f1g1ns1.dnspod.net.

;; ADDITIONAL SECTION:
f1g1ns2.dnspod.net.	100297	IN	A	121.51.128.164
f1g1ns2.dnspod.net.	100297	IN	A	182.140.167.188
f1g1ns2.dnspod.net.	100297	IN	A	61.129.8.159
f1g1ns2.dnspod.net.	100297	IN	A	101.226.220.16
f1g1ns1.dnspod.net.	100297	IN	A	14.215.150.17
f1g1ns1.dnspod.net.	100297	IN	A	58.247.212.36
f1g1ns1.dnspod.net.	100297	IN	A	61.151.180.44
f1g1ns1.dnspod.net.	100297	IN	A	180.163.19.15
f1g1ns1.dnspod.net.	100297	IN	A	182.140.167.166

;; Query time: 48 msec
;; SERVER: 192.168.25.100#53(192.168.25.100)
;; WHEN: Tue Sep 18 13:25:57 CST 2018
;; MSG SIZE  rcvd: 268


```
可以看到A记录已经生效了，我们访问对应域名elsef.com就可以直接指向GitHub Pages的服务器地址了

## 配置GitHub中仓库的CNAME
光指向GitHub的地址还不能够完成访问我们子域名的功能，需要在自己的仓库里配置自定义域名，配置完毕后会在我们的仓库里生成个CNAME文件包含对应的域名。

![]![github-custom-domain](http://stuartlau.github.io/img/in-post/github-custom-domain.jpg)

注意GitHub Pages是提供HTTPS支持的，但是需要在配置CNAME后的24小时之后，因为签名和生效都需要一定的时间。

## 访问自己的域名
试试是否可以用自己的域名访问博客了？
![my_domain](http://stuartlau.github.io/img/in-post/about-me-screenshot.jpg)

成功！

截图中看到我的域名已经支持了HTTPS功能，Issued by Let's Encrypt，这是一个提供免费的SSL/TLS证书的认证机构。

> Let’s Encrypt is a free, automated, and open Certificate Authority.

![lets-encrypt](http://stuartlau.github.io/img/in-post/lets-encrypt.jpg)

# Reference
- https://support.dnspod.cn/Kb/showarticle/tsid/177/#ChangeDomainNS
- https://help.github.com/articles/troubleshooting-custom-domains/
- https://letsencrypt.org/

> 本文首次发布于 [S.L's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,转载请保留原文链接.
