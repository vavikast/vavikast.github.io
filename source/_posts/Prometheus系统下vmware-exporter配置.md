---
title: Prometheus系统下vmware_exporter配置
catalog: true
date: 2019-07-09 14:32:03
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- vmware
- prometheus
catagories:
- vmware
- prometheus
updateDate: 2019-07-09 14:32:03
---

#### Prometheus系统下vmware_exporter配置



    为了方便管理设备，搞起了Prometheus。今天从vmware_exporter开始，监控起来我的vmware vsphere集群。

vmware_exporter由于编译问题不成功，选择使用docker方式执行。[vmware_exporter](https://github.com/pryorda/vmware_exporter)

1. ##### 编辑配置文件config.yml

   为了配置prometheus的文件发现服务，特将esx中的vsphere_hosthost删除。

   ```
   mkdir -p /data/vmware/config.yml
   vim /data/vmware/config.yml
   ```

   ```
   default:
       vsphere_host: "vcenter"
       vsphere_user: "user"
       vsphere_password: "password"
       ignore_ssl: False
       collect_only:
           vms: True
           vmguests: True
           datastores: True
           hosts: True
           snapshots: True
   
   esx:
       vsphere_user: 'root'
       vsphere_password: 'password'
       ignore_ssl: True
       collect_only:
           vms: True
           vmguests: True
           datastores: True
           hosts: True
           snapshots: True
   
   
   ```

2. ##### 执行vmware-exporter  docker程序。

   ```
   docker run -d   -v /data/vmware:/data/ -it --rm  -p 9272:9272 --name vmware_exporter pryorda/vmware_exporter   -c /data/config.yml
   ```

3. ##### 编译prometheus配置文件

   1. 编辑esx.yml文件，以便prometheus中sd_file配置文件使用。

      ```
      vim /etc/prometheus/esx.yml
      ```

      ```
      ---
      - targets:
        - 192.168.21.91
        - 192.168.21.92
        - 192.168.21.93
        - 192.168.21.95
        - 192.168.21.96
        labels:
          job: vmware_esx
      
      ```

      2.编辑prometheus文件，其中section：[esx] 对应config.yml文件中的ESX片段。

      ```
      vim /etc/prometheus/prometheus.yml
      ```

      ```
        - job_name: 'vmware_vcenter'
          metrics_path: '/metrics'
          static_configs:
            - targets:
              - 'vcenter.company.com'
          relabel_configs:
            - source_labels: [__address__]
              target_label: __param_target
            - source_labels: [__param_target]
              target_label: instance
            - target_label: __address__
              replacement: localhost:9272
      
        - job_name: 'vmware_esx'
          metrics_path: '/metrics'
          file_sd_configs:
            - files:
              - /etc/prometheus/esx.yml
          params:
            section: [esx]
          relabel_configs:
            - source_labels: [__address__]
              target_label: __param_target
            - source_labels: [__param_target]
              target_label: instance
            - target_label: __address__
              replacement: localhost:9272
      
      ```

4. ##### 添加Grafana监控，导入granafa的配置文件，配置文件默认在dashboard目录下。

   [导入dashboard配置文件](https://grafana.com/docs/reference/export_import/)

参考链接：

[vmware_exporter](https://grafana.com/docs/reference/export_import/)

   终于完成了一小步，但是总归是完成了，还有很多问题需要进一步解决，比如告警，比如更精细的监控，等等。

    我的prometheus系列逐步完善中，未完待续，加油！