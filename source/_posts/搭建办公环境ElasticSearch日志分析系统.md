---
title: 搭建办公环境ElasticSearch日志分析系统
catalog: true
date: 2019-12-20 12:32:03
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- elasticsearch
catagories:
- elasticsearch
updateDate: 2019-12-20 12:32:03
---
# 搭建办公环境ElasticSearch 日志分析系统

​			计划将公司的防火墙+交换机+服务器（centos7）+ Vmware+Windows server纳入到监控范围，所以开启了ELK监控之旅。

​			本文采用ELK架构栈进行组建，万丈高楼平地起，虽然开始比较简陋，后期会不断完善这个日志分析系统。

​	全文框架如下：	

​			Hillstone：                  syslog→logstash→elasticsearch→kibana

​			H3C:                              syslog→logstash→elasticsearch→kibana

​			ESXI:                              syslog→logstash→elasticsearch→kibana

​            Vcenter:                        syslog→logstash→elasticsearch→kibana

​			Windows server:         winlogbeat→logstash→elasticsearch→kibana

​			linux server：			  filebeate→lasticsearch→kibana

​	ELK说明：

​			ELK1: 192.168.20.18:9200

​			ELK2: 192.168.20.19:9200

​	规划：

​			Logstash: 192.168.20.18

​			不同服务根据端口不同进行标记，创建不同的索引。

​	

## 1 Hillstone部分

### 1.1 Hillstone配置

#### 1.1.1 配置步骤

​	本文通过web界面配置，当然也能进行命令行配置，具体配置请参考链接。	

​	找到Stoneos-日志管理-log配置-日志管理器，配置服务器日志：
​				主机名： 192.168.20.18
​				绑定方式： 虚拟路由器 trust-vr
​				协议： UDP
​				端口： 514  

​				//我使用的root运行，非root账号使用端口在1024以上。

