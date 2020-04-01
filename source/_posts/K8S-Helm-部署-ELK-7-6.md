---
title: 'K8S: Helm 部署 ELK 7.6'
catalog: true
date: 2020-04-01 15:29:59
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- rook
- elk
- kubernetes
catagories:
- rook
- elk
- kubernetes
updateDate: 2020-04-01 15:29:59
---

### 场景

​		在 K8S 上部署有状态应用 ELK，收集日常测试数据的上报（应用拨测的 Heartbeat、调用链追踪的 APM、性能指标 metabeat 等）。本文通过rook提供底层存储，用于安装elk的statefulset，然后部署MetalLB实现本地负载均衡，最后通过ingress-control实现访问kibana。

### 操作步骤

- 1.安装rook
- 2.安装helm
- 3.安装ES
- 4.安装kibana
- 5.安装filebeat
- 6.安装metalLB
- 7.安装Ingress-control
- 8.访问测试

### 1.安装rook

#### 1.1 安装说明

​		Rook是专用于Cloud-Native环境的文件、块、对象存储服务。它实现了一个自我管理的、自我扩容的、自我修复的分布式存储服务。Rook支持自动部署、启动、配置、分配（provisioning）、扩容/缩容、升级、迁移、灾难恢复、监控，以及资源管理。为了实现所有这些功能，Rook依赖底层的容器编排平台。

​		Ceph是一个分布式存储系统，支持文件、块、对象存储，在生产环境中被广泛应用。

​		此次安装就是通过rook进行ceph进行部署，简化了ceph的安装管理配置。同时也是为了能够利用本地资源。提供storageclass。	

#### 1.2 rook和ceph架构

