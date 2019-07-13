---
title: Prometheus配置钉钉告警
catalog: true
date: 2019-07-13 10:46:09
subtitle:
header-img:  "/img/article_header/article_header.png"
tags:
- prometheus
updateDate: 2019-07-13 10:46:09
---
### Prometheus配置钉钉告警

1. ##### 获取钉钉token

2. ##### 配置钉钉webhook

   ​       钉钉通过机器人提供了一个webhook接口，但是呢钉钉机器人对文件格式有严格要求，所以必须通过特定的格式转换，才能发送给你钉钉的机器人。有人已经写了转换插件，我是个拿来主义者（主要是自己不会搞，先跑起来再说），那就直接用吧。

   1. 命令行方式

      - 安装go语言

        ```
        wget -c https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz
        tar -C /usr/local/ -zxvf go1.8.3.linux-amd64.tar.gz 
        mkdir -p /home/gocode
        cat << EOF >> /etc/profile
        export GOROOT=/usr/local/go #设置为go安装的路径
        export GOPATH=/home/gocode  #默认安装包的路径
        export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
        EOF
        source  /etc/profile
        ```

      - 编译钉钉插件

        ```
        cd /home/gocode/
        mkdir -p src/github.com/timonwong/
        cd /home/gocode/src/github.com/timonwong/
        git clone  https://github.com/timonwong/prometheus-webhook-dingtalk.git
        make
        ln -s  /home/gocode/src/github.com/timonwong/prometheus-webhook-dingtalk/prometheus-webhook-dingtalk /usr/local/bin/prometheus-webhook-dingtalk
        ```

      - 启动插件

        ```
        nohup prometheus-webhook-dingtalk --ding.profile="webhook1=https://oapi.dingtalk.com/robot/send?access_token=xxxxx" &
        ```

      - 编译alertmanager配置文件

        ```
        global:
          resolve_timeout: 5m
          smtp_smarthost: 'smtp.qiye.aliyun.com:465'
          smtp_from: 'sxxx@yy.com.com'
          smtp_auth_username: 'sxxx@yy.com.com'
          smtp_auth_password: 'xxx'
          smtp_require_tls: false
        route:
          group_by: [cluster,]
          group_wait: 10s
          group_interval: 10s
          repeat_interval: 1h
          receiver: 'web.hook'
        receivers:
        - name: 'web.hook'
          email_configs:
          - to: 'sxxx@yy.com.com'
          webhook_configs:
          - url: 'http://localhost:8060/dingtalk/webhook1/send'
            send_resolved: false
        ```

        

      - 结果截图

        ![钉钉截图](https://cdn.sinaimg.cn.52ecy.cn/large/005BYqpgly1g4xyv4mrbrj30a70fxmy0.jpg)

      - 遇到问题说明

        ```
        1.如果编译出错，可能是go版本问题。
        2. 原始代码go编译好像写死了目录，如果出错，可以按照我写的去做。（具体不得而知，go语言不熟）
        ```

   2. docker方式

      - 直接执行docker程序

        ```
        docker run -d --restart always -p 8060:8060 timonwong/prometheus-webhook-dingtalk:master --ding.profile="webhook1=https://oapi.dingtalk.com/robot/send?access_token=XXXXXX"
        ```

      - 编译altermanager配置文件

        ```
        docker run -d --restart always -p 8060:8060 timonwong/prometheus-webhook-dingtalk:v0.3.0 --ding.profile="webhook1=https://oapi.dingtalk.com/robot/send?access_token=XXXXXX"
        ```

      - 结果截图

        ![钉钉截图](https://cdn.sinaimg.cn.52ecy.cn/large/005BYqpgly1g4xyv4mrbrj30a70fxmy0.jpg)

      - 遇到说明

        ```
        1.不知道我报警设置的有问题，还是程序有问题，钉钉发一次就报错了还需要继续改进。
        ```

        

3. ##### 参考文档

   [将钉钉接入 Prometheus AlertManager WebHook](https://theo.im/blog/2017/10/16/release-prometheus-alertmanager-webhook-for-dingtalk/)

   [配置钉钉告警](https://blog.51cto.com/jiay1/2419221)

   [docker镜像](https://hub.docker.com/r/timonwong/prometheus-webhook-dingtalk/tags)

   [二进制程序文件](https://github.com/timonwong/prometheus-webhook-dingtalk)

   [通过webhook推送钉钉](https://blog.csdn.net/AlbertFly/article/details/83414432)
   
   [cetnos7 安装go环境](https://blog.csdn.net/xianchanghuang/article/details/82722064)

