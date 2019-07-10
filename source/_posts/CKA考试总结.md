---
title: CKA考试总结
catalog: true
date: 2019-07-10 9:32:03
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- cka
- kubernetes
catagories:
- cka
- kubernetes
updateDate: 2019-07-10 9:32:03
---

#### CKA考试总结

​        很高兴自己考过了CKA，准备了很长一段时间，总算有所回报。

​        这次考试，花费了300$，预约的线上考试。最近看着华为云正在推广线下考试，只要一千多人民币，真是莫名心痛。先晒下证书缓解下。

![CKA](https://cdn.sinaimg.cn.52ecy.cn/large/005BYqpgly1g4tp5hllhlj30t20mek4b.jpg)

##### 经验说明

很多人已经分享过自己的经验，多的我就不说了，我只说我认为有用的。

1. 考试要求

   1. 考试环境：要求安静无人，桌面干净。 
   2. 考试方式： 只使用chrome浏览器，不会使用到其他软件。通过浏览器调用摄像头，麦克风，桌面分享等功能。具体操作方式类似于[katacoda](https://www.katacoda.com/courses/kubernetes/playground)。各位要趁机练练手啊。
   3. 考官交流： 不知道对面是男是女，与对方聊天全程打字，对方全程监控，并下达各种指令。对方会要求我们分享摄像头，分享桌面，没事不要乱动。全英文交流，所以只要英文阅读能力不错就行，其他时间可以全程 输入ok！

2. 网络要求

   1. 要翻墙。要求稳定，我使用的是google一年期的免费试用，自己搭建的梯子。速度很稳定，但是还是断了一次，幸亏考试是实时保存的，继续做题就行。

      可以申请阿里云-香港云主机；或者试用google云主机。下面的方式都搭梯子的方式是基于docker，简单快捷。

      [shadowsocks：](https://github.com/vavikast/docker-shadowsocks)

      [openvpn：](https://github.com/kylemanna/docker-openvpn)

   2. 最好安排早上： 这个是好多人提到的，好像是因为其他时间，国际网络出口较为拥挤。我是预约的6点，一切流程。

3. 技巧分享

   1.  多看书多练习

      [华为云CKA实战](https://bbs.huaweicloud.com/forum/thread-11064-1-1.html)

      [马哥kubernetes](https://www.bilibili.com/video/av44264629?from=search&seid=16743035774806411630)

      [kubernetes官网](https://kubernetes.io/)

      [kubernetes指南](https://legacy.gitbook.com/book/feisky/kubernetes/details)

         可以先看马哥的教程先了解kubernetes是什么，然后看feisky的kubernetes指南，了解k8s概念和操作，然后看k8s官网进行英文概念理解记忆，最后看华为云cka，准备考试。最重要的就是看kubernetes官网，所有的考点基本都在上面，而且是考试唯一可以看的网页。

   2. 加快考试速度。

      2.1使用命令补全：

      ```
      source <(kubectl completion bash)
      ```

      2.2使用别名；

      ```
      alias kc='kubectl'
      alias kgp='kubectl get pods'
      alias kgs='kubectl get svc'
      ```

      2.3能用命令行，就不要自己写yaml文件；

      2.4 能够从kubernetes.io中复制粘贴的，就不要自己敲命令；复制粘贴我用的是鼠标的粘贴复制，没有使用shift+insert,ctrl+insert，不支持ctrl+c,ctrl+v。

   3. 注意审题，认真做题

      3.1中英文切换审题，特别是有歧义的时候。

      3.2 注意切换运行环境，kubectl config set-config。

   4.  命令行和yaml问题

       考试中有关kubernetes本身的题目，除了pv和pvc，其他情况下基本都可以使用kubectl命令行去转换成yaml文件，没有命令行的可以直接粘贴复制kubernetes.io文件，请各位多加练习。可以将kubectl 命令行撸个好几遍。

   ​              不建议这个使用使用kubectl explain ,可以使用kubectl command -h更快。

      常用举例

      生成pod 

      ```
      kubectl run pod --image=nginx --restart=Never --requests=
      ```

      生成job

      ```
      kubectl run job --image=nginx --restart=OnFailure
      ```

      生成deploy

      ```
      kubectl run deploy --image=nginx  --replicas=2
      ```

      生成svc

      ```
      kubectl expose deployment sd2 --type=ClusterIP --target-port=80 --name=sd2 
      ```

      生成名称空间

      ```
      kubectl create ns ns
      ```

      生成configmap

      ```
      kubectl create configmap  db2   --from-literal=nt=nt
      ```

   5. 考试策略

      遇到难题不会就先过了，在考试提供的notepad上记下来，再来回头看。



##### 真题分享

 我把我能够记住的题目分享出来。具体答案就不分享了，不想引导错误，提供一点思路和相关链接，仅供参考。每个人的考题内容可能不一样，要多看啊。

1.列出指定pod的日志中状态为Error的行，并记录在指定的文件上。

```
kubectl logs <podname> | grep bash > xxx.txt
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-pod-replication-controller/
```

2.列出环境内所有的pv 并以 capacity字段排序

```
（使用kubectl自带排序功能，field字段关注下）
kubectl get pv --sort-by=.spec.capacity.storage
https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/
```

3.列出k8s可用的节点，不包含不可调度的 和 NoReachable的节点，并把数字写入到文件里。

```
kubectl get nodes; kubectl get nodes xxx -oyaml
https://kubernetes.io/docs/concepts/architecture/nodes/
```

4.创建一个pod名称为nginx，并将其调度到节点为 disk=stat上

```
nodeselector 字段
https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
```

5.提供一个pod的yaml，要求添加Init Container，Init Container的作用是创建一个空文件，pod的Containers判断文件是否存在，不存在则退出

```
init container
https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
```

6.指定在命名空间内创建一个pod名称为test，内含四个指定的镜像nginx、redis、memcached、busybox.

```
https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/
```

7.创建一个命名空间

```
kubectl create ns xx
https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/
```

8.在一个新的命名空间创建pod

```
https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/
```

9.列出Service名为test下的pod 并找出使用CPU使用率最高的一个，将pod名称写入文件中.

```
kubectl get svc test -oyaml; 查看配置内容，特别找出标签
kubectl top nodes --selector="xxx=yy"
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/
```

10.创建一个Pod名称为nginx-app，镜像为nginx，并根据pod创建名为nginx-app的Service，type为NodePort

```
kubectl expose xx
https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/
```

11.创建一个nginx的daemonset，保证其在每个节点上运行，注意不要覆盖节点原有的Tolerations

```
https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
```

12.将deployment为nginx-app的副本数从1变成4

```
kubectl scale deployment xx --replicas=4
https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/
```

13.创建nginx-app的deployment ，使用镜像为nginx:1.11.0-alpine ,修改镜像为1.11.3-alpine，并记录升级，再使用回滚，将镜像回滚至nginx:1.11.0-alpine

```
kubectl rollout xx
https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/
```

14.根据已有的一个nginx的pod、创建名为nginx的svc、并使用nslookup查找出service dns记录，pod的dns记录并分别写入到指定的文件中

```
busybox:1.28
https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/
```

15.创建Secret 名为mysecret，内含有password字段，值为bob，然后 在pod1里 使用ENV进行调用，Pod2里使用Volume挂载在/data 下

```
https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/
https://kubernetes.io/docs/concepts/storage/volumes/
```

16.使node1节点不可调度，并重新分配该节点上的pod

```
kubectl drain nodex
kubectl uncordon nodex
https://kubernetes.io/docs/concepts/architecture/nodes/
```

17.使用etcd 备份功能备份etcd（提供enpoints，ca、cert、key）

```
https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
https://blog.51cto.com/9406836/2409012
https://github.com/mmumshad/certified-kubernetes-administrator-course-answers/blob/master/etcd-backup-and-restore.md
```

18.给出一个失联节点的集群，排查节点故障，要保证改动是永久的。

```
kubectl get nodes 故障节点 -oyaml 查看故障
远程到故障节点启动kubelet服务
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/
```

19.给出一个集群，排查出集群的故障。集群无法使用kubectl。

```
集群无法使用kubectl，先到主节点，查看kubelet配置，static-pod配置文件出错，更改重新
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/
```

20.创建static pods.

```
https://kubernetes.io/docs/tasks/administer-cluster/static-pod/
```

21.给出一个集群，将节点node1添加到集群中，并使用TLS bootstrapping

```
https://github.com/mmumshad/certified-kubernetes-administrator-course-answers/blob/master/tls-bootstrap-worker-node-2
https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/07-2.kubelet.md
```

22.创建一个pv，类型是hostPath，位于/data中，大小1G，模式ReadOnlyMany

```
https://kubernetes.io/docs/concepts/storage/persistent-volumes/
```

23.创建一个pod，使用hostpath。

```

```



##### 经验贴推荐

下面这些都是我我认为写的很好，考试一定会用到的资料，如果大家考CKA，请一定要看。

[CKA学习资源汇总，只要找到CKA部分就行](https://docs.google.com/spreadsheets/d/10NltoF_6y3mBwUzQ4bcQLQfCE1BWSgUDcJXy-Qp2JEU/edit#gid=0)

[ETCD备份+node节点认证](https://github.com/mmumshad/certified-kubernetes-administrator-course-answers)

[CKA-Studyguide](https://github.com/David-VTUK/CKA-StudyGuide)

[CKA考试知识总结](http://ljchen.net/2018/11/07/CKA%E8%80%83%E8%AF%95%E7%9F%A5%E8%AF%86%E6%80%BB%E7%BB%93/)

[CKA考试经验总结](http://ljchen.net/2018/11/07/CKA%E8%80%83%E8%AF%95%E7%9F%A5%E8%AF%86%E6%80%BB%E7%BB%93/)