​		![image.png](https://i.loli.net/2019/12/06/JZOcqSkiVoTp8nU.png)

![image.png](https://i.loli.net/2019/12/06/CtlWXfbrxE5djTJ.png)

![image.png](https://i.loli.net/2019/12/06/oMpAyXRQx9LPO1b.png)

#### 1.1.2 参考链接

[elk收集数据中心网络设备日志](https://elasticsearch.cn/question/8247)

[hillstone常见配置命令](https://wenku.baidu.com/view/2c6a500149649b6649d747c4.html)



### 1.2 添加logstash 配置文件

#### 1.2.1  创建配置测试文件

```
cat  > /data/config/test-hillstone.config << EOF
input{
        udp {port => 518 type => "Hillstone"}
}
output {
    stdout { codec=> rubydebug }
}
EOF
logstash -f test-hillstone.config
```

#### 1.2.2  选取其中一条日志进行分析调试。

```
<190>Nov 29 17:24:52 1404726150004842(root) 44243624 Traffic@FLOW: SESSION: 10.6.2.43:49608->192.168.20.160:11800(TCP), application TCP-ANY, interface tunnel6, vr trust-vr, policy 1, user -@-, host -, send packets 1,send bytes 74,receive packets 1,receive bytes 110,start time 2019-11-29 17:24:50,close time 2019-11-29 17:24:52,session end,TCP RST\n\u0000
```

#### 1.2.3Logstash分析说明

		可以通过grok debug网站进行自动匹配（https://grokdebug.herokuapp.com/discover?#），再根据分析出来的日志，进行二次调整。
		同样晚上有很多案例进行参考，可以先去参考别人想法，再补充自己的想法。
		关于grok部分详细讲解，请参考https://coding.imooc.com/class/181.html，老师讲的很棒，当然吾爱破解论坛和B站，有免费版。
#### 1.2.4  Logstash综合配置

​		只选取了会话+NAT部分

```
cat  > /data/config/hillstone.config<< EOF
input{
        udp {
		port => 518 
		type => "hillstone"
		}

}
filter {
    grok {
##流量日志

#SESSION会话结束日志
        match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{BASE10NUM:serial}\(%{WORD:ROOT}\) %{DATA:logid}\ %{DATA:Sort}@%{DATA:Class}\: %{DATA:module}\: %{IPV4:srcip}\:%{BASE10NUM:srcport}->%{IPV4:dstip}:%{WORD:dstport}\(%{DATA:protocol}\), application %{USER:app}\, interface %{DATA:interface}\, vr %{USER:vr}\, policy %{DATA:policy}\, user %{USERNAME:user}\@%{DATA:AAAserver}\, host %{USER:HOST}\, send packets %{BASE10NUM:sendPackets}\,send bytes %{BASE10NUM:sendBytes}\,receive packets %{BASE10NUM:receivePackets}\,receive bytes %{BASE10NUM:receiveBytes}\,start time %{TIMESTAMP_ISO8601:startTime}\,close time %{TIMESTAMP_ISO8601:closeTime}\,session %{WORD:state}\,%{GREEDYDATA:reason}"}
#SESSION会话开始日志
        match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{BASE10NUM:serial}\(%{WORD:ROOT}\) %{DATA:logid}\ %{DATA:Sort}@%{DATA:Class}\: %{DATA:module}\: %{IPV4:srcip}\:%{BASE10NUM:srcport}->%{IPV4:dstip}:%{WORD:dstport}\(%{DATA:protocol}\), interface %{DATA:interface}\, vr %{DATA:vr}\, policy %{DATA:policy}\, user %{USERNAME:user}\@%{DATA:AAAserver}\, host %{USER:HOST}\, session %{WORD:state}%{GREEDYDATA:reason}"}
#SNAT日志
        match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{BASE10NUM:serial}\(%{WORD:ROOT}\) %{DATA:logid}\ %{DATA:Sort}@%{DATA:Class}\: %{DATA:module}\: %{IPV4:srcip}\:%{BASE10NUM:srcport}->%{IPV4:dstip}:%{WORD:dstport}\(%{DATA:protocol}\), %{WORD:state} to %{IPV4:snatip}\:%{BASE10NUM:snatport}\, vr\ %{DATA:vr}\, user\ %{USERNAME:user}\@%{DATA:AAAserver}\, host\ %{DATA:HOST}\, rule\ %{BASE10NUM:rule}"}
#DNAT日志
                match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{BASE10NUM:serial}\(%{WORD:ROOT}\) %{DATA:logid}\ %{DATA:Sort}@%{DATA:Class}\: %{DATA:module}\: %{IPV4:srcip}\:%{BASE10NUM:srcport}->%{IPV4:dstip}:%{WORD:dstport}\(%{DATA:protocol}\), %{WORD:state} to %{IPV4:dnatip}\:%{BASE10NUM:dnatport}\, vr\ %{DATA:vr}\, user\ %{USERNAME:user}\@%{DATA:AAAserver}\, host\ %{DATA:HOST}\, rule\ %{BASE10NUM:rule}"}
    }
    mutate {
				lowercase => [ "module" ]
                remove_field => ["host", "message", "ROOT", "HOST", "serial", "syslog_pri", "timestamp", "mac", "AAAserver", "user"]
        }
}
output {
    elasticsearch {
    hosts => "192.168.20.18:9200"  #elasticsearch服务地址
    index => "logstash-hillstone-%{module}-%{state}-%{+YYYY.MM.dd}"
    }
}
EOF

```

##### 3.1.2.3 参考文件

[hillstone中logstash配置参考](https://github.com/sectong/elk-docker-hillstone/blob/master/config/14-filters-hillstone.conf)

[elk收集数据中心网络设备日志](https://elasticsearch.cn/question/8247)

[山石hillstone Logstash配置流程](https://www.jianshu.com/p/459f3b12f56b)

[ELK从入门到实践](https://www.bilibili.com/video/av77345745?from=search&seid=5493175331969085253)



## 2 H3C交换机部分

## 2.1 H3C Syslog转发

#### 2.1.1 H3C交换机配置

​	本文通过命令行进行配置，具体配置请参考链接。

- ​	将交换机的时间设置正确

  ```
  clock datetime hh:mm:ss year/month/day
  save force
  ```

- 设置交换机syslog转发。

  ```
  system-view
  info-center enable // 开启info-center
  info-center loghost 192.168.20.18 port 516 facility local8 // 设置日志主机/端口/日志级别
  info-center source  default  loghost  level   informational  //设置日志级别
  save force
  
  ```

#### 2.1.2 参考链接

​	[H3C设置时间](https://www.cnblogs.com/hello1123/p/9202309.html)

​	[H3C网络日志转发](https://blog.csdn.net/vbaspdelphi/article/details/76735949)

​	[H3C配置日志主机](http://www.voidcn.com/article/p-kmrkuavo-bha.html)

### 2.2 添加logstash 配置文件

#### 2.2.1  创建配置测试文件

```
cat  > /data/config/test-h3c.congfig << EOF
input{
        udp {port => 516 type => "h3c"}
}
output {
    stdout { codec=> rubydebug }
}
EOF
```

#### 2.2.2 选取一条日志进行分析调试。

```
<190>Nov 30 16:27:23 1404726150004842(root) 44243622 Traffic@FLOW: SESSION: 10.6.4.178:48150->192.168.20.161:11800(TCP), interface tunnel6, vr trust-vr, policy 1, user -@-, host -, session start\n\u0000
```

#### 2.2.3 mutate基本用法说明

​			参考链接： https://blog.csdn.net/qq_34624315/article/details/83013531

#### 2.2.4 综合配置。

```
cat > H3C.conf <<EOF
###h3c 日志过滤
    grok {
        match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{DATA:year} %{DATA:hostname} \%\%%{DATA:ddModuleName}\/%{POSINT:severity}\/%{DATA:brief}\: %{GREEDYDATA:reason}" }       
        add_field => {"severity_code" => "%{severity}"}          
        }
    mutate {	
		gsub => [
            "severity", "0", "Emergency",
            "severity", "1", "Alert",
            "severity", "2", "Critical",
            "severity", "3", "Error",
            "severity", "4", "Warning",
            "severity", "5", "Notice",
            "severity", "6", "Informational",
            "severity", "7", "Debug"
        ]
        remove_field => ["message", "syslog_pri"]
        }
}
output {
	stdout { codec=> rubydebug }
#    elasticsearch {
#    hosts => "192.168.20.18:9200"  #elasticsearch服务地址
#    index => "logstash-h3c-%{+YYYY.MM.dd}"
#    }
}
EOF

```

#### 2.2.5 参考文件

[交换路由等网络设备logstash配置](https://github.com/qyecst/notes/blob/master/elk%E6%8E%A5%E5%85%A5%E4%BA%A4%E6%8D%A2%E6%9C%BA%26%E8%B7%AF%E7%94%B1%E5%99%A8%E7%AD%89%E7%BD%91%E7%BB%9C%E8%AE%BE%E5%A4%87%E6%97%A5%E5%BF%97.md)

[logstash配置文件](https://gist.github.com/mrlesmithjr/72e99caf36fcc2b5d323)



## 3 Vmware部分

### 3.1 Esxi日志收集部分

​			    主要是收集ESXI机器日志，方便进行安全日志分析；

​			    主要通过syslog进行日志收集，再通过ELK栈提供的logstash进行分析

​			    ESXI-syslog--logstash--elasticsearch

#### 3.1.1 Esxi机器syslog服务器配置

##### 3.1.1.1 配置步骤

​	本文通过客户端配置，当然也能进行web配置，方法基本一致，具体配置请参考链接。

- ​	开启syslog服务：

  打开esxi客户端-选择主机-主机配置-高级设置-syslog-设置远程syslog服务器为： udp://192.168.20.18:514

  ![1575357915_1_.jpg](https://i.loli.net/2019/12/03/FvIjaylK15ikZnM.png)

-    允许防火墙放行。

  打开esxi客户端-选择主机-主机配置-安全配置文件-防火墙-编辑-勾选syslog服务器，点击确定。

  ![1575358020_1_.jpg](https://i.loli.net/2019/12/03/y4zejJZC5RIW7Pu.png)

  ![1575358066_1_.jpg](https://i.loli.net/2019/12/03/b2OIAMklJKGWf4a.png)

  

  ##### 3.1.1.2 参考链接

  [Vmware Esxi syslog配置]([https://techexpert.tips/zh-hans/vmware-zh-hans/vmware-esxi-syslog%E9%85%8D%E7%BD%AE/](https://techexpert.tips/zh-hans/vmware-zh-hans/vmware-esxi-syslog配置/))

  [在Esxi上配置syslog](https://kb.vmware.com/articleview?docid=2003322&lang=zh_CN)

  [Monitoring VMWare ESXi with the ELK Stack](https://mtalavera.wordpress.com/2015/05/18/monitoring-vmware-esxi-with-the-elk-stack/)

  #### 3.1.2 添加logstash 配置文件

  ##### 3.1.2.1 创建配置测试文件
  
  ```
  cat  > /data/config/test-vmware.config << EOF
  input{
          udp {port => 514 type => "Hillstone"}
  }
  output {
      stdout { codec=> rubydebug }
  }
  EOF
logstash -f test-vmware.config
  ```

  ##### 3.1.2.2 选取其中一条日志进行分析调试。
  
  ```
<167>2019-12-03T07:36:11.689Z localhost.localdomain Vpxa: verbose vpxa[644C8B70] [Originator@6876 sub=VpxaHalCnxHostagent opID=WFU-2d14bc3d] [WaitForUpdatesDone] Completed callback\n
  ```

  ##### 3.1.2.3 结合网上的综合案例对测试文件进行配置改造。
  
  ```
  cat > vmware.conf <<EOF
  input{
          udp {
          port => 514
          type => "vmware"
  }
  }
  
  filter {
  	if "vmware" in [type] {
  		grok {
  			break_on_match => true
  			match => [
  				"message", "<%{POSINT:syslog_pri}>%{TIMESTAMP_ISO8601:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{SYSLOGPROG:syslog_program}: (?<message-body>(?<message_system_info>(?:\[%{DATA:message_thread_id} %{DATA:syslog_level} \'%{DATA:message_service}\'\ ?%{DATA:message_opID}])) \[%{DATA:message_service_info}]\ (?<syslog_message>(%{GREEDYDATA})))",
  				"message", "<%{POSINT:syslog_pri}>%{TIMESTAMP_ISO8601:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{SYSLOGPROG:syslog_program}: (?<message-body>(?<message_system_info>(?:\[%{DATA:message_thread_id} %{DATA:syslog_level} \'%{DATA:message_service}\'\ ?%{DATA:message_opID}])) (?<syslog_message>(%{GREEDYDATA})))",
  				"message", "<%{POSINT:syslog_pri}>%{TIMESTAMP_ISO8601:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{SYSLOGPROG:syslog_program}: %{GREEDYDATA:syslog_message}"
  			]
  		}
  		 date {
                          match => [ "syslog_timestamp", "YYYY-MM-ddHH:mm:ss", "ISO8601" ]
                  }
  		mutate {
  			replace => [ "@source_host", "%{syslog_hostname}" ]
  		}
  		mutate {
  			replace => [ "@message", "%{syslog_message}" ]
  		}
  		mutate {
  			remove_field => ["@source_host","program","@timestamp","syslog_hostname","@message"]
  		}
  		if "Device naa" in [message] {
  			grok {
  				break_on_match => false
  				match => [
  					"message", "Device naa.%{WORD:device_naa} performance has %{WORD:device_status}%{GREEDYDATA} of %{INT:datastore_latency_from}%{GREEDYDATA} to %{INT:datastore_latency_to}",
  					"message", "Device naa.%{WORD:device_naa} performance has %{WORD:device_status}%{GREEDYDATA} from %{INT:datastore_latency_from}%{GREEDYDATA} to %{INT:datastore_latency_to}"
  				]
  			}
  		}
  		if "connectivity issues" in [message] {
  			grok {
  				match => [
  					"message", "Hostd: %{GREEDYDATA} : %{DATA:device_access} to volume %{DATA:device_id} %{DATA:datastore} (following|due to)"
  				]
  			}
  		}
  		if "WARNING" in [message] {
  			grok {
  				match => [
  					"message", "WARNING: %{GREEDYDATA:vmware_warning_msg}"
  				]
  			}
  		}
  	}
  }
  output {
      elasticsearch {
      hosts => "192.168.20.18:9200"  #elasticsearch服务地址
      index => "logstash-vmware-%{+YYYY.MM.dd}"
      }
  #    stdout { codec=> rubydebug }
  }
  EOF

  ```

  

  ##### 3.1.2.3 参考文件

  [mutate基本用法](https://blog.csdn.net/qq_34624315/article/details/83013531)

  [基本logstash配置文件参考](https://gist.github.com/vavikast/f74d90fd59f2db639bb01965527adaf7)

  [vmware and syslog](https://discuss.elastic.co/t/vmware-and-syslog/65479)

  [logstash  VCSA6.0](https://everythingshouldbevirtual.com/logstash-vcsa-6-0/)

  [filter plugins](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)

  ### 3.2 Vcenter日志收集部分

  #### 3.2.1 Vcenter-syslog服务器配置

  ​			主要是收集vcsa机器日志，方便进行安全日志分析；

  ​			主要通过syslog进行日志收集，再通过ELK栈提供的logstash进行分析

  ​			VCSA-Syslog--Logstash--Elasticsearch

  ##### 3.2.1.1 配置步骤

  ​	打开VCSA的管理后台URL: http://192.168.20.90:5480，输入账号和密码（开机root和密码)--点击syslog配置中心，输入syslog配置信息。

  ![1575460227_1_.jpg](https://i.loli.net/2019/12/05/O6ZjYzro4GnxWLI.png)

  ![1575460311_1_.jpg](https://i.loli.net/2019/12/05/KJ2akhibjDN9ZPE.png)
  
  ![1575460358_1_.jpg](https://i.loli.net/2019/12/05/xzRQkWn9dpVIgPi.png)
  
  
  
  ##### 3.2.1.2 参考链接
  
  [Vmware Esxi syslog配置]([https://techexpert.tips/zh-hans/vmware-zh-hans/vmware-esxi-syslog%E9%85%8D%E7%BD%AE/](https://techexpert.tips/zh-hans/vmware-zh-hans/vmware-esxi-syslog配置/))
  
  [VCSA 6.5 forward to multiple syslog](https://www.virtuallyghetto.com/2017/12/can-the-vcsa-6-5-forward-to-multiple-syslog-targets.html)
  
  [VCSA syslog](https://www.virtuallyghetto.com/2017/02/what-logs-do-i-get-when-i-enable-syslog-in-vcsa-6-5.html)
  
  #### 3.2.2 添加logstash 配置文件
  
  ##### 3.2.2.1 创建测试配置
  
  ```
  input{
          udp {
          port => 1514
          type => "vcenter"
  }
  }
  
  output {
      stdout { codec=> rubydebug }
  }
  
  ```
  
  ##### 3.2.2.2 选取一条日志进行分析调试
  
  ```
  <14>1 2019-12-05T02:44:17.640474+00:00 photon-machine vpxd 4035 - -  Event [4184629] [1-1] [2019-12-05T02:44:00.017928Z] [vim.event.UserLoginSessionEvent] [info] [root] [Datacenter] [4184629] [User root@192.168.20.17 logged in as pyvmomi Python/3.6.8 (Linux; 3.10.0-957.el7.x86_64; x86_64)]\n
  ```
  
  ##### 3.2.2.3 结合网上的综合案例对配置文件进行配置改造。
  
  ```
  cat > vcenter.conf <<EOF
  input{
          udp {
          port => 1514
          type => "vcenter"
  }
  }
  filter {
    if "vcenter" in [type] {
      }
      grok {
        break_on_match => true
        match => [
          "message", "<%{NONNEGINT:syslog_pri}>%{NONNEGINT:syslog_ver} +(?:%{TIMESTAMP_ISO8601:syslog_timestamp}|-) +(?:%{HOSTNAME:syslog_hostname}|-) +(-|%{SYSLOG5424PRINTASCII:syslog_program}) +(-|%{SYSLOG5424PRINTASCII:syslog_proc}) +(-|%{SYSLOG5424PRINTASCII:syslog_msgid}) +(?:%{SYSLOG5424SD:syslog_sd}|-|) +%{GREEDYDATA:syslog_msg}"
        ]
      }
      date {
           match => [ "syslog_timestamp", "YYYY-MM-ddHH:mm:ss,SSS", "YYYY-MM-dd HH:mm:ss,SSS", "ISO8601" ]
           #timezone => "UTC" #For vCenter Appliance
           #timezone => "Asia/Shanghai"
      }
  
     mutate {
           remove_field => ["syslog_ver", "syslog_pri"]
                          }
  
  }
  output {
      elasticsearch {
      hosts => "192.168.20.18:9200"  #elasticsearch服务地址
      index => "logstash-vcenter-%{+YYYY.MM.dd}"
      }
  #    stdout { codec=> rubydebug }
  }
  
  EOF
  
  ```
  
  ##### 3.2.2.4 参考文件
  
  [mutate基本用法](https://blog.csdn.net/qq_34624315/article/details/83013531)
  
  [基本logstash配置文件参考](https://gist.github.com/vavikast/f74d90fd59f2db639bb01965527adaf7)
  
  [vmware and syslog](https://discuss.elastic.co/t/vmware-and-syslog/65479)
  
  [logstash  VCSA6.0](https://everythingshouldbevirtual.com/logstash-vcsa-6-0/)
  
  
  
  ## 4 Windows server部分
  
  ### 4.1 windows server日志收集部分
  
  ​				主要是收集AD域日志，方便进行安全日志分析；
  
  ​			    主要通过ELK栈提供的winlogbeat进行收集
  
  ​			    winlogbeat--logstash--elasticsearch
  
  #### 4.1.1 windows服务器配置
  
  - ##### 下载安装
  
    下载连接地址：https://www.elastic.co/cn/downloads/beats/winlogbeat
  
    ![1575529356_1_.jpg](https://i.loli.net/2019/12/05/6VoPqRWOTt5CevQ.png)
  
  - ##### 配置： 
  
    将解压后的文件放到“C:\Program Files”，重命名为winlogbeat
  
    ![1575529614_1_.jpg](https://i.loli.net/2019/12/05/uX7AjbDeJ8TISPC.png)
  
    
  
  - ##### 安装 ： 
  
    命令安装
  
    ![1575529682_1_.jpg](https://i.loli.net/2019/12/05/hpxg19vrmujiJEc.png)
  
    
  
  - ##### 编辑：
  
    编辑winlogbeat.yml文件
  
    ```
    winlogbeat.event_logs:
      - name: Application
      - name: Security
      - name: System
    
    output.logstash:
      enbaled: true
      hosts: ["192.168.20.18:5044"]
    
    logging.to_files: true
    logging.files:
      path: D:\ProgramData\winlogbeat\Logs
    logging.level: info
    ```
  
  - ##### 测试配置文件
  
    ```
    PS C:\Program Files\Winlogbeat> .\winlogbeat.exe test config -c .\winlogbeat.yml -e
    ```
  
  - ##### 启动： 
  
    启动winlogbeat
  
    powershell命令行启动：
  
    ```
    PS C:\Program Files\Winlogbeat> Start-Service winlogbeat
    ```
  
    powershell命令行关闭
  
    ```
    PS C:\Program Files\Winlogbeat> Stop-Service winlogbeat
    ```
  
  - ##### 导入模板
  
    导入winlogindex模板,因为我们使用的logstash，所以需要手动导入。
  
    ```
    PS > .\winlogbeat.exe setup --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["192.168.20.18"]'
    ```
  
  - ##### 导入dashboard
  
    导入kibana-dashboard，因为我们使用的logstash，所以需要手动导入。
  
    ```
    PS > .\winlogbeat.exe setup -e  -E output.logstash.enabled=false   -E output.elasticsearch.hosts=['192.168.20.18:9200']   -E setup.kibana.host=192.168.20.18:5601
    ```
  
  #### 4.1.2 参考链接
  
  ​	   [windows 上winlogbeat安装](https://bbs.ichunqiu.com/forum.php?mod=viewthread&tid=20712&highlight=elk)
  
  ​       [官方手册分析](https://s0www0elastic0co.icopy.site/guide/en/beats/winlogbeat/7.x/winlogbeat-configuration.html)
  
  ### 4.2 添加logstash 配置文件
  
  #### 4.2.1 创建测试配置文件
  
  ```
  cat  > /data/config/test-windows.config << EOF
  input {
    beats {
      port => 5044
    }
  }
  
  output {
    stdout { codec=> rubydebug }
  }
  EOF
  logstash -f test-windows.config
  ```
  
  #### 4.2.2  创建配置文件
  
  创建正式配置文件，查看内容（因为已有模板，所以不错其他修改处理）
  
  ```
  cat  > /data/config/windows.config << EOF
  input {
    beats {
      port => 5044
    }
  }
  
  output {
    elasticsearch {
      hosts => ["http:192.168.20.18:9200"]
      index => "%{[@metadata][beat]}-%{[@metadata][version]}" 
    }
  }
  EOF
  logstash -f windows.config
  
  ```
  
  #### 4.2.3 参考文件
  
  [beats input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-beats.html)
  
  ### 4.3查看Discover
  
  ​			因为已经winlogbeat是elk栈的标准模块，已经被定义，所以我们不再自行定义。
  
  ​	        直接打开搜索winlogbaet*的index。
  
  ![1575533728_1_.jpg](https://i.loli.net/2019/12/05/UkEhZTpN42o9gnv.png)
  
  ### 4.4 查看dashboard
  
  ​			因为已经winlogbeat是elk栈的标准模块，已经被定义，所以我们不再自行定义。
  
  ​	        直接打开搜索winlogbaet*的dashboard
  
  ![1575533672_1_.jpg](https://i.loli.net/2019/12/05/kUiDL8mV4ZjHPce.png)
  
  
  
  ![1575533606_1_.jpg](https://i.loli.net/2019/12/05/xtOaieJFTdCWBmV.png)
  
  
  
  ## 5 Linux server部分
  
  ### 5.1 linux 系统日志收集部分
  
  ​	        主要是收集linux上的开关机日志，安全日志；
  
  ​			主要通过ELK栈提供的filebeat进行收集
  
  ​			 filebeat-filebeat-module--elasticsearch
  
  ​	![1575546466_1_.jpg](https://i.loli.net/2019/12/05/vlP1wUhpi7d6m4O.png)
  
  
  
  #### 5.1.1  安装部署
  
  - ##### 下载
  
    官网下载，连接地址：https://www.elastic.co/cn/downloads/beats/filebeat
  
    ![1575539230_1_.jpg](https://i.loli.net/2019/12/05/aSlejqExJtLyQrb.png)
  
    
  
  - ##### 安装 ：
  
    命令行安装
  
    ```
    curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.5.0-linux-x86_64.tar.gz
    tar xzvf filebeat-7.5.0-linux-x86_64.tar.gz
    ```
  
  - ##### 查看布局
  
    查看filebeat目录布局
  
    ![image.png](https://i.loli.net/2019/12/05/hzexyXouBJtOVkc.png)
  
    
  
  - ##### 编辑：
  
    编辑filebeat.yml文件
  
    ```
    filebeat.inputs:
    - type: log
      enabled: true
      paths:
        - /var/log/*.log
        #- c:\programdata\elasticsearch\logs\*
    output.elasticsearch:
      hosts: ["192.168.20.18:9200"]
    setup.kibana:
      host: "192.168.20.18:5601"
    
    ```
  
  - ##### 启动： 
  
    运行filebeat文件
  
    ```
    filebeat  -c filebeat.yml -e
    ```
  
  #### 5.1.2 参考链接
  
  [windows 上winlogbeat安装](https://bbs.ichunqiu.com/forum.php?mod=viewthread&tid=20712&highlight=elk)
  
  [官方手册分析](https://s0www0elastic0co.icopy.site/guide/en/beats/winlogbeat/7.x/winlogbeat-configuration.html)
  
  ### 5.2 修改filebeat配置文件
  
  #### 5.2.1索引名称
  
  关闭ilm声明周期管理
  
  ```
  setup.ilm.enabled: false
  ```
  
  更改索引名称
  
  ```
  setup.template.overwrite: true
  output.elasticsearch.index: "systemlog-7.3.0-%{+yyyy.MM.dd}"
  setup.template.name: "systemlog"
  setup.template.pattern: "systemlog-*"
  setup.template.enabled: false
  ```
  
  修改预先构建的kibana仪表盘
  
  ```
  setup.dashboards.index: "systemlog-*"
  ```
  
  #### 5.2.1 开启system模块
  
  ```
  ./filebeat modules enable system
  ./filebeat modules list
  ```
  
  #### 5.2.3 重新初始化环境
  
  ```
  ./filebeat setup --template -e -c  filebeat.yml
  ```
  
  #### 5.2.4 配置模块，修改配置文件
  
  ```
  filebeat.modules:
  - module: system
    syslog:
      enabled: true
  	#默认位置/var/log/messages* /var/log/syslog*
    auth:
      enabled: true
  	#默认位置/var/log/auth.log*  /var/log/secure*
  output.elasticsearch:
    hosts: ["192.168.20.18:9200"]
  setup.kibana:
    host: "192.168.20.18:5601"
  ```
  
  #### 5.2.5 重新启动filebeats，初始化模块
  
  ```
  ./filebeat setup --index-management -e -c  filebeat.yml
  ```
  
  #### 5.2.6 参考文件
  
  [beats input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-beats.html)
  
  [filebeat模块与配置](https://www.cnblogs.com/cjsblog/p/9495024.html)
  
  [system module](https://s0www0elastic0co.icopy.site/guide/en/beats/filebeat/current/filebeat-module-system.html)
  
  #### 5.3 创建index-patterns
  
  management--kibana-index patterns-Create index pattern
  
  ![image.png](https://i.loli.net/2019/12/12/qSYCVewgs5ydtZn.png)
  
  ![image.png](https://i.loli.net/2019/12/12/4vEtNfbHDSLZT2l.png)
  
  
  
  
  
  #### 5.4查看discover
  
  ![1575548396_1_.jpg](https://i.loli.net/2019/12/05/N9laMESCiQZxHkn.png)
  
  
  
  #### 5.4 查看dashboard
  
  ![image.png](https://i.loli.net/2019/12/05/mgGSpJY5wvhAtDr.png)
  
  
  
  ## 6 整合logstash配置
  
  ### 6.1 整合logstash配置文件
  
  ```
  input{
    udp {
      port => 516
      type => "h3c"
    }
  }
  input{
    udp {
      port => 518
      type => "hillstone"
    }
  }
  input{
    udp {
      port => 514
      type => "vmware"
    }
  }
  input{
    udp {
      port => 1514
      type => "vcenter"
    }
  }
  input {
    beats {
      port => 5044
      type => "windows"
    }
  }
  filter {
    if [type] == "hillstone" {
    grok {
      match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{BASE10NUM:serial}\(%{WORD:ROOT}\) %{DATA:logid}\ %{DATA:Sort}@%{DATA:Class}\: %{DATA:module}\: %{IPV4:srcip}\:%{BASE10NUM:srcport}->%{IPV4:dstip}:%{WORD:dstport}\(%{DATA:protocol}\), application %{USER:app}\, interface %{DATA:interface}\, vr %{USER:vr}\, policy %{DATA:policy}\, user %{USERNAME:user}\@%{DATA:AAAserver}\, host %{USER:HOST}\, send packets %{BASE10NUM:sendPackets}\,send bytes %{BASE10NUM:sendBytes}\,receive packets %{BASE10NUM:receivePackets}\,receive bytes %{BASE10NUM:receiveBytes}\,start time %{TIMESTAMP_ISO8601:startTime}\,close time %{TIMESTAMP_ISO8601:closeTime}\,session %{WORD:state}\,%{GREEDYDATA:reason}"}
      match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{BASE10NUM:serial}\(%{WORD:ROOT}\) %{DATA:logid}\ %{DATA:Sort}@%{DATA:Class}\: %{DATA:module}\: %{IPV4:srcip}\:%{BASE10NUM:srcport}->%{IPV4:dstip}:%{WORD:dstport}\(%{DATA:protocol}\), interface %{DATA:interface}\, vr %{DATA:vr}\, policy %{DATA:policy}\, user %{USERNAME:user}\@%{DATA:AAAserver}\, host %{USER:HOST}\, session %{WORD:state}%{GREEDYDATA:reason}"}
      match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{BASE10NUM:serial}\(%{WORD:ROOT}\) %{DATA:logid}\ %{DATA:Sort}@%{DATA:Class}\: %{DATA:module}\: %{IPV4:srcip}\:%{BASE10NUM:srcport}->%{IPV4:dstip}:%{WORD:dstport}\(%{DATA:protocol}\), %{WORD:state} to %{IPV4:snatip}\:%{BASE10NUM:snatport}\, vr\ %{DATA:vr}\, user\ %{USERNAME:user}\@%{DATA:AAAserver}\, host\ %{DATA:HOST}\, rule\ %{BASE10NUM:rule}"}                                                                                                                                                         match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{BASE10NUM:serial}\(%{WORD:ROOT}\) %{DATA:logid}\ %{DATA:Sort}@%{DATA:Class}\: %{DATA:module}\: %{IPV4:srcip}\:%{BASE10NUM:srcport}->%{IPV4:dstip}:%{WORD:dstport}\(%{DATA:protocol}\), %{WORD:state} to %{IPV4:dnatip}\:%{BASE10NUM:dnatport}\, vr\ %{DATA:vr}\, user\ %{USERNAME:user}\@%{DATA:AAAserver}\, host\ %{DATA:HOST}\, rule\ %{BASE10NUM:rule}"}
  }
    mutate {
    lowercase => [ "module" ]
    remove_field => ["host", "message", "ROOT", "HOST", "serial", "syslog_pri", "timestamp", "mac", "AAAserver", "user"]
  }
    }
    if [type] == "h3c" {
      grok {
        match => { "message" => "\<%{BASE10NUM:syslog_pri}\>%{SYSLOGTIMESTAMP:timestamp}\ %{DATA:year} %{DATA:hostname} \%\%%{DATA:ddModuleName}\/%{POSINT:severity}\/%{DATA:brief}\: %{GREEDYDATA:reason}" }
      add_field => {"severity_code" => "%{severity}"}
  }
    mutate {
      gsub => [
    "severity", "0", "Emergency",
    "severity", "1", "Alert",
    "severity", "2", "Critical",
    "severity", "3", "Error",
    "severity", "4", "Warning",
    "severity", "5", "Notice",
    "severity", "6", "Informational",
    "severity", "7", "Debug"
  ]
      remove_field => ["message", "syslog_pri"]
  }
    }
    if [type] == "vmware" {
    grok {
    break_on_match => true
    match => [
    "message", "<%{POSINT:syslog_pri}>%{TIMESTAMP_ISO8601:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{SYSLOGPROG:syslog_program}: (?<message-body>(?<message_system_info>(?:\[%{DATA:message_thread_id} %{DATA:syslog_level} \'%{DATA:message_service}\'\ ?%{DATA:message_opID}])) \[%{DATA:message_service_info}]\ (?<syslog_message>(%{GREEDYDATA})))",
    "message", "<%{POSINT:syslog_pri}>%{TIMESTAMP_ISO8601:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{SYSLOGPROG:syslog_program}: (?<message-body>(?<message_system_info>(?:\[%{DATA:message_thread_id} %{DATA:syslog_level} \'%{DATA:message_service}\'\ ?%{DATA:message_opID}])) (?<syslog_message>(%{GREEDYDATA})))",
    "message", "<%{POSINT:syslog_pri}>%{TIMESTAMP_ISO8601:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{SYSLOGPROG:syslog_program}: %{GREEDYDATA:syslog_message}"
  ]
  }
    date {
    match => [ "syslog_timestamp", "YYYY-MM-ddHH:mm:ss", "ISO8601" ]
  }
    mutate {
    replace => [ "@source_host", "%{syslog_hostname}" ]
  }
    mutate {
    replace => [ "@message", "%{syslog_message}" ]
  }
    mutate {
    remove_field => ["@source_host","program","syslog_hostname","@message"]
  }
  }
    if [type] == "vcenter" {
    grok {
    break_on_match => true
    match => [
    "message", "<%{NONNEGINT:syslog_pri}>%{NONNEGINT:syslog_ver} +(?:%{TIMESTAMP_ISO8601:syslog_timestamp}|-) +(?:%{HOSTNAME:syslog_hostname}|-) +(-|%{SYSLOG5424PRINTASCII:syslog_program}) +(-|%{SYSLOG5424PRINTASCII:syslog_proc}) +(-|%{SYSLOG5424PRINTASCII:syslog_msgid}) +(?:%{SYSLOG5424SD:syslog_sd}|-|) +%{GREEDYDATA:syslog_msg}"
  ]
  }
    date {
    match => [ "syslog_timestamp", "YYYY-MM-ddHH:mm:ss,SSS", "YYYY-MM-dd HH:mm:ss,SSS", "ISO8601" ]
  }
    mutate {
    remove_field => ["syslog_ver", "syslog_pri"]
  }
  }
  }
  output {
    if [type] == "hillstone" {
    elasticsearch {
      hosts => "192.168.20.18:9200"
      index => "hillstone-%{module}-%{+YYYY.MM.dd}"
  }
    }
    if [type] == "h3c" {
    elasticsearch {
      hosts => "192.168.20.18:9200"
      index => "h3c-%{+YYYY.MM.dd}"
  }
    }
    if [type] == "vmware" {
    elasticsearch {
      hosts => "192.168.20.18:9200"
      index => "vmware-%{+YYYY.MM.dd}"
  }
  }
    if [type] == "vcenter" {
    elasticsearch {
      hosts => "192.168.20.18:9200"
      index => "vcenter-%{+YYYY.MM.dd}"
  }
  }
    if [type] == "windows" {
    elasticsearch {
      hosts => "192.168.20.18:9200"
      index => "%{[@metadata][beat]}-%{[@metadata][version]}"
  }
  }
  }
  
  ```
  
  ### 6.2 配置logstash服务
  
  启动logstash服务，并把mix.config 更改为logstash.conf  ,放到/etc/logstash 目录下。
  
  ```
  systemctl enable logstash
  systemc	start logstash
  ```
  
  
  
  ##### 未完待续
  
  
  
  ### 推荐查看英文文档方式
  
  找到一篇英文站点，将鼠标移动到url开始，添加icopy.site/回车。
  
  ​		"icopy.site/"+"https://www.elastic.co" 
  
  例如 ： 源网址：https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html
  
  ​			转译后网址：https://s0www0elastic0co.icopy.site/guide/en/logstash/current/plugins-filters-grok.html
  
  推荐理由：
  
  ​		1.比谷歌全文翻译更准确，而且关键代码不翻译。
  
  ​		2.如果你英文不好，或者看英文文档太累，可以试下哦。



​		推荐学习视频： 

​		[ELK入门到实践](https://www.bilibili.com/video/av77345745?p=115)
