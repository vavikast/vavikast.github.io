---
title: kubernetes v1.14.3 HA集群安装
catalog: true
date: 2019-07-09 15:31:58
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- etcd
- ha
- kubernetes
catagories:
- etcd
- ha
- kubernetes
updateDate: 2019-07-09 15:31:58
---

## kubernetes v1.14.3 HA集群安装

### 目录结构

1. ##### 集群规划

   | 主机名    | ip                 | 角色                  | 组件                                                         |
   | --------- | ------------------ | --------------------- | ------------------------------------------------------------ |
   | master1-3 | 192.168.14.138-140 | master+etcd           | etcd kube-apiserver kube-controller-manager kubectl kubeadm kubelet kube-proxy flannel |
   | worker1   | 192.168.14.141     | node                  | kubectl kubeadm kubelet kube-proxy flannel                   |
   | vip       | 192.168.14.142     | 实现apiserver的高可用 |                                                              |

2. ##### 组件版本

   | 组件                 | 版本                  |
   | -------------------- | --------------------- |
   | centos               | 7.3.1611              |
   | kernel               | 3.10.0-957.el7.x86_64 |
   | kubeadm              | v1.14.3               |
   | kubelet              | v1.14.3               |
   | kubectl              | v1.14.3               |
   | kube-proxy           | v1.14.3               |
   | flannel              | v0.11.0               |
   | etcd                 | 3.3.10                |
   | docker               | 18.09.5               |
   | kubernetes-dashboard | v1.10.1               |
   | keepalived           | 1.3.5                 |
   | haproxy              | 1.5.18                |

