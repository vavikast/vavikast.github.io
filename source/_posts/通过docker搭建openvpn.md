---
title: 通过docker搭建openvpn
catalog: true
date: 2019-07-09 13:32:03
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- openvpn
- docker
- easyrsa
catagories:
- openvpn
- docker
updateDate: 2019-07-09 13:32:03
---

通过docker搭建openvpn

##### 1、安装docker

    yum install yum-utils device-mapper-persistent-data lvm2 -y
    curl -L https://download.docker.com/linux/centos/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo
    sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo
    yum makecache fast

##### 2、启动docker

    yum install docker-ce -y
    systemctl enable  docker
    systemctl start docker

##### 3、拉取openvpn镜像

    docker pull kylemanna/openvpn

##### 4、创建一个目录

    mkdir -p /data/openvpn
    cat <<  EOF >> ~/.bashrc
    export OVPN_DATA="/data/openvpn"
    EOF
    source ~/.bashrc

##### 5、生成配置文件和私钥文件

    docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://{公网IP地址}
    docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
    	输入私钥密码（输入时是看不见的）：
    	Enter PEM pass phrase:12345678
    	再输入一遍
    	Verifying - Enter PEM pass phrase:12345678
    	输入一个CA名称（我这里直接回车）
    	Common Name (eg: your user, host, or server name) [Easy-RSA CA]:
    	输入刚才设置的私钥密码（输入完成后会再让输入一次）
    	Enter pass phrase for /etc/openvpn/pki/private/ca.key:12345678

##### 6、生成客户端证书（这里的CLIENTNAME 改成你想要的名字）

    docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm -it kylemanna/openvpn easyrsa build-client-full CLIENTNAME nopass
       如果后面跟上nopass，则直接输入上面ca的密码，提供CA认证就行.
       如果后面省略nopassw， 则需要先输入设置客户端密码，再输入上面ca的密码，提供CA认证就行.

##### 7、导出客户端配置文件

    mkdir -p /data/openvpn/conf
    docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient CLIENTNAME > /data/openvpn/conf/CLIENTNAME.ovpn

##### 8.重新启动docker服务

    systemctl restart docker

##### 9、启动OpenVPN服务

    docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn

##### 10、将登录的证书下载到本地

    yum install lrzsz -y
    sz /data/openvpn/conf/CLIENTNAME.ovpn
    sz /data/openvpn/pki/private/ca.crt

##### 参考链接

https://github.com/kylemanna/docker-openvpn
https://blog.whsir.com/post-2809.html