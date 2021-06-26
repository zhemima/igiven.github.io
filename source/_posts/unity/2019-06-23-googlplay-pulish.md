---
title:  "googlePlay 发布问题"
---


# 问题 隐私声明
使用这个站点https://app-privacy-policy-generator.firebaseapp.com/ 根据app的名称，类型，平台等等，选择对应的选项，右边按钮还包含对应的第三方隐私服务链接，如果你用到的话，比如google play service, firebase等等，那么就勾选上。最后点击GENERATE，就会生成一个适合你自己app的privacy-policy页面了。
最后的最后，记得将这个url输入到google play后台的隐私设置区域，点击保存，重新发布app等候google play团队的审核。一两个小时就好了。

https://blog.usejournal.com/how-to-fix-advertising-id-policy-violation-in-google-play-store-6d9cf92d335d



# 问题 -1002
Google Play应用需要授予"允许后台弹窗"的权限


# 问题  此商品无法在您设备所在的国家地区安装
- 删除家庭内容..
- 使用vpn切换到指定国家
- googleplay--账号里切换地区


# 问题 无法购买您要的商品

做Google Play 支付， 遇到“无法购买您要的商品”的问题，在网上搜了些答案，下面罗列了一些

1. 保证 versionCode 和版本号与你上传的apk的包的一样。

2. 保证后台和你传入的购买商品的 id 一致。

3. 确保你所使用的账号是在测试人员里。

在"APK"页面里，有一个“选择使用网址”，把这个网址给你的测试人员，让你的测试人员用他的google账号点进去，点那个“成为测试人员”（前提是你把他加进了测试人员列表），还需要把连接地址发送给测试人员，必须要测试人员点击同意参加测试才行！！



# 问题 无法使用该应用,此应用的测试版尚未发布，或者无法通过此帐号使用。
https://play.google.com/apps/testing/com.zhepama.gyj  发给测试人员激活..记住..登录的用户一定是测试人员

# 问题 此版本的应用未配置为通过googleplay 结算

造成这个错误的原因有两个:

- 第一个是打包的时候，versionCode的值比提交到google play后台的版本要高。 
- 第二个就是：打包的时候，和google play后台上的包的签名不一致。

# 问题  关于如何测试
先使用release版本发布到googleplay.测试的时候可以使用debug版本,先测试下基本功能..查看哪里报错..

apk可以使用谷歌提供的测试地址下载

### 以下是Google IAB测试的要求清单。
#### 先决条件：
- AndroidManifest必须包含“com.android.vending.BILLING”权限 。
- APK内置于发行模式 。
- APK已使用发行证书进行 签名 。
- 至少一次将APK上传到Alpha / Beta版本的渠道（以前 - 作为草稿）到开发者控制台。 （需要一段时间〜2h-24h ）。
- IAB 产品已发布 ，其状态设置为活动 。
- 测试帐户被添加到开发者控制台中。

#### 测试要求：
- 测试APK 与上传到开发者控制台的版本号相同 。
- 测试APK的签名与上传到dev.console的证书相同 。
- 测试帐户 （不是开发人员） - 是设备上的主要帐户 。
- 测试帐户作为测试者选择加入，并与有效的付款方式相关联 。 （ @Kyone ）
- PS： 使用发行证书进行调试  



## 相关链接

-  [https : //stackoverflow.com/a/15754187/1321401]()