![rook-architecture-0.png](https://i.loli.net/2020/03/23/E6MqSgbHVhkCZKd.png)

![rook-architecture-2.png](https://i.loli.net/2020/03/23/I6fUqYLCJvcDWE4.png)



#### 1.2 安装ceph

git配置文件

```
git clone --single-branch --branch release-1.2 https://github.com/rook/rook.git
cd cluster/examples/kubernetes/ceph
```

安装CRD设置RBAC权限

```
kubectl apply -f common.yaml
```

安装operator，设置ceph启动的各种资源

```
kubectl apply -f operator.yaml
```

安装ceph集群

```
kubectl apply -f cluster-test.yaml
```

安装ceph管理工具

```
kubectl apply -f toolbox.yaml
```

#### 1.3 注意事项

- rook的镜像默认不是从dockerhub下载，大部分是从quay.io仓库下载，镜像拉取较慢，如果直接通过kubectl创建，那么运行时会因为下载超时而报错，所以最好提前下载好。

- 据我目前了解，rook新版已经不支持挂载本地目录提供ceph存储了。所以只能在本地挂载新磁盘了。让operator程序自动探测发现硬盘设备，并为之所用。（如果想要挂载本地目录提供ceph存储，可以通过更改operator.yml中的ROOK_HOSTPATH_REQUIRES_PRIVILEGED=true，大家可以测试下，我没试）

- rook已经使用csi方式挂载ceph，flex方式已逐渐被淘汰。但是使用csi方式有一个大坑，就是kernel>4.10才能正常使用，否则会报很多奇怪的错误。例如“rook  rpc error: code = Internal desc = an error occurred while running (790) mount -t ceph”，可参考[On the failure of Mount PVC by pod](https://github.com/rook/rook/issues/5066)

- 如果想使用flex方式挂载ceph，记得在operator.yml中更改“ROOK_ENABLE_FLEX_DRIVER=true”，同时记得创建storageclass.yml时，使用flex[目录下的配置](https://github.com/rook/rook/tree/release-1.2/cluster/examples/kubernetes/ceph/flex)。

- 升级内核可参考 [centos7升级内核](https://blog.csdn.net/kikajack/article/details/79396793)

- 本地添加磁盘扫描：

  ```
  echo "- - -" > /sys/class/scsi_host/host*/scan
  fdisk -l
  ```

- 因为ceph要求3+节点，所以我为了使用master节点，把master的污点取消了。

  ```
  kubectl taint node master node-role.kubernetes.io/master:NoSchedule-
  ```

#### 1.4 查看运行状态

查看rook状态

```
[root@docker1 ~]# kubectl get pods -n rook-ceph 
NAME                                                READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-2gb6l                              3/3     Running     0          3h18m
csi-cephfsplugin-hnwkn                              3/3     Running     0          3h18m
csi-cephfsplugin-provisioner-7b8fbf88b4-2wchx       4/4     Running     0          3h18m
csi-cephfsplugin-provisioner-7b8fbf88b4-6bjb8       4/4     Running     0          3h18m
csi-cephfsplugin-s5mln                              3/3     Running     0          3h18m
csi-rbdplugin-provisioner-6b8b4d558c-2nhrk          5/5     Running     0          3h18m
csi-rbdplugin-provisioner-6b8b4d558c-8n57n          5/5     Running     0          3h18m
csi-rbdplugin-qv5qt                                 3/3     Running     0          3h18m
csi-rbdplugin-rtsxc                                 3/3     Running     0          3h18m
csi-rbdplugin-v67hv                                 3/3     Running     0          3h18m
rook-ceph-crashcollector-docker3-6fbc6c7fc6-trmtz   1/1     Running     0          78m
rook-ceph-crashcollector-docker4-fcb9bf67c-kk7v4    1/1     Running     0          78m
rook-ceph-crashcollector-docker5-7775464c9b-dv4bh   1/1     Running     0          78m
rook-ceph-mds-myfs-a-7f4cdb685d-k9skd               1/1     Running     0          78m
rook-ceph-mds-myfs-b-5847d89857-254d9               1/1     Running     0          78m
rook-ceph-mgr-a-5b5f4588b7-bsf72                    1/1     Running     0          3h17m
rook-ceph-mon-a-965f65f76-gv2fv                     1/1     Running     0          3h17m
rook-ceph-operator-69f856fc5f-2dp9h                 1/1     Running     0          3h19m
rook-ceph-osd-0-7879445dfc-5g48g                    1/1     Running     0          3h17m
rook-ceph-osd-1-6bc695c476-v5xzt                    1/1     Running     0          3h17m
rook-ceph-osd-2-c96b7b6b7-cnhb7                     1/1     Running     0          3h16m
rook-ceph-osd-prepare-docker3-lgrwm                 0/1     Completed   0          3h17m
rook-ceph-osd-prepare-docker4-qp56x                 0/1     Completed   0          3h17m
rook-ceph-osd-prepare-docker5-q5czc                 0/1     Completed   0          3h17m
rook-ceph-tools-7d764c8647-ftxsb                    1/1     Running     0          3h15m
rook-discover-bgsf6                                 1/1     Running     0          3h18m
rook-discover-ldc7k                                 1/1     Running     0          3h18m
rook-discover-w2wb6                                 1/1     Running     0          3h18m

```

查看ceph状态

```
kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
    ceph status
    ceph osd status
    ceph df
    rados df

```



### 2. 安装helm

#### 2.1 安装说明

​		helm之于k8s，相当于yum之于centos。是k8s的包管理器。具体使用侧参考 [helm教程](https://whmzsu.github.io/helm-doc-zh-cn/quickstart/using_helm-zh_cn.html)

#### 2.2 安装helm

​	安装helmv3版本

```
wget https://get.helm.sh/helm-v3.1.2-linux-amd64.tar.gz
tar -xzvf helm-v3.1.2-linux-amd64.tar.gz
cd linux-amd64 && mv helm /usr/bin/
```

#### 2.3 安装helm仓库

添加elk提供的repo仓库

```
helm repo add elastichttps://helm.elastic.co
helm repo search elastic
```

#### 2.4 注意事项

- helmv2已经被抛弃，所以还是安装helmv3版本吧



### 3. 安装 ES

#### 3.1 安装说明

​		`Elasticsearch` 是一个实时的、分布式的可扩展的搜索引擎，允许进行全文、结构化搜索，它通常用于索引和搜索大量日志数据，也可用于搜索许多不同类型的文档。

#### 3.2 安装ES

​		helm默认安装的elastic需要太多空间，所以我把elastic的helm包下载到本地，并修改了volume的值，同时为了PVC模板能够匹配，增加了storageClass名称。

修改statefulsets中的volumeClaimTemplate. 此处的理解可参考[kubernetes 关于statefulset的理解](https://blog.51cto.com/09112012/2307767)

```
helm pull elastic/elasticsearch
tar -zxvf elasticsearch-7.6.1.tgz
cd elasticsearch
vim values.yaml 
volumeClaimTemplate:
  storageClassName: "csi-cephfs"
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 3Gi

```

安装es

```
helm install elastic ./
```

#### 3.3 查看运行状态

查看 StatefulSet

```bash
[root@docker1 elasticsearch]# kubectl get sts -owide
NAME                   READY   AGE    CONTAINERS      IMAGES
elasticsearch-master   3/3     124m   elasticsearch   docker.elastic.co/elasticsearch/elasticsearch:7.6.1
```

查看 Pod，有 3 个节点，同时 Pod 命名带有序号，和 Deployment 的随机命名 Pod 不一样；

```bash
[root@docker1 elasticsearch]# kubectl get pods 
NAME                             READY   STATUS    RESTARTS   AGE
elasticsearch-master-0           1/1     Running   0          125m
elasticsearch-master-1           1/1     Running   0          125m
elasticsearch-master-2           1/1     Running   0          125m

```

查看PVC和storageclass

```
[root@docker1 elasticsearch]# kubectl get storageclasses.storage.k8s.io 
NAME         PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
csi-cephfs   rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   137m
[root@docker1 elasticsearch]# kubectl get pvc
NAME                                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cephfs-pvc                                    Bound    pvc-4ebff26d-d2d1-4f61-9d73-9141f631f780   1Gi        RWX            csi-cephfs     121m
elasticsearch-master-elasticsearch-master-0   Bound    pvc-2cb96692-20f1-44ba-a307-3daeb4e7aded   3Gi        RWO            csi-cephfs     126m
elasticsearch-master-elasticsearch-master-1   Bound    pvc-323697ae-065a-4d1b-a9ec-3fabc66a4419   3Gi        RWO            csi-cephfs     126m
elasticsearch-master-elasticsearch-master-2   Bound    pvc-95f070f9-a4e0-4cd0-a8c9-969f69521063   3Gi        RWO            csi-cephfs     126m
```

#### 3.4 注意事项

- storageclass和pvc已经创建，但是pod无法挂载。发现kernel>4.10才能正常使用，否则会报很多奇怪的错误。例如“rook  rpc error: code = Internal desc = an error occurred while running (790) mount -t ceph”，可参考[On the failure of Mount PVC by pod](https://github.com/rook/rook/issues/5066)
- 注意根据自己的需求修改es配置文件，记得提前更改。
- 如果出现报错，多用kubectl log,kubectl  describe 查找文件所在。



### 4. 部署 Kibana

#### 4.1 安装说明

​		kibana是对elastic数据进行可视化、分析和探索。

#### 4.2 安装

​		把kibana的helm包下载到本地，并修改了volume的值。其中

```
helm pull elastic/kinaba
tar -zxvf kibana-7.6.1.tgz
cd kibana
vim values.yaml 
#此处修改了kibana的配置文件，默认位置/usr/share/kibana/kibana.yaml
kibanaConfig:
   kibana.yml: |
     server.port: 5601
     server.host: "0.0.0.0"
     elasticsearch.hosts: [ "http://elasticsearch-master:9200" ]
#开启ingress，在后面可以看出效果
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  path: /
  hosts:
    - kibana-kibana

helm install kibana ./

```

#### 4.3 查看运行状态

```
[root@docker1 kibana]# kubectl get pods -l app=kibana
NAME                             READY   STATUS    RESTARTS   AGE
kibana-kibana-6b65d94756-p4tr4   1/1     Running   0          74m

```

#### 4.4 注意事项

- 安装kibana出错，状态为READY 0/1 ,报错内容为{"type":"log","@timestamp":"2020-03-31T10:22:21Z","tags":["warning","savedobjects-service"],"pid":6,"message":"Unable to connect to Elasticsearch. Error: [resource_already_exists_exception] index [.kibana_task_manager_1/mN6of5kpT86Eyyuj9BNDIg] already exists, with { index_uuid=\"mN6of5kpT86Eyyuj9BNDIg\" & index=\".kibana_task_manager_1\" }"}，此处删除es中的.index文件即可。参考链接[kibana-task-manager error](https://discuss.elastic.co/t/kibana-task-manager-1-issues/222839) 

  ```
  #此处10.96.193.elasticsearch-master的svc地址.
  curl -X DELETE http://10.96.193.2:9200/.kibana*
  ```

  

### 5. 安装Filebeat

#### 5.1 安装说明

​	  filebeat是本地文件的日志数据采集器，可监控日志目录或特定日志文件（tail file），并将它们转发给Elasticsearch或Logstatsh进行索引、kafka等.

#### 5.2 安装filebeat

```
helm install filebeat elastic/filebeat
```

#### 5.3 查看运行状态

```
[root@docker1 ingress]# kubectl get pods -l app=filebeat-filebeat
NAME                      READY   STATUS    RESTARTS   AGE
filebeat-filebeat-6clnj   1/1     Running   0          18h
filebeat-filebeat-kpzcb   1/1     Running   0          18h
filebeat-filebeat-xpxnz   1/1     Running   0          18h
```



### 6. 部署metalLB

#### 6.1 安装说明

​		MetalLB可以为Kubernetes集群提供了网络负载平衡器实现。简而言之，它允许您在本地运行的集群中创建类型为“ LoadBalancer”的Kubernetes服务。

​		在第2层模式下，服务IP的所有流量都流向一个节点。 `kube-proxy`将流量传播到所有服务的Pod。

![image.png](https://i.loli.net/2020/04/01/ONycraKsAFLdPo9.png)

#### 6.2 安装metalLB

安装metallb

```
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.3/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

使用Layer2 类型配置config文件，增加本地可用地址范围。

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.14.161-192.168.14.165

kubectl apply  -f config.yaml
```

#### 6.3 查看运行状态

关于获取地址的信息，可以在后面ingress-service中lb的地址中看到。

```
[root@docker1 ingress]# kubectl get pods -n metallb-system 
NAME                          READY   STATUS    RESTARTS   AGE
controller-5c9894b5cd-hrhdf   1/1     Running   0          17m
speaker-68hzp                 1/1     Running   0          17m
speaker-892bv                 1/1     Running   0          17m
speaker-l9mcg                 1/1     Running   0          17m
speaker-m4pbl                 1/1     Running   0          17m
speaker-q4jrq                 1/1     Running   0          17m
speaker-qwhqm                 1/1     Running   0          17m
```

#### 6.4 注意事项

- 因为使用Layer 2模式，所以分配的地址，必须是同网段的未用地址。

- metallb需借助kube-proxy实现功能，可以选择kube-proxy实现方式，如果更改成ipvs，可参考配置 [MetalLB安装](https://metallb.universe.tf/installation/)

- externalTrafficPolicy可分为cluster和local两种方式进行选择。cluster：kube-proxy在收到流量的节点上进行负载平衡，并将流量分配到服务中的所有Pod。local：`kube-proxy`在收到流量的节点上进行负载平衡，并将流量分配到服务中的所有Pod。

- BGP模式可自行探究。

  

### 7. 部署Ingress

#### 7.1 安装说明

​	管理对集群中的服务(通常是HTTP)的外部访问的API对象。Ingress可以提供负载平衡、SSL终端和基于名称的虚拟主机。此处是搭配metallb提供地址访问内部请求。如果不搭配metallb，只使用ingress会在端口选择这个地方很麻烦。

#### 7.2 安装ingress

安装deployment

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
```

下载修改service文件，将externalTrafficPolicy=Cluster（推荐）。

```
wget  https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/cloud-generic.yaml

kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  externalTrafficPolicy: Cluster
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
    - name: https
      port: 443
      protocol: TCP
      targetPort: https

---
kubectl apply -f cloud-generic.yaml
```

#### 7.3 查看运行状态

查看ingress运行状态

```
[root@docker1 ingress]# kubectl get svc -n ingress-nginx 
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.96.25.12   192.168.14.161   80:31622/TCP,443:31187/TCP   20s

```

访问80端口进行测试

```
[root@docker1 ingress]# curl 192.168.14.161 
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.17.8</center>
</body>
</html>
```



### 8.配置ES访问

#### 8.1配置说明

​		安装kibana的时，已开启了ingress选项，所以我们可以直接看到ingress信息，如果当时不开启，我们也可以自己写一个kibana-ingress.yml 文件。

```
#配置host的名称为kibana-kibana
#配置url后缀匹配地址是/
#配置后端服务名称为kibana-kibana，端口是5601
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name:  kibana
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: kibana-kibana
    http:
      paths:
      - path: /
        backend:
          serviceName: kibana-kibana
          servicePort: 5601
kubectl apply -f kibana-ingress.yaml
```

#### 8.2 查看运行状态

```
[root@docker1 ingress]# kubectl get ingresses.
NAME            HOSTS           ADDRESS          PORTS   AGE
kibana-kibana   kibana-kibana   192.168.14.161   80      15h
```

#### 8.3 访问kibana

​		我们在本地配置hosts添加192.168.14.161 kibana-kibana（此处的kibana-kibana是ingress中host配置的名称）。然后网页直接访问http://kibana-kibana

​	    kibana没有开启xpack功能，所以无法打开monitor了。

![image.png](https://i.loli.net/2020/04/01/fxk8sXDUT91c6yZ.png)



添加filebeat索引

![image.png](https://i.loli.net/2020/04/01/HL56EbFeXV8CRfO.png)

![image.png](https://i.loli.net/2020/04/01/iny1GqvCerQ8bpz.png)

![image.png](https://i.loli.net/2020/04/01/LdYaRHwXS7U5kDx.png)

![image.png](https://i.loli.net/2020/04/01/ELAy98gtoWbYnpw.png)

查看discover

![image.png](https://i.loli.net/2020/04/01/4BXuAtDbpjGrvZh.png)



### 

### 后续	

​		只是粗略的将elk的集群搭建起来，后期慢慢整合。

​		未完待续。



##### 参考：

[K8S : Helm 部署 ELK 7.3](https://dhcp.cn/k8s/deploy/elk7.html)

[StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

[rook官方教程](https://rook.io/docs/rook/v1.2/ceph-filesystem.html)

[基于Rook的Kubernetes存储方案](https://blog.gmem.cc/rook-based-k8s-storage-solution)

[kubernetes 关于statefulset的理解](https://blog.51cto.com/09112012/2307767)

[centos7升级内核](https://blog.csdn.net/kikajack/article/details/79396793)

[MetalLB安装](https://metallb.universe.tf/installation/)

[nginx ingress 控制器原理探究](https://mp.weixin.qq.com/s/hzeS8LFHTTEA8IyNlej4tw)

[kibana-task-manager error](https://discuss.elastic.co/t/kibana-task-manager-1-issues/222839)

[helm教程](https://whmzsu.github.io/helm-doc-zh-cn/quickstart/using_helm-zh_cn.html)

