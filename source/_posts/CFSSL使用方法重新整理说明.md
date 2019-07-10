---
title: CFSSL使用方法重新整理说明
catalog: true
date: 2019-07-09 12:32:03
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- cfssl
- kubernetes
catagories:
- cfssl
- kubernetes
updateDate: 2019-07-09 12:32:03
---



CFSSL是CloudFlare的PKI / TLS瑞士军刀。它既是命令行工具，也是用于签名，验证和捆绑TLS证书的HTTP API服务器.

1. ​    下载安装CFSSL

   用于签名，验证和捆绑TLS证书的HTTP API工具)（master节点

   ```
   wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
   wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
   wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
   mv  cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
   mv  cfssl_linux-amd64 /usr/local/bin/cfssl
   mv  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 /usr/local/bin/cfssljson
   chmod +x  /usr/local/bin/cfssl /usr/local/bin/cfssl-certinfo /usr/local/bin/cfssljson
   ```

2. 创建CA(Certificate Authority)（master节点）

   ```
   cat > ca-config.json <<EOF
   {
     "signing": {
   	"default": {
   	  "expiry": "87600h"
   	},
   	"profiles": {
   	  "kubernetes": {
   		"usages": [
   			"signing",
   			"key encipherment",
   			"server auth",
   			"client auth"
   		],
   		"expiry": "87600h"
   	  }
   	}
     }
   }
   EOF
   
   		
   知识点：
   	ca-config.json：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；此实例只有一个kubernetes模板。
   	signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；
   	server auth：表示client可以用该 CA 对server提供的证书进行验证；
   	client auth：表示server可以用该CA对client提供的证书进行验证；
   	注意标点符号，最后一个字段一般是没有都好的。
   ```

3. 创建证书请求

   ```
   	cat > ca-csr.json <<EOF
   	{
   	  "CN": "kubernetes",    
   	  "key": {
   		"algo": "rsa",
   		"size": 2048
   	  },
   	  "names": [
   		{
   		  "C": "CN",
   		  "ST": "GuangDong",
   		  "L": "ShenZhen",
   		  "O": "k8s",
   		  "OU": "System"
   		}
   	  ],
   		"ca": {
   		   "expiry": "87600h"
   		}
   	}	
   	EOF
   
   知识点：
   	"CN"：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)
   	"O"：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)
   ```

4.  生成CA证书和私钥

   ```
   cfssl gencert -initca ca-csr.json | cfssljson -bare ca
   将会生成 ca-key.pem（私钥）  ca.pem（公钥）
   知识点： cfssljson只是整理json格式，-bare主要的意义在于命名 （个人见解，以便理解，勿喷）
   ```

5. 创建kubernetes证书请求文件

   ```
   cat > kubernetes-csr.json <<EOF
   {
   	"CN": "kubernetes",
   	"hosts": [
   	  "127.0.0.1",
   	  "10.192.44.129",
   	  "10.192.44.128",
   	  "10.192.44.126",
   	  "10.192.44.127",
   	  "10.254.0.1",
   	  "*.kubernetes.master",
   	  "localhost",
   	  "kubernetes",
   	  "kubernetes.default",
   	  "kubernetes.default.svc",
   	  "kubernetes.default.svc.cluster",
   	  "kubernetes.default.svc.cluster.local"
   	],
   	"key": {
   		"algo": "rsa",
   		"size": 2048
   	},
   	"names": [
   		{
   			"C": "CN",
   			"ST": "GuangDong",
   			"L": "ShenZhen",
   			"O": "k8s",
   			"OU": "System"
   		}
   	]
   }
   EOF
   知识点：
   	这个证书目前专属于 apiserver加了一个 *.kubernetes.master 域名以便内部私有 DNS 解析使用(可删除)；至于很多人问过 kubernetes 这几个能不能删掉，答案是不可以的；因为当集群创建好后，default namespace 下会创建一个叫 kubenretes 的 svc，有一些组件会直接连接这个 svc 来跟 api 通讯的，证书如果不包含可能会出现无法连接的情况；其他几个 kubernetes 开头的域名作用相同
   				hosts包含的是授权范围，不在此范围的的节点或者服务使用此证书就会报证书不匹配错误。
   				10.254.0.1是指kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP。
   	            #hosts配置区域，即一个证书的网站可以是*.youku.com也是可以是*.google.com
   ```

6. kubernetes证书和私钥

   ```
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes	
      知识点： -config 引用的是模板中的默认配置文件，-profiles是指定特定的使用场景，比如ca-config.json中的kubernetes区域
   ```

7. 创建admin证书

   ```
   cat > admin-csr.json <<EOF
   {
     "CN": "admin",
     "hosts": [],
     "key": {
   	"algo": "rsa",
   	"size": 2048
     },
     "names": [
   	{
   	  "C": "CN",
   	  "ST": "GuangDong",
   	  "L": "ShenZhen",
   	  "O": "system:masters",
   	  "OU": "System"
   	}
     ]
   }
   EOF
   
   ```

8. 生成admin证书和私钥

   ```
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
   知识点：
      这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group
   ```