3. 高可用架构说明

   ![Image result for kubeadm-ha](https://user-images.githubusercontent.com/15189954/33195427-4cd2deb2-d112-11e7-8a8f-5a0fc047417e.png)

   **kubernetes架构概念**

   kube-apiserver：集群核心，集群API接口、集群各个组件通信的中枢；集群安全控制；
   etcd：集群的数据中心，用于存放集群的配置以及状态信息，通过RAFT同步信息。
   kube-scheduler：集群Pod的调度中心；默认kubeadm安装情况下–leader-elect参数已经设置为true，保证master集群中只有一个kube-scheduler处于活跃状态；
   kube-controller-manager：集群状态管理器，当集群状态与期望不同时，kcm会努力让集群恢复期望状态，比如：当一个pod死掉，kcm会努力新建一个pod来恢复对应replicas set期望的状态；默认kubeadm安装情况下–leader-elect参数已经设置为true，保证master集群中只有一个kube-controller-manager处于活跃状态；
   kubelet: kubernetes node上的 agent，负责与node上的docker engine打交道；
   kube-proxy: 每个node上一个，负责service vip到endpoint pod的流量转发，原理是通过设置iptables规则实现

   **负载均衡说明**

   haproxy： 主要用于apiserver负载均衡
   keepalived： 主要用于apiserver高可用。
   `haproxy+keepalived`主要功能就是实现高可用状态的负载均衡。首先通过keepalived生成一个虚地址VIP（主机点宕机后漂移到其他机器，VIP在哪台机器上，就和本地ip地址一样，都代表本机，共用同网卡，共用本地服务service，本地接口socket），然后当访问VIP:PORT再通过haproxy负载至后端的实际端口RIP:6443，即真正的apiserver服务.

4. 准备工作

   1、安装docker

   ```
   [参考](https://blog.51cto.com/9406836/2314122)
   ```

   2、修改内核参数

   ```
   cat <<EOF >  /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward=1
   EOF
   sysctl --system
   ```

   3、关闭Swap

   ```
   sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
   setenforce 0
   ```

   4、开启ipvs

   ```
   需要开启的模块是
   ip_vs
   ip_vs_rr
   ip_vs_wrr
   ip_vs_sh
   nf_conntrack_ipv4
   
   检查有没有开启
   cut -f1 -d " "  /proc/modules | grep -e ip_vs -e nf_conntrack_ipv4
   
   没有的话，使用以下命令加载
   modprobe -- ip_vs
   modprobe -- ip_vs_rr
   modprobe -- ip_vs_wrr
   modprobe -- ip_vs_sh
   modprobe -- nf_conntrack_ipv4
   ```

   5、禁用selinux，关闭防火墙

   ```
   关闭selinux
   setenforce 0
   sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
   关闭防火墙
   systemctl stop firewall.service
   systemctl disable firewall.service
   ```

   6、开启免密，检查网络，dns，hosts，ntp是否正常

   ```
   开启免密
   [master1]ssh-kengen 
   [master1]ssh-copy-id root@hosts
   编辑hosts
   vim /etc/hosts
   开启ntp同步
   systemctl start ntpd.service
   systemctl enable ntpd.service
   ```

5. ##### 安装步骤

   一、安装haproxy和keepalived来创建一个负载均衡器。

   1、安装haproxy

```
   分发安装haproxy（所有master节点）
   for i in master1 master2 master3; do ssh  root@$i  "yum install haproxy ";done
   配置haproxy文件（所有master节点）
   cat <EOF > /etc/haproxy/haproxy.cfg
   global
       log         127.0.0.1 local2
       chroot      /var/lib/haproxy
       pidfile     /var/run/haproxy.pid
       maxconn     4000
       user        haproxy
       group       haproxy
       daemon
   
   defaults
       mode                    tcp
       log                     global
       retries                 3
       timeout connect         10s
       timeout client          1m
       timeout server          1m
   
   frontend kubernetes
       bind *:8443              #配置端口为8443
       mode tcp
       default_backend kubernetes-master
   
   backend kubernetes-master           #后端服务器，也就是说访问192.168.255.140:8443会将请求转发到后端的三台，这样就实现了负载均衡
       balance roundrobin
       server master1  192.168.14.138:6443 check maxconn 2000
       server master2  192.168.14.139:6443 check maxconn 2000
       server master3  192.168.14.140:6443 check maxconn 2000
   EOF
   分发配置文件
   for i in master1 master2 master3; do scp  /etc/haproxy/haproxy.cfg   root@$i:/etc/haproxy/haproxy.cfg;done
   重启服务器
   systemctl daemon-reload && systemctl restart haproxy && systemctl haproxy
```

```
2、安装keepalived
```

```
   分发安装haproxy（所有master节点）
   for i in master1 master2 master3; do ssh  root@$i  "yum install keepalived ";done
   配置keepalived文件（所有master节点）
   
   cat <EOF > /etc/keepalived/keepalived.conf
   global_defs {
      notification_email {
        acassen@firewall.loc
        failover@firewall.loc
        sysadmin@firewall.loc
      }
      notification_email_from Alexandre.Cassen@firewall.loc #表示发送通知邮件时邮件源地址是谁
      smtp_server 192.168.200.1 #表示发送email时使用的smtp服务器地址
      smtp_connect_timeout 30 #连接smtp连接超时时间
      router_id LVS_docker1  #主机标识,每台机器上修改
      vrrp_skip_check_adv_addr
      vrrp_strict
      vrrp_garp_interval 0
      vrrp_gna_interval 0
   }
   
   vrrp_instance VI_1 {
       state MASTER         #备服务器上改为BACKUP
       interface ens160       #改为自己的接口
       virtual_router_id 51   # VRID 不可更改
       priority 100         #备服务器上改为小于100的数字，90,80
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass 1111
       }
       virtual_ipaddress {
           192.168.14.142          #虚拟vip，自己设定
       }
   }
   EOF
   keepalived配置不太一样，要求修改后分发。
   重启服务器
   systemctl daemon-reload && systemctl restart keepalived && systemctl keepalived
```

```
3、验证keepalived是否正常工作。
```

```
   登录主节点
   ip a
```

   二、安装 kubeadm, kubelet 和 kubectl

   1、添加国内yum源

```
   cat <<EOF > /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
```

   2、指定版本安装

```
   更新仓库
   yum makecache fast
   查找相应版本软件
   yum search --showduplicates kubeadm
```

   3、在所有安装kubelet的节点上，将kubelet设置为开机启动

```
   yum install  -y kubectl-1.14.3  kubeadm-1.14.3 kubelet-1.14.3
   systemctl daemon-reload && systemctl restart kubelet && systemctl kubelet 
```

   四、提前下载所有需要用到的镜像（谷歌镜像，通过代理下载）

   1、查看需要下载的镜像

```
   kubeadm config images list
```

   2、通过代理转换镜像(命令行直接输入)（所有节点）

```
   for i  in kube-apiserver:v1.14.3 kube-controller-manager:v1.14.3  kube-scheduler:v1.14.3 kube-proxy:v1.14.3 pause:3.1 etcd:3.3.10 coredns:1.3.1 ;do docker pull gcr.akscn.io/google_containers/$i ;docker tag gcr.akscn.io/google_containers/$i  k8s.gcr.io/$i; done
  
 国内镜像参考
 https://kubernetes.feisky.xyz/fu-lu/mirrors
```

   五、初始化master

   1、初始化master，所有master算作控制平面节点

   1） 在master1编写集群的的初始化配置文件

   	集群版本是v1.14.3

```
    指定集群的api端口和地址

    使用flannel，指定pod端口范围"10.244.0.0/16"

模板文件生成参考：
```

```
	kubeadm config print init-defaults --component-configs KubeletConfiguration
```

```
   说明： 
```

```
apiServer:
  timeoutForControlPlane: 4m0s #超时时间
apiVersion: kubeadm.k8s.io/v1beta1 #版本信息
certificatesDir: /etc/kubernetes/pki #证书位置
clusterName: kubernetes #集群名称
controlPlaneEndpoint: ""  #控制平面LOAD_BALANCER_DNS:LOAD_BALANCER_PORT
controllerManager: {}
dns:
  type: CoreDNS  #dns类型
etcd:
  local:
    dataDir: /var/lib/etcd  #etcd位置
imageRepository: k8s.gcr.io #镜像仓库
kind: ClusterConfiguration  #类型是集群配置
kubernetesVersion: v1.14.0  # kubernetes版本
networking:
  dnsDomain: cluster.local #dns
  podSubnet: ""  #子网类型，和插件有关
  serviceSubnet: 10.96.0.0/12  #service地址设定
scheduler: {}
```

```
   最终配置文件
```

```
   cat <EOF >  kubeadm-config.yaml
   apiVersion: kubeadm.k8s.io/v1beta1
   kind: ClusterConfiguration
   kubernetesVersion: v1.14.3
   controlPlaneEndpoint: "192.168.14.142:8443"
   networking:
     podSubnet: "10.244.0.0/16"
   EOF
```

   2） 执行初始化

```
   kubeadm init  --config=kubeadm-config.yaml --experimental-upload-certs
   #--pod-network-cidr=10.244.0.0/16 设置flannel子网地址访问
   # --experimental-upload-certs标志用于将应在所有控制平面实例之间共享的证书上载到群集，主控制平面的证书被加密并在上传的kubeadm-certs sercert
```

   3） 初始化完成后的主要提示如下

```
   其他的master节点加入控制平面的命令
   kubeadm join 192.168.14.142:8443 --token g9up4e.wr2zvn1y0ld2u9c8     --discovery-token-ca-cert-hash sha256:d065d5c2bfce0f5e0f784ed18cb0989dd19542721969c12888f04496b03f121c     --experimental-control-plane --certificate-key ddad01f830084f0dd4a9f89e914020cf1001aa31f4550cf5fccce9bad2d6d599
   node节点加入node的命令
   kubeadm join 192.168.14.131:6443 --token xl9aa5.2lxz30aupuf9lbhh     --discovery-token-ca-cert-hash sha256:0fa135166d86ad9e654f7b92074b34a42a5a25152c05e9253df62af8541c7bad
   配置kubeconfig的命令提示
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   
```

   4）配置kubectl，主要是配置访问apiserver 的权限 

```
      mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

   5）kubectl命令补选

```
 echo "source <(kubectl completion bash)"  >>  ~/.bashrc
 source ~/.bashrc
```

```
 2、初始化控制平面其他master 主机
```

```
   其他的master节点加入控制平面的命令
   kubeadm join 192.168.14.142:8443 --token g9up4e.wr2zvn1y0ld2u9c8     --discovery-token-ca-cert-hash sha256:d065d5c2bfce0f5e0f784ed18cb0989dd19542721969c12888f04496b03f121c     --experimental-control-plane --certificate-key 
```

```
3、集群证书和hash值，2个小小时后过期。重新生成证书方法如下
```

```
  控制平面生成证书的解密密钥
  sudo kubeadm init phase upload-certs --experimental-upload-certs
  集群重新生成token
  kubeadm token create
  集群重新生成hash值
  openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
  控制平面重新加入集群的方法
  kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash> --experimental-control-plane --certificate-key cert
  work节点重新加入集群的方法
  kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

   4、检查三台master是否初始化成功

```
   kubectl get nodes 

```

#####    6、 将worker节点加入集群

```
   kubeadm join 192.168.14.131:6443 --token xl9aa5.2lxz30aupuf9lbhh     --discovery-token-ca-cert-hash sha256:0fa135166d86ad9e654f7b92074b34a42a5a25152c05e9253df62af8541c7bad

```

#####  7、安装网络插件Flannel(主节点运行即可)

```
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/62e44c867a2846fefb68bd5f178daf4da3095ccb/Documentation/kube-flannel.yml

```

##### 8、查看集群状态

```
 master1   Ready      master   4h41m   v1.14.3
master2   Ready      <none>   4h36m   v1.14.3
master3   Ready      master   4h50m   v1.14.3
worker1   Ready      master   4h39m   v1.14.3

```

参考链接：

[kubeadm高可用安装kubernetes v1.14.3](https://blog.csdn.net/fanren224/article/details/86573264)

[通过kubeadm创建单个集群](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

[通过kubeadm创建高可用集群](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

[通过kubeadm配置集群中的每个kubelet](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/)

后续继续补充，mark！