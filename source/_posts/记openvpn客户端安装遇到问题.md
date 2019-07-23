---
title: 记openvpn客户端安装遇到问题
catalog: true
date: 2019-07-23 11:30:31
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- openvpn
catagories:
- openvpn
updateDate: 2019-07-23 11:30:31
---

#### **记openvpn客户端安装遇到问题**

因工作需要，最近在研究OPENVPN。因为它免费开源，而且使用社区活跃度较高。
目前已在windows，IMAC，IPhone，andriod成功部署使用客户端。具体使用方式不再说明，简单说明遇到的问题吧。

##### 		1.MAC中导入证书的问题。

mac系统上不支持crt证书，需要转换为pkcs12证书。

```
openssl pkcs12 -export -out ca.p12 -inkey ca.key -in ca.crt
```



##### 		2.iphone客户端中导入证书报错。

iphone无法显示的支持证书，需手动将证书添加到ovpn配置文件中。

```
#iPhone-OpenVPN客户端配置模板
client
dev tun
connect-retry-max 5
connect-retry 5
resolv-retry 60
#------------------------
remote xxx.xxx.xxx.xxx 3389 tcp-client
#------------------------------------
<http-proxy-user-pass>
kangml
kangml
</http-proxy-user-pass>
resolv-retry infinite
nobind
persist-key
persist-tun
ns-cert-type server
comp-lzo
verb 3
<ca>
ca.crt CA证书
</ca>
#----
<cert>
user01.crt 客户端的证书
</cert>
#----
<key>
user01.key  客户端的秘钥
</key>
#----
key-direction 1
<tls-auth>
ta.key      TLS-auth密钥
</tls-auth>
#----
auth-user-pass
#----

注明： 
	1.模板只是参考
	2.通过vim ca.crt 粘贴复制证书里面的文件。
	3.需要注意的是user01.key 客户端的秘钥，不需要全部复制.只需要最后面的 ----BEGIN CERTIFICATE----到----END CERTIFICATE---- 包括这两句都要一起复制。

```

##### 	3.记iphone xr系列openvpn网络无法连接。

​    在IPhone xs和IPhone xr 使用客户端时，提示"network is unavailable， please try to connect later with active network"

​    原因是openvpn没有触发IOS的授权，一直无法获取IPhone的网络授权。

​     方法： 转到主界面，点击private tunnel,点击agree，点击sign up，然后允许使用蜂窝网络和无线网络，就可以正常使用了。