---
title:  "unity使用代理"
---

Unity的AssetStore下载package的时候经常抽风，而且开了代理工具的全局代理依然无效。

检索网络后得知，这是因为它下载的时候不检测IE代理设置，而是取环境变量中HTTPS_proxy和HTTP_proxy的值，所以添加这两个变量并指定其为你的代理服务地址就可以了。

具体步骤：

    打开 系统属性->高级->环境变量
    新建 HTTPS_PROXY 和 HTTP_PROXY 系统变量，设置其为你的代理服务地址
    
    例如公司的代理IP是：127.0.0.1 端口：1080 
    变量名：HTTPS_PROXY
    变量值：http://127.0.0.1:1080    这里也是http..因为ss没开https
    
    变量名：HTTP_PROXY
    变量值：http://127.0.0.1:1080
![](../../assets/images/2019-11-15-unity-proxy/20190515220955161%5B1%5D.png)



需要注意的是设置完成后可能需要重启Unity才会生效。
