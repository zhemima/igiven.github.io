---
title : "github下载代码的速度太慢"
---



作为程序员，最大的同性交友网站估计是大家的标配了，常常会苦恼于git clone某个项目的时候速度太慢，看着控制台那几K十一二K的速度，吐血！！

原因很简单：github的[CDN](https://cloud.tencent.com/product/cdn?from=10680)被高高的墙屏蔽所致了。 所以解决方案也很简单，就是手动把 cdn 和IP地址绑定一下。

## 1、获取github地址

访问 http://github.com.ipaddress.com/ 获取cdn域名以及ip地址..或者打开 http://tool.chinaz.com/dns ,这是一个查询域名映射关系的工具 

## 2、获取 global.ssl.fastly地址

http://github.global.ssl.fastly.net.ipaddress.com/ 获取cdn域名以及ip地址

## 3、打开hosts映射

### Windows环境

```javascript
C:\Windows\System32\drivers\etc\hosts
```

最末尾添加两句话保存:

```javascript
151.101.185.194 http://github.global.ssl.fastly.net 
192.30.253.112 http://github.com
```

打开CMD刷新一下DNS就好了。

```javascript
ipconfig /flushdns
```

### Linux环境

```javascript
sudo vi /etc/hosts
```

添加

```javascript
192.30.253.112 https://github.com
151.101.185.194 https://github.global.ssl.fastly.net 
```

保存,退出,并重启网络

```javascript
systemctl restart network
```

## 速度对比:

配置前

```javascript
Receiving objects:  17% (151/883), 348.00 KiB | 18.00 KiB/s
```

配置后

```javascript
Receiving objects:  81% (86141/104384), 81.31Mib | 562.00 KiB/s
```