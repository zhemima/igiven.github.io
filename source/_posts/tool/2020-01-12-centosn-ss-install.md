---
title:  "CentOS 下安装Shadowsocks 搭建ss"
---

CentOS 7 开始默认使用[Systemd](https://en.wikipedia.org/wiki/Systemd)作为开启启动脚本的管理工具，[Shadowsocks](https://github.com/shadowsocks/)则是当前比较受欢迎的科学上网工具，本文将介绍如何在 CentOS 下安装和配置 Shadowsocks 服务。

## 安装 pip

[pip](https://pip.pypa.io/en/stable/installing/)是 python 的包管理工具。在本文中将使用 python 版本的 shadowsocks，此版本的 shadowsocks 已发布到 pip 上，因此我们需要通过 pip 命令来安装。

在控制台执行以下命令安装 pip：

```bash
curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
python3 get-pip.py
```

## 安装配置 shadowsocks

在控制台执行以下命令安装 shadowsocks：

```bash
pip install --upgrade pip
pip install shadowsocks
```

安装完成后，需要创建配置文件`/etc/shadowsocks.json`，内容如下：

```json
{
  "server": "0.0.0.0",
  "server_port": 8388,
  "password": "uzon57jd0v869t7w",
  "method": "aes-256-cfb"
}
```

说明：

- `method`为加密方法，可选`aes-128-cfb, aes-192-cfb, aes-256-cfb, bf-cfb, cast5-cfb, des-cfb, rc4-md5, chacha20, salsa20, rc4, table`
- `server_port`为服务监听端口
- `password`为密码，可使用[密码生成工具](http://ucdok.com/project/generate_password.html)生成一个随机密码

以上三项信息在配置 shadowsocks 客户端时需要配置一致，具体说明可查看 shadowsocks 的帮助文档。

## 配置自启动

新建启动脚本文件`/etc/systemd/system/shadowsocks.service`，内容如下：

```
[Unit]
Description=Shadowsocks
After=network.target auditd.service

[Service]
Type=forking
TimeoutStartSec=0
ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks.json --pid-file /var/run/shadowsocks.pid -d start
ExecStop=/usr/local/bin/ssserver -c /etc/shadowsocks.json  --pid-file /var/run/shadowsocks.pid -d stop
PIDFile=/var/run/shadowsocks.pid
Restart=always
RestartSec=4


[Install]
WantedBy=multi-user.target
```

执行以下命令启动 shadowsocks 服务：

```bash
systemctl enable shadowsocks
systemctl start shadowsocks
```

为了检查 shadowsocks 服务是否已成功启动，可以执行以下命令查看服务的状态：

```bash
systemctl status shadowsocks -l
```

如果服务启动成功，则控制台显示的信息可能类似这样：

```
● shadowsocks.service - Shadowsocks
   Loaded: loaded (/etc/systemd/system/shadowsocks.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2015-12-21 23:51:48 CST; 11min ago
 Main PID: 19334 (ssserver)
   CGroup: /system.slice/shadowsocks.service
           └─19334 /usr/bin/python /usr/bin/ssserver -c /etc/shadowsocks.json

Dec 21 23:51:48 morning.work systemd[1]: Started Shadowsocks.
Dec 21 23:51:48 morning.work systemd[1]: Starting Shadowsocks...
Dec 21 23:51:48 morning.work ssserver[19334]: INFO: loading config from /etc/shadowsocks.json
Dec 21 23:51:48 morning.work ssserver[19334]: 2015-12-21 23:51:48 INFO     loading libcrypto from libcrypto.so.10
Dec 21 23:51:48 morning.work ssserver[19334]: 2015-12-21 23:51:48 INFO     starting server at 0.0.0.0:8388
```



# 错误

```
AttributeError: /lib64/libcrypto.so.1.1: undefined symbol: EVP_CIPHER_CTX_cleanup
```

以前在openssl，有`EVP_CIPHER_CTX_cleanup`函数.1.1.0版本中替换成为`EVP_CIPHER_CTX_reset`

解决办法：找到报错的文件(注意:根据你的python版本修改,看报错信息中使用的openssl文件)

`vim /usr/local/lib/python2.7/dist-packages/shadowsocks/crypto/openssl.py`

全文搜索cleanup将所有`EVP_CIPHER_CTX_cleanup`替换成为`EVP_CIPHER_CTX_reset`

```
:%s/cleanup/reset/

:x
```

