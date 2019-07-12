---
title: Alertmanger中邮箱配置
catalog: true
date: 2019-07-12 15:28:24
subtitle:
header-img:"/img/article_header/article_header.png"
tags:
- prometheus
updateDate:2019-07-12 15:28:24
---

#####  alertmanger配置文件-邮箱配置问题。

```
smtp_smarthost: 'smtp.qiye.aliyun.com:465'
smtp_hello: 'company.com'
smtp_from: 'username@company.com'
smtp_auth_username: 'username@company.com'
smtp_auth_password: password
smtp_require_tls: false
```

##### 端口说明

RFC 8314要求到端口465的SMTP连接使用TLS（而不是STARTTLS）,require_tls（或全局smtp_require_tls）必须设置为false，以避免alertmanager尝试STARTTLS。
使用端口587时，STARTTLS。

##### smtp端口历史

当用户开始寻找安全电子邮件消息的方法时，该端口首先被引入。当时出现的想法是使用SSL（安全套接字层）加密消息。但在那个时候，这样做意味着使用一个独立的端口。
使用其他两个端口，一个用于明文消息，另一个用于加密消息，也可以在其他网络协议中找到，比如：
FTP - 21为明文和990加密（通过隐式SSL）；
IMAP 143加密的明文和993；
POP 110明文和995加密。
在SMTP中，为加密连接选择的端口是465。
不幸的是，465号端口从未被IETF（因特网工程任务组）认可，这个机构负责开发Internet标准，作为SMTP的正式端口。相反，IANA（互联网数字分配机构）分配给smtps（简单邮件传输协议），现在depracated确保SMTP的方法。
今天，即使使用同一端口（例如587），SMTP也可以被保护。一个明文的SMTP连接可以升级为一个安全的连接通过TLS加密（传输层安全）或SSL通过简单的执行提供STARTTLS命令，当然服务器支持。

##### 参考链接：

[SMTP几个端口比较](https://blog.csdn.net/zhangyuan12805/article/details/78781330)

[smtp：465 fail to send mail](https://github.com/prometheus/alertmanager/issues/980)

