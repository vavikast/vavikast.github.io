---
title: kubernetest master 节点恢复灾备恢复操作指南
catalog: true
date: 2019-07-09 12:45:03
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- etcd
- kubernetes
catagories:
- etcd
- kubernetes
updateDate: 2019-07-09 12:45:03
---

## kubernetest master 节点恢复灾备恢复操作指南

本文基本转载别人文章，文末会标明出处。

### 1. 基本说明

	本文档简述了Kubernetes主节点灾备恢复的相关步骤，供在发生k8s master崩溃时操作。k8s里部署了etcd群集, 主节点控制组件的高可用节点，灾备恢复也是必须要实现的操作，才能形成完备的企业级服务方案。K8s集群在master节点发生故障时，并不会影响已有的pod运行和服务开放，所以对服务是没有影响的。故而我们可以在发生故障之后，挑选合适的时间窗口进行维护和恢复，可以对外部客户造成最低的影响。
	
	文档参考了国外比较正规的作法，形成了每天自动备份的机制。主要参考URL：[Backup and Restore a Kubernetes Master with Kubeadm](https://labs.consol.de/kubernetes/2018/05/25/kubeadm-backup.html)

### 2. Etcd基本概念

	Etcd使用的是raft一致性算法来实现的，是一款分布式的一致性KV存储，主要用于共享配置和服务发现。关于raft一致性算法请参考[该动画演示](http://thesecretlivesofdata.com/raft/)。
	
	关于Etcd的原理解析请参考[Etcd 架构与实现解析](http://jolestar.com/etcd-architecture/)。
	
	etcdctl的V3版本与V2版本使用命令有所不同，etcd2和etcd3是不兼容的。**使用etcdctlv3的版本时，需设置环境变量ETCDCTL_API=3**。kubernetes默认是使用V3版本。


​	

### 3. Etcd数据备份及恢复

   etcd的数据默认会存放在我们的命令工作目录中，我们发现数据所在的目录，会被分为两个文件夹中：

- *snap:* *存放快照数据,etcd防止WAL文件过多而设置的快照，存储etcd数据状态。*
- *wal:* *存放预写式日志,最大的作用是记录了整个数据变化的全部历程。在etcd中，所有数据的修改在提交前，都要先写入到WAL中。*

#### 3.1 单节点etcd数据备份

       我们使用的是v1.14.3版本，为了部署方便和兼容，使用了k8s安装时本身的images作为运行容器(k8s.gcr.io/etcd:3.3.10)。使用以下yaml文件，运行在k8s的master上，即每天备份etcd的数据了。
    
    首先查看集群中etcd的信息，顺便通过提取etcd pod中的，作为我们执行命令的参考。

```
kubectl get pods -n kube-system
kubectl describe pods -n kube-system  etcd-docker1
找出liveness中的命令
ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

**编写etcd-backup.yaml**

```
apiVersion: batch/v1beta1
kind: CronJob   #cronjob方式
metadata:
  name: etcdbackup
  namespace: kube-system
spec:
  schedule: "0 0 * * *"   #每天备份数据
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            # Same image as in /etc/kubernetes/manifests/etcd.yaml
            image: k8s.gcr.io/etcd:3.3.10 #默认镜像
            env:
            - name: ETCDCTL_API
              value: "3"
            command: ["/bin/sh"]
            args: ["-c", "etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d_%H:%M:%S_%Z).db"]
            volumeMounts:
            - mountPath: /etc/kubernetes/pki/etcd #将服务器证书位置映射进来
              name: etcd-certs
              readOnly: true
            - mountPath: /backup #证书备份位置映射进来
              name: backup
          restartPolicy: OnFailure
          nodeSelector:
            node-role.kubernetes.io/master: ""  #节点选择为主节点
          tolerations:
          - key: "node-role.kubernetes.io/master" #打taints，为了使pod能够容忍master
            effect: "NoSchedule"
          hostNetwork: true
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd #挂载主机证书文件位置
              type: DirectoryOrCreate
          - name: backup
            hostPath:
              path: /tmp/etcd_backup/  #备份文件位置
              type: DirectoryOrCreate
```



	从上面的yaml文件中，我们可以看到其实现思路：

1. 定义为CronJob，这个pod每天凌晨会自动运行(schedule: "0 0 * * *")。
2. 此pod是运行在master上的(nodeSelector + tolerations 实现)。
3. 挂载了master机器上的/tmp/etcd_backup/作为备份目录，这个目录生产环境最好挂载或及时cp到其它机器，防止机器本身的意外情况。
4. 传进的参数为ETCDCTL_API版本3的命令进行备份。Args参数中的$(date +%Y-%m-%d_%H:%M:%S_%Z).db"即为备份命令。它按照时间的格式命名etcd的备份数据。

 

#### 3.2 单节点etcd数据恢复

    **如果已有备份数据，在只有etcd数据损坏的下，恢复备份如下。主要考虑的顺序：停止kube-apiserver，停止ETCD，恢复数据，启动ETCD，启动kube-apiserver。**

1.将/etc/kubernetes/manifests/ kube-apiserver.yaml文件里的镜像版本更改，停止kube-api server服务。

2.将/etc/kubernetes/manifests/ etcd.yaml文件里的镜像版本更改，停止etcd server服务。

3.运行如下命令，将损坏的数据文件移至其它地方。

```
mv /var/lib/etcd/\* /tmp/
```

1. 运行以下命令，以临时docker运行的方式，将数据从备份里恢复到/var/lib/etcd/。

```
docker run --rm \
-v '/tmp:/backup' \
-v '/var/lib/etcd:/var/lib/etcd' \
--env ETCDCTL_API=3 \
'k8s.gcr.io/etcd:3.3.10' \
/bin/sh -c "etcdctl snapshot restore '/backup/etcd-snapshot-xxx_UTC.db' ; mv /default.etcd/member/ /var/lib/etcd/"
```

1. 改回/etc/kubernetes/manifests/kube-apiserver.yaml文件里的镜像版本，恢复etcd server服务。

2. 改回/etc/kubernetes/manifests/etcd.yaml文件里的镜像版本，恢复kube-api server服务。

#### 3.3 集群etcd节点控制组件备份

一般来说，如果master节点需要备份恢复，那除了误操作和删除，很可能就是整个机器已出现了故障，故而可能需要同时进行etcd数据的恢复。

而在恢复时，有个前提条件，**机器名称和ip地址需要与崩溃前的主节点配置完成一样**，因为这个配置是写进了etcd数据存储当中的。

因为高可用集群中的etcd的文件是一样的，所以我们备份一份文件即可。

**具体备份方法参考3.1**

#### 3.4 etcd节点控制组件的恢复

首先需要分别停掉三台Master机器的kube-apiserver，确保kube-apiserver已经停止了。

```bash
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak
docker ps|grep k8s_  # 查看etcd、api是否up，等待全部停止
mv /var/lib/etcd /var/lib/etcd.bak
```

etcd集群用同一份snapshot恢复。

```
# 准备恢复文件
cd /tmp
tar -jxvf /data/backup/kubernetes/2018-09-18-k8s-snapshot.tar.bz
rsync -avz 2018-09-18-k8s-snapshot.db 192.168.105.93:/tmp/
rsync -avz 2018-09-18-k8s-snapshot.db 192.168.105.94:/tmp/
```

在lab1上执行：

```bash
cd /tmp/
export ETCDCTL_API=3
etcdctl snapshot restore 2018-09-18-k8s-snapshot.db \
    --endpoints=192.168.105.92:2379 \
    --name=lab1 \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --initial-advertise-peer-urls=https://192.168.105.92:2380 \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster=lab1=https://192.168.105.92:2380,lab2=https://192.168.105.93:2380,lab3=https://192.168.105.94:2380 \
    --data-dir=/var/lib/etcd
```

在lab2上执行：

```bash
cd /tmp/
export ETCDCTL_API=3
etcdctl snapshot restore 2018-09-18-k8s-snapshot.db \
    --endpoints=192.168.105.93:2379 \
    --name=lab2 \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --initial-advertise-peer-urls=https://192.168.105.93:2380 \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster=lab1=https://192.168.105.92:2380,lab2=https://192.168.105.93:2380,lab3=https://192.168.105.94:2380 \
    --data-dir=/var/lib/etcd
```

在lab3上执行：

```bash
cd /tmp/
export ETCDCTL_API=3
etcdctl snapshot restore 2018-09-18-k8s-snapshot.db \
    --endpoints=192.168.105.94:2379 \
    --name=lab3 \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --initial-advertise-peer-urls=https://192.168.105.94:2380 \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster=lab1=https://192.168.105.92:2380,lab2=https://192.168.105.93:2380,lab3=https://192.168.105.94:2380 \
    --data-dir=/var/lib/etcd
```

全部恢复完成后，三台Master机器恢复manifests。

```bash
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests
```

最后确认：

```
# 再次查看key
[root@lab1 kubernetes]# etcdctl get / --prefix --keys-only --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt
registry/apiextensions.k8s.io/customresourcedefinitions/apprepositories.kubeapps.com

/registry/apiregistration.k8s.io/apiservices/v1.

/registry/apiregistration.k8s.io/apiservices/v1.apps

/registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io

           ........此处省略..........
[root@lab1 kubernetes]# kubectl get pod -n kube-system
NAME                                              READY     STATUS    RESTARTS   AGE
coredns-777d78ff6f-m5chm                          1/1       Running   1          18h
coredns-777d78ff6f-xm7q8                          1/1       Running   1          18h
dashboard-kubernetes-dashboard-7cfc6c7bf5-hr96q   1/1       Running   0          13h
dashboard-kubernetes-dashboard-7cfc6c7bf5-x9p7j   1/1       Running   0          13h
etcd-lab1                                         1/1       Running   0          18h
etcd-lab2                                         1/1       Running   0          1m
etcd-lab3                                         1/1       Running   0          18h
kube-apiserver-lab1                               1/1       Running   0          18h
kube-apiserver-lab2                               1/1       Running   0          1m
kube-apiserver-lab3                               1/1       Running   0          18h
kube-controller-manager-lab1                      1/1       Running   0          18h
kube-controller-manager-lab2                      1/1       Running   0          1m
kube-controller-manager-lab3                      1/1       Running   0          18h
kube-flannel-ds-7w6rl                             1/1       Running   2          18h
kube-flannel-ds-b9pkf                             1/1       Running   2          18h
kube-flannel-ds-fck8t                             1/1       Running   1          18h
kube-flannel-ds-kklxs                             1/1       Running   1          18h
kube-flannel-ds-lxxx9                             1/1       Running   2          18h
kube-flannel-ds-q7lpg                             1/1       Running   1          18h
kube-flannel-ds-tlqqn                             1/1       Running   1          18h
kube-proxy-85j7g                                  1/1       Running   1          18h
kube-proxy-gdvkk                                  1/1       Running   1          18h
kube-proxy-jw5gh                                  1/1       Running   1          18h
kube-proxy-pgfxf                                  1/1       Running   1          18h
kube-proxy-qx62g                                  1/1       Running   1          18h
kube-proxy-rlbdb                                  1/1       Running   1          18h
kube-proxy-whhcv                                  1/1       Running   1          18h
kube-scheduler-lab1                               1/1       Running   0          18h
kube-scheduler-lab2                               1/1       Running   0          1m
kube-scheduler-lab3                               1/1       Running   0          18h
kubernetes-dashboard-754f4d5f69-7npk5             1/1       Running   0          13h
kubernetes-dashboard-754f4d5f69-whtg9             1/1       Running   0          13h
tiller-deploy-98f7f7564-59hcs                     1/1       Running   0          13h
```

进相应的安装程序确认，数据全部正常。

### 4.Etcd最后说明

	1.本文是综合了很多篇文章，整理而成，最后会放参考链接！
	
	2.单点etcd数据恢复的验证没有问题，但是恢复命令可以优化
	
	3.集群etcd恢复没有问题，因为集群中存在--initial-cluster-token=etcd-cluster-0所以没有写，是可以恢复的。
	
	    4.时间有限，有空继续优化编辑。mark下！

### 5.参考链接

[阿里云上kubernetes的备份和恢复](https://yq.aliyun.com/articles/336781)

[kubeadm安装etcd备份恢复](https://blog.51cto.com/ygqygq2/2176492)

[Kubernetes Master节点灾备恢复操作指南](https://www.cnblogs.com/aguncn/p/9983603.html)

[etcd解析](https://jimmysong.io/kubernetes-handbook/concepts/etcd.html)