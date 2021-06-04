---
title: tcp_tw_recle参数引发的SYN重传
catalog: true
date: 2021-06-04 17:40:53
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- TCP/IP
catagories:
- TCP/IP
updateDate: 2021-06-04 17:40:53
---

公司的网络环境下打开App加载奇慢或者无响应，特此记录下曲折的排错历程。这是一个从网络到服务器，从SYN不断重传转到net.ipv4.tcp_tw_recycle = 0的过程。作为公司的网络管理者，我想说不能局限于别人的怀疑，不能局限于自己的思维定式，要不然解决问题真的很难。

#### 问题背景：

​		   公司的开发的APP在获取数据的时候无法加载，或者加载极慢。刚开始发现问题时表现如下：

- 平时打开App访问页面基本没有什么异常，打开App上页面也挺流畅（测试人员也不密集测试）。

  - 一般都是上线测试当天，多人访问就大概率会出现页面没有响应。导致测试开发人员十分不满，怒气值拉满。

  - 当他们切换到自己的4G网络时，又没有问题，加载速度杠杠的。

    

#### 排错思路第一阶段（自我怀疑）：

##### 分析问题：

1. 公司测试开发人员使用公司WiFi打开App有问题，使用4G访问没有问题，那肯定是公司WiFi问题。公司WiFi不行（技术人员如是说，我也觉得可能吧）

2. 为什么我也觉得可能。因为我们的WiFi虽然使用的AC+AP模式，但是AP使用的是普通H3C，性能一般，而且技术开发中心区域人员密集，最最重要的是排除每个人的手机，我们光测试机就有几十台。一台AP带机40-50很正常。

   公司WIFI架构简略

   ![image.png](https://i.loli.net/2021/05/21/adxHmOs2cuiFIQA.png)

   公司WiFi负载图

   ![image.png](https://i.loli.net/2021/05/21/NnQWtxVl6aiDSfY.png)

   

##### 调整测试：

1. 公司WiFi调整：

   ​	1.增加AP: 在部门位置增加了一个AP,发现没用，技术开发中心正上方AP带机量依然多。

   ​	2.优化AP性能-调整WiFi功率: 主要是提高信道强度，证实没用。

   ​	3.优化AP性能-调整WiFi 信道： 主要是排除信号干扰，发现没用，信号一直挺好。工具推荐使用WiFi魔盒测试，细节就不说了。

   ​	4.优化AP性能-提高AP性能： 参考各路大神的神仙操作乱秀一通，发现没有鸟用。证实没用。

   ​	5.优化AP性能-优化AP间负载： 即为同一区域AP带机数量太多自动负载到邻近AP上。证实没用。

   ​    6.检测是不是频繁切换AP: 发现即使机器在两个ap之间也稳如老狗。证实没用。

   ​	

##### 第一阶段成果：

​		心态崩了，问题没有解决，反而技术同事说网络更差了。有点欣慰的地方： 没有人比我更懂我的WiFi。如果不是WiFi问题，难道是运营商问题？



#### 排错思路第二阶段（深深的迷惘）：

##### 分析问题：

1. 难道因为公司WiFi线路是ADSL线路，访问连接数被运营商限制。

2. 难道是内网架构有问题，那就抓个包看看。

   

##### 调整测试：

​	1.直接将WiFi出口调整为电信ICP城域网： ADSL线路可能有并发线路限制，ICP城域网没有。证实没用。

​	2. 排除公司的网络准入和上网行为管理： 排除法和白名单，证实没用。

​	3.  出口防火墙的问题没办法证明： 公司的出口是防火墙并做负载均衡。

 4. 抓包分析： 

    手机APP抓包，最开始采用的抓包策略：FIDDLER+WIRESHARK.

    手机通过代理连接WIFI,代理机器是装有windows系统的笔记本。

    发现问题：手机ios老是不认fiddler证书。

    更换抓包方法：最终发现工具charles工具很是完美。最终的抓包方式为charles+wireshark. 

    演示图如下。

    ![image.png](https://i.loli.net/2021/05/21/XT2gv5B4r7b6tCd.png)

    然后崩溃的事情出现了：

    - 在代理使用App一切都正常，根本会出现慢的情况，所以压根抓不到有用的包。

    

    ##### 第一阶段成果：

##### 第二阶段成果：

​		此刻已经无解，抓包抓不到，问题也解决不了。但是不甘心，现在的怀疑方向就是出口防火墙和WiFi（还是它）。



#### 排错思路第三阶段（垂死病中惊坐起）：

分析问题：

##### 分析问题：

1. 怎么排除是WiFi的问题。

2. 怎么排除是防火墙的问题。

   

##### 调整测试：

1. 直接买了一个TP-LINK当作小路由器，抛弃内网WiFi： 测试发现使用tp-link，还是会出现同样的问题，直接证明和公司WiFi没有关系。
2. 将公网线路直连TP-LINK，抛弃内网干扰。测试发现还是出现问题，这证明和公司网络没有关系啊。但是我没有办法证明为什么使用4G连接app快啊。
3. 转折点是技术另外一位同事可以用charles可以抓到包。so 。。。。

##### 第三阶段成果：

​		我已经放弃，目前已经证明和公司网络没有关系。也就是和我没有关系。但是别人认为坚持认为4G没有问题，为啥公司WiFi不行，就是网络有问题。这让我真是太难了。而且解决不了问题，我会很难受，只能继续。



#### 排错思路第四阶段（起死回生）：

分析问题：

1. 怎么证明4G也有问题。
2. 赶紧去同事那里抓包分析。

##### 调整测试：

1. 既然公司是人多连的时候才会出现问题，那么共享热点不就可以验证4G网络是否可行了吗： 发现开了热点之后，不管4G，5G，只要人多，他照样有问题。这更证明不是公司网络问题。

2. 重点分析

   首先分析：为什么我抓不到，同事可以抓到包。我发现唯一的区别就是我使用的是windows环境，而同事使用的是mac。

   进而思考： 说明作为代理机器，windows发出去的数据包和mac机器不大相同。

3. 抓包分析：

   第一步抓取了代理客户端发出去的包，通过tcpdump -i en0 -nn host xxxx 只抓访问服务器的包。

   ![image.png](https://i.loli.net/2021/05/21/DHwlc2LhgoXPVTI.png)

   ![image-20210521162715302](C:\Users\wangwei\AppData\Roaming\Typora\typora-user-images\image-20210521162715302.png)

   抓包分析：

   ​		1.通过抓包结果来看。错误的包基本定位在SYN一直重传，而服务器端没有SYN+ACK.这就很离谱。 回想三次握手流程。

   ![image.png](https://i.loli.net/2021/05/21/bosRkiJ6rmHa8Iz.png)

     2. 经过排查在服务器没有SYN+ACK期间，是有其他连接的在传输数据的。这说明网络是正常的

        ![image.png](https://i.loli.net/2021/05/21/JZx6bmYXAMpaQqE.png)

     3. 既然网络正常连接，而一直SYN重传，说明服务器一直没有收到SYN包或者没有正确处理SYN包，如果服务器收到SYN包后，发送SYN+ACK，而没有收到客户端的ACK，那么服务器也会一直重传ACK+SYN,显然没有。

4. 排查服务器问题：

   ​	 1. 公司的机器在阿里云上，我也管不到开发服务器，所以先了解了基本情况。

   ​	  公司服务入口是阿里云的负载均衡SLB,后面是两台真实服务器,模式为RR.图例如下。同时我一直以为阿里云使用的lvs是fullnat,所以我认为在真实服务器上抓不到CIP，所以我提了工单让阿里云帮我抓SLB的包。

   ![image.png](https://i.loli.net/2021/05/21/3CP9kgi2NLMsU6Z.png)

   ​	2.阿里云方面出谋划策的拖了我几天，然后告诉我SLB抓不到包，要到真实服务器上抓包。同时告诉他们没用fullnat。最终我找到同事和我一起抓包。

   同样的问题还是出现了。服务器上同样是不停重传SYN，但是没有响应。再次印证了我的想法，网络没有问题。。。。。 服务器有问题。

   ![image.png](https://i.loli.net/2021/05/21/cjI7tMAulb9kEaR.png)

    3. 关于为什么服务器为什么没有响应。我通过”netstat -natp|grep  SYN_RECV“，服务器压根没有理会这个包。但是又是为什么呢，我猜测是内核参数问题，但是具体哪个参数设置有问题，我没有想到，网上也查不到相关资料。只能继续跟阿里云沟通了。

    4. 阿里云也给出参考：[linux系统内核配置问题导致NAT环境访问实例出现异常](https://help.aliyun.com/knowledge_detail/41297.html)。然后根据这两个参数的设置，终于在网上找到相关问题。

       [生产环境不要乱修改net.ipv4.tcp_tw_recycle](https://www.iyunw.cn/archives/xie-lin-lin-jiao-xun-sheng-cheng-huan-jing-bu-yao-luan-xiu-gai-net-ipv4-tcp-tw-recycle/)

       [改内核的小心了-net.ipv4_tw_recycle=1不能随便设置](http://www.cloudyy.cn/archives/1053/)

       ```
       大多都写开启net.ipv4.tcp_tw_recycle这个开关，可以快速回收处于TIME_WAIT状态的socket（针对Server端而言）。
       而实际上，这个开关，需要net.ipv4.tcp_timestamps（默认开启的）这个开关开启才有效果。
       更不为提到却很重要的一个信息是：当tcp_tw_recycle开启时（tcp_timestamps同时开启，快速回收socket的效果达到），对于位于NAT设备后面的Client来说，是一场灾难——会导到NAT设备后面的Client连接Server不稳定（有的Client能连接server，有的Client不能连接server）。也就是说，tcp_tw_recycle这个功能，是为“内部网络”（网络环境自己可控——不存在NAT的情况）设计的，对于公网，不宜使用。
       
       我们在一些高并发的 WebServer上，为了端口能够快速回收，打开了 tcp_tw_reccycle ，而在关闭 tcp_tw_reccycle 的时候，kernal 是不会检查对端机器的包的时间戳的；打开了 tcp_tw_reccycle 了，就会检查时间戳，很不幸移动的cmwap发来的包的时间戳是乱跳的，所以我方的就把带了“倒退”的时间戳的包当作是“recycle的tw连接的重传数据，不是新的请求”，于是丢掉不回包，造成大量丢包。
       ```

       

    5. 通过检查我们这两个参数确实修改了。最终修改这两个设置，解决问题。

       

##### 第四阶段成果：

​		终于解决，心里五味杂陈。



#### 最终总结：

​		1.不宜妄自菲薄。

​		2.不能因为别人的言语就干扰了自己的判断。

​		3.遇到问题解决问题，理清思路，一步一步的验证自己的想法。

​		4. 不要拘泥于百度和谷歌，有时在知名的技术论坛搜索，也可以事半功倍。

​		5.努力提升自己。



#### 参考链接：

[linux系统内核配置问题导致NAT环境访问实例出现异常](https://help.aliyun.com/knowledge_detail/41297.html)

[生产环境不要乱修改net.ipv4.tcp_tw_recycle](https://www.iyunw.cn/archives/xie-lin-lin-jiao-xun-sheng-cheng-huan-jing-bu-yao-luan-xiu-gai-net-ipv4-tcp-tw-recycle/)

[改内核的小心了-net.ipv4_tw_recycle=1不能随便设置](http://www.cloudyy.cn/archives/1053/)

小林图解网络。百度网盘链接: https://pan.baidu.com/s/1FfR8jpxdk6Bro23KddyU8A  密码: 9c4f 公众号：小林coding

