# 介绍

略



## 监控告警架构

目前鲁班产品采用Prometheus的整体架构，同时也参照kingsoft cloud的eagles监控架构修改，eagles目前采用架构为：



![image-20210324200100002](introduction.assets/image-20210324200100002.png)

prometheus的原架构：

<img src="Untitled.assets/image-20210324200315142.png" alt="image-20210324200315142" style="zoom:50%;" />



目前鲁班架构增加luban_agent接受原有的eagles的自定义监控脚本，增加node_exporter物理机监控，增加告警历史存储到elasticsearch以及实时告警处理推送飞书，具体架构如下：

<img src="introduction.assets/image-20210522152828743.png" alt="image-20210522152828743" style="zoom:50%;" />



## 日志架构

在日志收集中，都是使用的filebeat+ELK的日志架构。但是如果业务每天会产生海量的日志，就有可能引发logstash和elasticsearch的性能瓶颈问题。因此改善这一问题的方法就是filebeat+kafka+logstash+ELK，
也就是将存储从elasticsearch转移给消息中间件，减少海量数据引起的宕机，降低elasticsearch的压力，这里的elasticsearch主要进行数据的分析处理，然后交给kibana进行界面展示

<img src="introduction.assets/image-20210522152858428.png" alt="image-20210522152858428" style="zoom:50%;" />





## CMDB架构

Dgraph组件包括三个部分：   

Zero: 是集群的核心, 负责调度集群服务器和平衡服务器组之间的数据，类比于Elasticsearch的master节点；
Alpha: 保存数据的 谓词 和 索引. 谓词包括数据的 属性 和数据之间的 关系; 索引是为了更快的进行数据的过滤和查找，类比于Elasticsearch的data节点；
Ratel: dgraph 的 UI 接口, 可以在此界面上进行数据的 CURD, 也可以修改数据的 schema，类比于Elasticsearch的kibana角色

<img src="introduction.assets/image-20210522155547554.png" alt="image-20210522155547554" style="zoom:50%;" />





# 安装步骤

## 环境准备

目前鲁班的组件（除监控数据采集端的node_exporter以及luban_agent是部署在物理机服务器上之外，其他组件均部署在kubernetes集群中，安装鲁班组件之前需要初始化一个高可用的kubernetes集群，集群最低配置如下，计算资源与存储资源根据监控数据量以及存储数据量适当加大。

| 角色                                      | 服务器名称    | 主机名  | 计算资源 | 磁盘（见存储资源规划）                                       | 推荐网卡 | 操作系统  | 描述     | 网络类型                     | 备注        |
| ----------------------------------------- | ------------- | ------- | -------- | ------------------------------------------------------------ | -------- | --------- | -------- | ---------------------------- | ----------- |
| 管理节点（3台做高可用，虚拟机或者物理机） | Master1       | x.x.x.x | 8C16G    | 1、每台虚机系统盘，20G， 关闭swap          2、每台虚机单独加一块（裸）数据盘，100-500G，作为Docker VG          3、每台虚机单独加一块数据盘，50G，作为ETCD数据存储     挂载点：/var/lib/etcd      文件系统： xfs | 数量：1  | Centos7.x | 管理节点 | 物理网络或VLAN类型的虚拟网络 |             |
| 管理节点（3台做高可用，虚拟机或者物理机） | Master2       | x.x.x.x | 8C16G    | 同上                                                         | 数量：1  | Centos7.x | 管理节点 | 同上                         |             |
| 管理节点（3台做高可用，虚拟机或者物理机） | Master3       | x.x.x.x | 8C16G    | 同上                                                         | 数量：1  | Centos7.x | 管理节点 | 同上                         |             |
| 计算节点                                  | Worker1       | x.x.x.x | 8C32G    | 1、每台虚机系统盘，20G， 关闭swap          2、每台虚机单独加一块（裸）数据盘，100G-500G，作为Docker VG | 数量：1  | Centos7.x | 计算节点 | 同上                         |             |
| 计算节点                                  | Worker2       | x.x.x.x | 8C32G    | 同上                                                         | 数量：1  | Centos7.x | 计算节点 | 同上                         |             |
| 计算节点                                  | Worker....... | x.x.x.x | 8C32G    | 同上                                                         | 数量：1  | Centos7.x | 计算节点 | 同上                         |             |
| 镜像仓库（虚拟机）                        | Registry1     | x.x.x.x | 4C8G     | 同上                                                         | 数量：1  | Centos7.x | 仓库     | 同上                         |             |
| 镜像仓库（虚拟机）                        | Registr2      | x.x.x.x | 4C8G     | 同上                                                         | 数量：1  | Centos7.x | 仓库     | 同上                         |             |
| MasterLb1     Master节点负载均衡          | MasterLb1     | x.x.x.x | 4C8G     | 同上                                                         | 数量：1  | Centos7.x | 负载均衡 | 同上                         | 或者使用slb |
| MasterLb2     Master节点负载均衡          | MasterLb2     | x.x.x.x | 4C8G     | 同上                                                         | 数量：1  | Centos7.x | 负载均衡 | 同上                         | 或者使用slb |
| Registry节点负载均衡                      | RouterLb1     | x.x.x.x | 4C8G     | 同上                                                         | 数量：1  | Centos7.x | 负载均衡 | 同上                         | 或者使用slb |
| Registry节点负载均衡                      | RouterLb2     | x.x.x.x | 4C8G     | 同上                                                         | 数量：1  | Centos7.x | 负载均衡 | 同上                         | 或者使用slb |

## 安装Kubernetes

### 环境信息

| 主机名  | IP地址      |
| :------ | :---------- |
| master0 | 192.168.0.2 |
| master1 | 192.168.0.3 |
| master2 | 192.168.0.4 |
| node0   | 192.168.0.5 |

服务器密码：123456

### 高可用安装教程

只需要准备好服务器，在任意一台服务器上执行下面命令即可

```sh
# 下载并安装sealos, sealos是个golang的二进制工具，直接下载拷贝到bin目录即可, release页面也可下载
wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/latest/sealos && \
    chmod +x sealos && mv sealos /usr/bin 

# 下载离线资源包
wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/2fb10b1396f8c6674355fcc14a8cda7c-v1.20.0/kube1.20.0.tar.gz

# 安装一个三master的kubernetes集群
$ sealos init --passwd '123456' \
    --master 192.168.0.2  --master 192.168.0.3  --master 192.168.0.4 \ 
    --node 192.168.0.5 \
    --pkg-url /root/kube1.20.0.tar.gz \
    --version v1.20.0
```

参数含义

| 参数名  | 含义                                                         | 示例                    |
| :------ | :----------------------------------------------------------- | :---------------------- |
| passwd  | 服务器密码                                                   | 123456                  |
| master  | k8s master节点IP地址                                         | 192.168.0.2             |
| node    | k8s node节点IP地址                                           | 192.168.0.3             |
| pkg-url | 离线资源包地址，支持下载到本地，或者一个远程地址             | /root/kube1.20.0.tar.gz |
| version | [资源包](https://www.sealyun.com/goodsDetail?type=cloud_kernel&name=kubernetes)对应的版本 | v1.20.0                 |

增加master

```shell
sealos join --master 192.168.0.6 --master 192.168.0.7
sealos join --master 192.168.0.6-192.168.0.9  # 或者多个连续IP
```

增加node

```shell
sealos join --node 192.168.0.6 --node 192.168.0.7
sealos join --node 192.168.0.6-192.168.0.9  # 或者多个连续IP
```

删除指定master节点

```shell
sealos clean --master 192.168.0.6 --master 192.168.0.7
sealos clean --master 192.168.0.6-192.168.0.9  # 或者多个连续IP
```

删除指定node节点

```shell
sealos clean --node 192.168.0.6 --node 192.168.0.7
sealos clean --node 192.168.0.6-192.168.0.9  # 或者多个连续IP
```

清理集群

```shell
sealos clean --all
```

备份集群

```shell
sealos etcd save
```



## 安装服务网格

### 安装Istio

istio用于管理银河的服务，服务之间的权限设置。下载istioctl到安装环境 执行安装命令

离线镜像列表

```
docker.io/istio/pilot:1.9.1
docker.io/istio/proxyv2:1.9.1
```

同步镜像

```shell
for i in `cat image.txt`;do docker pull $i;done 
for i in `cat image.txt`;do docker tag $i harbor.inner.galaxy.ksyun.com/istio/${i##*/};done
for i in `cat image.txt`;do docker push harbor.inner.galaxy.ksyun.com/istio/${i##*/};done
```

安装

```
istioctl install --set profile=demo -y --set values.global.hub="harbor.inner.galaxy.ksyun.com/istio"
```



## 安装监控告警组件

### 安装Prometheus+Alertmanager

```shell
kubectl create ns monitoring
rm -rf custom-values.yaml
export REGISTRY="harbor.inner.galaxy.ksyun.com"
cat <<EOF > custom-values.yaml
prometheusOperator:
  image:
    repository: $REGISTRY/luban/prometheus-operator
    tag: v0.45.0
  prometheusConfigReloaderImage:
    repository: $REGISTRY/luban/prometheus-config-reloader
    tag: v0.45.0
  admissionWebhooks:
     patch:
       image:
         repository: $REGISTRY/luban/kube-webhook-certgen
         tag: v1.5.0

alertmanager:
  alertmanagerSpec:
    image:
      repository: $REGISTRY/luban/alertmanager
      tag: v0.21.0
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname']
      group_wait: 30s
      group_interval: 1m
      repeat_interval: 1m

prometheus:
  prometheusSpec:
    image:
      repository: $REGISTRY/luban/prometheus
      tag: v2.24.0
kube-state-metrics:
  image:
    repository: $REGISTRY/luban/kube-state-metrics
    tag: v1.9.8

grafana:
  image:
    repository: $REGISTRY/luban/grafana
    tag: 7.4.2
  sidecar:
    image:
      repository: $REGISTRY/luban/k8s-sidecar
      tag: 1.10.6
  adminPassword: Kingsoft123

prometheus-node-exporter:
  image:
    repository: $REGISTRY/luban/node-exporter
    tag: v1.0.1
EOF

helm install pm ./kube-prometheus-stack-13.13.1.tgz -n monitoring -f custom-values.yaml
```



### 安装鲁班Agent

目前鲁班的版本发布放在青岛5服务器，后续会跟银河产品一下打包发布。

centos65版本物理机服务器

```shell
#!/bin/sh

export agent_http_server="http://10.177.152.168:8888/luban/luban_on_k8s/agent/luban_agent"
export bin_dir="/opt/luban/bin"
mkdir -p $bin_dir
ps -aux |grep luban_agent|grep -v grep|awk '{print $2}'|xargs kill -9
rm -rf $bin_dir/luban_agent
curl -o $bin_dir/luban_agent $agent_http_server
chmod a+x $bin_dir/luban_agent
rm -rf /var/log/luban_agent.log

cat > /opt/luban/bin/start_agent.sh << EOF
nohup $bin_dir/luban_agent >/var/log/luban_agent.log 2>&1 &
EOF
chmod a+x /opt/luban/bin/start_agent.sh 

cat > /opt/luban/bin/stop_agent.sh << EOF
ps -aux |grep luban_agent|grep -v grep|awk '{print $2}'|xargs kill -9
EOF
chmod a+x /opt/luban/bin/stop_agent.sh 

cat > /etc/init.d/luban_agent << EOF
#!/bin/bash
#
# /etc/rc.d/init.d/luban_agent
#
#  Luban Agent 
#
#  description: Luban Agent
#  processname: luban_agent
#  chkconfig: - 85 15

# Source function library.
. /etc/rc.d/init.d/functions

PROGNAME=luban_agent
PROG=/opt/luban/bin/\$PROGNAME
USER=root
LOGFILE=/var/log/luban_agent.log
LOCKFILE=/var/run/\$PROGNAME.pid

start() {
    echo -n "Starting \$PROGNAME: "
    daemon --user $USER --pidfile="\$LOCKFILE" "\$PROG &>\$LOGFILE &"
    echo
}

stop() {
    echo -n "Shutting down $PROGNAME: "
    kill -9 \`pidof \$PROGNAME\`
    rm -f $LOCKFILE
    echo
}


case "\$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status $PROGNAME
        ;;
    restart)
        stop
        start
        ;;
    *)
        echo "Usage: service luban_agent {start|stop|status|restart}"
        exit 1
        ;;
esac
EOF
chmod a+x /etc/init.d/luban_agent
service luban_agent start
chkconfig luban_agent on
```

centos73的物理服务器

```shell
#!/bin/sh

export http_server="http://10.177.9.11:8888/luban/"
export bin_dir="/opt/luban/bin"
mkdir -p $bin_dir
rm -rf $bin_dir/*
curl -o $bin_dir/luban_agent $http_server/luban_agent
chmod a+x $bin_dir/*
rm -rf /var/log/luban_agent.log

cat > /opt/luban/bin/start_agent.sh << EOF
nohup $bin_dir/luban_agent >/var/log/luban_agent.log 2>&1 &
EOF
chmod a+x /opt/luban/bin/start_agent.sh 

cat > /opt/luban/bin/stop_agent.sh << EOF
ps -aux |grep luban_agent|grep -v grep|awk '{print $2}'|xargs kill -9
EOF
chmod a+x /opt/luban/bin/stop_agent.sh 

cat > /usr/lib/systemd/system/luban_agent.service  << EOF
[Unit]
Description=Luban agent
After=network.target

[Service]
ExecStart=/opt/luban/bin/luban_agent > /var/log/luban_agent.log
ExecStop=kill -9 `pidof luban_agent`
Type=notify
User=root
Group=root

[Install]
WantedBy=multi-user.target

EOF
systemctl enable luban_agent 
systemctl start luban_agent 
```



### 安装Node_exporter

Node_exporter负责监控物理机服务器以及部署银河服务的虚拟机监控信息，监控主要包括cpu，内存，磁盘，网络

安装步骤

```shell
#!/bin/sh
export node_exporter_url="http://10.177.152.168:8888/luban/luban_on_k8s/install/node_exporter/node_exporter"
mkdir -p /opt/luban/bin/
export node_exporter_binary="/opt/luban/bin/node_exporter"
rm -rf $node_exporter_binary
curl -o $node_exporter_binary ${node_exporter_url}

chmod a+x $node_exporter_binary
ps -aux |grep $node_exporter_binary|awk '{print $2}'|xargs kill -9
nohup $node_exporter_binary --collector.luban >/var/log/node_exporter.log 2>&1 &
```



### 安装Process_exporter

Process_exporter负责监控物理机服务器以及部署银河服务的虚拟机监控信息，监控主要包括进程监控

安装步骤

```shell
#!/bin/sh
export process_exporter_url="http://10.177.152.168:8888/luban/luban_on_k8s/install/process_exporter/process-exporter"
mkdir -p /opt/luban/bin/
mkdir -p /opt/luban/conf/
export process_exporter_binary="/opt/luban/bin/process-exporter"

rm -rf $process_exporter_binary
curl -o $process_exporter_binary ${process_exporter_url}
chmod a+x $process_exporter_binary
ps -aux |grep $process_exporter_binary |grep -v grep|awk '{print $2}'|xargs kill -9

if [ ! -f "/opt/luban/conf/process-name.yaml" ];then
  echo "/opt/luban/conf/process-name.yaml not exist"
else
  cp -rf /opt/luban/conf/process-name.yaml /opt/luban/conf/process-name.yaml.bak
fi
cat > /opt/luban/conf/process-name.yaml <<EOF
process_names:
  - name: "{
   {.Comm}}"
    cmdline:
    - '.+'
EOF

nohup $process_exporter_binary -config.path=/opt/luban/conf/process-name.yaml >/var/log/process_exporter.log 2>&1 &

```



### 安装Blackbox_exporter

Blackbox_exporter负责监控Ping，DNS，Telnet等监控

安装步骤

```shell
#!/bin/sh

kubectl apply -f blackbox_alert.yaml -f blackbox_exporter.yaml -f blackbox_monitor.yaml -n monitoring
```



### 安装Cluster_exporter

Cluster_exporter负责监控Ebs,Ks3等存储总量信息以及使用量信息等监控

安装步骤

```shell
#!/bin/sh

kubectl apply -f cluster_exporter.yaml -n monitoring
```





```
## 安装agent,以下10.177.9.1 物理服务器为例，ssh登录服务器,10.177.9.11为鲁班的http文件服务器
export http_server="http://10.177.9.11:8888/luban/"
export bin_dir="/opt/luban/bin"
mkdir -p $bin_dir
rm -rf $bin_dir/*
curl -o $bin_dir/node_exporter http://10.177.9.11:8888/luban/node_exporter
curl -o $bin_dir/luban_agent http://10.177.9.11:8888/luban/luban_agent
chmod a+x $bin_dir/*
 
## restart node_exporter
ps -aux |grep node_exporter|grep -v grep|awk '{print $2}'|xargs kill -9
echo "node_exporter shutdown [ok]"
 
## restart luban_agent
ps -aux |grep luban_agent|grep -v grep|awk '{print $2}'|xargs kill -9
echo "luban_agent shutdown [ok]"
 
## start node_exporter and luban_agent
nohup $bin_dir/node_exporter --collector.luban >/var/log/node_exporter.log 2>&1 &
echo "node_exporter start [ok]"
nohup $bin_dir/luban_agent >/var/log/luban_agent.log 2>&1 &
echo "luban_agent start [ok]"
```

安装完成后检查 node_exporter 以及luban_agent 是否正确运行

```
ps aux |grrep node_exporter
ps aux |grrep luban_agent
```

### 安装自定义监控脚本

此步骤由运维同学安装底层银河环境时候，自动化安装，自定义监控脚本存放位置 http://newgit.op.ksyun.com/galaxy_cloud/monitor_scripts/tree/master/sdn_monitor_shell，由运维同学自动化安装，以下示例供研发同学参考

以下mem_monitor.sh为例，脚本内容如下：

```
#!/bin/bash
# 内存监控脚本
 
source /etc/profile
 
function send_data(){
        ts=`date +%s`;
        metric=$1;
        endpoint=$4;
        value=$2;
        tags=$3
        curl -X POST -d "[{\"metric\": \"$metric\", \"endpoint\": \"$endpoint\", \"timestamp\": $ts,\"step\": 60,\"value\": $value,\"counterType\": \"GAUGE\",\"tags\": \"$tags\"}]" http://127.0.0.1:1988/v1/push
}
 
item='mem_check'
item1='crash_check'
day=`date +%d`
tag1='metric=crash_status'
tag2='metric=/var/log/mcelog'
tag3='metric=/var/log/messages'
tag4='metric=/var/log/dmesg'
now_time=`date +%F`
hostn=`hostname`
 
    # 判断是否有crash日志
function crashlog(){
    crash_log=`ls /var/crash/|grep "$now_time" |wc -l`
    if [ $crash_log -eq 0 ];then
        crash_status=1
    else
        crash_status=0
    fi
    echo $item $crash_status $tag1 $hostn crash
    send_data $item1 $crash_status $tag1 $hostn crash
}
 
    # 判断是否有内存日志文件
function mcelog(){
    test -a /var/log/mcelog
    if [ $? -eq 0 ];then
    counts=`cat /var/log/mcelog|wc -l`
    if [ $counts -eq 0 ];then
        mcelog_status=1
    else
        mcelog_status=0
    fi
    else
        mcelog_status=1
    fi
    echo $item $mcelog_status $tag2 $hostn mcelog
    send_data $item $mcelog_status $tag2 $hostn mcelog
}
 
    # 判断日志
function messagelog (){
    message_log=`tail -n500 /var/log/messages|egrep -i 'err|fail' |egrep -i "memory" |egrep -v 'nova.compute.manager'|wc -l`
    if [ $message_log -gt 1 ];then
        mem_status=0
    else
        mem_status=1
    fi
    echo $item $mem_status $tag3 $hostn message
    send_data $item $mem_status $tag3 $hostn message
}
 
    # 判断日志
function dmesglog (){
    dme_log1=`tail -n500 /var/log/dmesg|egrep -i "err|fail" |egrep -i "memory" |wc -l`
    #dme_log2=`ssh $i dmesg |tail -n200 |egrep -i 'error|fail'|wc -l`
    if [ $dme_log1 -gt 1 ];then
        dme_status=0
    else
        dme_status=1
    fi
    echo $item $dme_status $tag4 $hostn dmesg
    send_data $item $dme_status $tag4 $hostn dmesg
}
crashlog
mcelog
messagelog
dmesglog
```

配置物理服务器上的crontab，如下图所示

```sh
cat /etc/crontab
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
HOME=/
 
# For details see man 4 crontabs
 
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
*/1  *  *  *  * root bash /opt/luban/script/mem_monitor.sh
```

## 安装日志组件

目前日志的组件采用filebeat+kafka+logstash+ELK架构,参照http://ezone.ksyun.com/ezCode/luban/luban_on_k8s/tree中的logging文件夹中的README.md

需要离线镜像列表

```
harbor.inner.galaxy.ksyun.com/luban/elasticsearch:7.10.1
harbor.inner.galaxy.ksyun.com/luban/kibana:7.10.1
harbor.inner.galaxy.ksyun.com/luban/filebeat:7.10.1
harbor.inner.galaxy.ksyun.com/luban/kafka:2.7.0-debian-10-r1
harbor.inner.galaxy.ksyun.com/luban/zookeeper:3.6.2-debian-10-r89
harbor.inner.galaxy.ksyun.com/luban/logstash:7.10.1
```

#### 安装Elasticsearch

```
kubectl create ns elastic-system
export REGISTRY="harbor.inner.galaxy.ksyun.com"
helm install es ./elasticsearch-7.10.1.tgz -n elastic-system --set imageTag=7.10.1 --set persistence.enabled=false --set image=$REGISTRY/luban/elasticsearch
```

#### 安装Kibana

```
kubectl create ns elastic-system
export REGISTRY="harbor.inner.galaxy.ksyun.com"
helm install kibana -n elastic-system ./kibana-7.10.1.tgz  --set imageTag=7.10.1 --set image=$REGISTRY/luban/kibana
```

#### 安装Kafka

注意kafka集群需要集群外访问权限，最好增加eip的slb绑定kubernetes集群的三个master节点

```
kubectl create ns elastic-system
export REGISTRY="harbor.inner.galaxy.ksyun.com"
helm install kafka ./kafka-12.4.2.tgz -n elastic-system  --set persistence.enabled=false  --set image.registry=$REGISTRY --set image.repository=luban/kafka --set image.tag=2.7.0-debian-10-r1 --set zookeeper.image.registry=$REGISTRY --set zookeeper.image.repository=luban/zookeeper --set zookeeper.image.tag=3.6.2-debian-10-r89 --set zookeeper.persistence.enabled=false --set replicaCount=3 \
--set externalAccess.enabled=true \
--set externalAccess.service.type=NodePort \
--set externalAccess.autoDiscovery.enabled=false \
--set serviceAccount.create=true \
--set rbac.create=true \
--set externalAccess.service.nodePorts='{30092,30093,30094}' \
--set externalAccess.service.domain=10.177.152.168 ##3*master节点的的slb的eip
```

#### 安装Filebeat（集群内部，用于收集容器日志，直接上传es集群）

```
kubectl create ns elastic-system
export REGISTRY="harbor.inner.galaxy.ksyun.com"
helm install filebeat ./filebeat-7.10.1.tgz  -n elastic-system  --set imageTag=7.10.1 --set  image=$REGISTRY/luban/filebeat
```

#### 安装Filebeat（集群外部，用于收集服务器或者其他物理设备日志，写入kafka）

```
rpm -ivh filebeat-7.10.1-x86_64.rpm
```

修改/etc/filebeat/filebeat.yml配置，官方地址：https://www.elastic.co/guide/en/beats/filebeat/current/configuring-howto-filebeat.html

#### 安装Logstash

注意修改logstash的配置文件values.yaml

```
kubectl create ns elastic-system
export REGISTRY="harbor.inner.galaxy.ksyun.com"
helm install logstash ./logstash -n elastic-system  --set imageTag=7.10.1 --set image=$REGISTRY/luban/logstash
```

#### 安装自动清理Curator

```
helm install curator ./elasticsearch-curator -n elastic-system --set image.repository=harbor.inner.galaxy.ksyun.com/luban/curator --set image.tag=5.7.6
```



以下是自动化脚本

```
kubectl create ns elastic-system
export REGISTRY="harbor.inner.galaxy.ksyun.com"

helm install es ./elasticsearch-7.10.1.tgz -n elastic-system --set imageTag=7.10.1 --set persistence.enabled=false --set image=$REGISTRY/luban/elasticsearch 

helm install kibana -n elastic-system ./kibana-7.10.1.tgz  --set imageTag=7.10.1 --set image=$REGISTRY/luban/kibana

helm install filebeat ./filebeat-7.10.1.tgz  -n elastic-system  --set imageTag=7.10.1 --set  image=$REGISTRY/luban/filebeat

## 注意修改kafka的externalAccess.service.domain，用于其他物理服务器filebeat配置，10.177.152.168应为K8s的Master节点负载均衡IP
helm install kafka ./kafka-12.4.2.tgz -n elastic-system  --set persistence.enabled=false  --set image.registry=$REGISTRY --set image.repository=luban/kafka --set image.tag=2.7.0-debian-10-r1 --set zookeeper.image.registry=$REGISTRY --set zookeeper.image.repository=luban/zookeeper --set zookeeper.image.tag=3.6.2-debian-10-r89 --set zookeeper.persistence.enabled=false --set replicaCount=3 \
--set externalAccess.enabled=true \
--set externalAccess.service.type=NodePort \
--set externalAccess.autoDiscovery.enabled=false \
--set serviceAccount.create=true \
--set rbac.create=true \
--set externalAccess.service.nodePorts='{30092,30093,30094}' \
--set externalAccess.service.domain=10.177.152.168

helm install logstash ./logstash -n elastic-system  --set imageTag=7.10.1 --set image=$REGISTRY/luban/logstash

helm install curator ./elasticsearch-curator -n elastic-system --set image.repository=$REGISTRY/luban/curator --set image.tag=5.7.6
```

## 安装Cmdb组件

参照aws的dgraph HA架构，原文地址：

https://aws.amazon.com/cn/blogs/opensource/dgraph-on-aws-setting-up-a-horizontally-scalable-graph-database/

```shell
MANIFEST="https://raw.githubusercontent.com/dgraph-io/dgraph/master/contrib/config/kubernetes/dgraph-ha/dgraph-ha.yaml"

kubectl apply --filename $MANIFEST
```



## 安装认证服务

### 安装Dex

开源项目dex，一个基于OpenID Connect的身份服务组件。 CoreOS已经将它用于生产环境，用户认证和授权是应用安全的一个重要部分，用户身份管理本身也是一个特别专业和复杂的问题，尤其对于企业应用而言， 安全的进行认证和授权是必选项，dex无疑是解决这一问题的一大利器。

 ```shell
kubectl create ns dex
kubectl -n dex delete secret dex.example.com.tls
kubectl -n dex create secret tls dex.example.com.tls --cert=./ssl/cert.pem --key=./ssl/key.pem

cat <<EOF | kubectl -n dex apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: dex
  name: dex
  namespace: dex
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dex
  template:
    metadata:
      labels:
        app: dex
    spec:
      serviceAccountName: dex # This is created below
      containers:
      - image: harbor.inner.galaxy.ksyun.com/luban/dex:v2.27.0 #or quay.io/dexidp/dex:v2.26.0
        name: dex
        command: ["/usr/local/bin/dex", "serve", "/etc/dex/cfg/config.yaml"]

        ports:
        - name: https
          containerPort: 5556

        volumeMounts:
        - name: config
          mountPath: /etc/dex/cfg
        - name: tls
          mountPath: /etc/dex/tls
        - name: data
          mountPath: /data
      volumes:
      - name: config
        configMap:
          name: dex
          items:
          - key: config.yaml
            path: config.yaml
      - name: tls
        secret:
          secretName: dex.example.com.tls
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: dex
  namespace: dex
data:
  config.yaml: |
    issuer: https://luban.dex.galaxy.cloud/dex
    storage:
      type: memory
    logger:
      level: debug
      format: text
    web:
      http: 0.0.0.0:5556
    expiry:
      signingKeys: "6h"
      idTokens: "24h"
    connectors:
    - type: ldap
      name: ActiveDirectory
      id: ladp
      config:
        host: openldap:389
        insecureNoSSL: true
        insecureSkipVerify: true
        bindDN: cn=admin,dc=galaxy,dc=cloud
        bindPW: Kingsoft123
        usernamePrompt: Email Address
        userSearch:
          baseDN: ou=People,dc=galaxy,dc=cloud
          filter: "(objectClass=inetOrgPerson)"
          username: mail
          idAttr: uid
          emailAttr: mail
          nameAttr: uid
        groupSearch:
          baseDN: ou=Groups,dc=galaxy,dc=cloud
          filter: "(objectClass=groupOfUniqueNames)"
          userAttr: uid
          groupAttr: memberUid
          nameAttr: cn
    oauth2:
      skipApprovalScreen: true
    logger:
      level: "debug"
      format: text
    staticClients:
    - id: oidc-auth-client
      redirectURIs:
      - 'https://luban.auth.galaxy.cloud/callback'
      - 'https://luban.idp.galaxy.cloud/idp/v1/token/callback'
      - 'http://localhost/idp/v1/token/callback'
      name: 'oidc-auth-client'
      secret: ZXhhbXBsZS1hcHAtc2VjcmV0
---
apiVersion: v1
kind: Service
metadata:
  name: dex
  namespace: dex
spec:
  ports:
  - name: dex
    port: 5556
    protocol: TCP
    targetPort: 5556
  selector:
    app: dex
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app: dex
  name: dex
  namespace: dex
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dex
rules:
- apiGroups: ["dex.coreos.com"] # API group created by dex
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["create"] # To manage its own resources, dex must be able to create customresourcedefinitions
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dex
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dex
subjects:
- kind: ServiceAccount
  name: dex           # Service account assigned to the dex pod, created above
  namespace: dex  # The namespace dex is running in
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: authcodes.dex.coreos.com
spec:
  group: dex.coreos.com
  names:
    kind: AuthCode
    listKind: AuthCodeList
    plural: authcodes
    singular: authcode
  scope: Namespaced
  version: v1
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: dex
rules:
- apiGroups: ["dex.coreos.com"] # API group created by dex
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["create"] # To manage its own resources identity must be able to create customresourcedefinitions.
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: dex
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dex
subjects:
- kind: ServiceAccount
  name: dex                 # Service account assigned to the dex pod.
  namespace: dex
EOF
​```
 ```

### 安装Ldap

Open-ldap

```shell
kubectl create ns dex
helm install openldap ./openldap-1.2.7.tgz -f online-values.yaml -n dex
```

```shell
## get password
$ kubectl -n dex get secret openldap -o jsonpath="{.data.LDAP_ADMIN_PASSWORD}" | base64 --decode; echo
$ kubectl -n dex get secret openldap -o jsonpath="{.data.LDAP_CONFIG_PASSWORD}" | base64 --decode; echo


# node add ldap label
$ kubectl label nodes ip-10-178-224-65 luban/ldap="" --overwrite



ldapsearch -x -H ldap://openldap:389 \
                -b dc=galaxy,dc=cloud \
                -D "cn=admin,dc=galaxy,dc=cloud" \
                -w Kingsoft123
```

### 配置Kubernetes认证服务

```
## set kubernetes apiserver 

- --oidc-issuer-url=https://luban.dex.galaxy.cloud/dex
- --oidc-client-id=oidc-auth-client
- --oidc-ca-file=/etc/kubernetes/ssl/dex-ca.pem
- --oidc-username-claim=email
- --oidc-groups-claim=groups

```



## 安装鲁班服务

### 安装鲁班Server

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: server
  name: server
  namespace: luban
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: server
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: server
    spec:
      containers:
      - image: harbor.inner.galaxy.ksyun.com/luban/server
        imagePullPolicy: Always
        name: server
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: server
  name: server
  namespace: luban
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: server
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: server-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "luban.server.galaxy.cloud"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: server
spec:
  hosts:
    - "luban.server.galaxy.cloud"
  gateways:
    - server-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            port:
              number: 80
            host: server
      corsPolicy:
        allowOrigins:
          - exact: "*"
        allowMethods:
          - GET
          - POST
          - PATCH
          - PUT
          - DELETE
          - OPTIONS
        allowCredentials: false
        allowHeaders:
          - authorization
        maxAge: "24h"
```

### 安装鲁班Idp

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: idp
  name: idp
  namespace: luban
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: idp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: idp
    spec:
      containers:
      - args:
        - --tls_cert=
        - --tls_key=
        - --domain=luban.idp.galaxy.cloud
        - --ldap_base=dc=galaxy,dc=cloud
        - --ldap_bind_password=Kingsoft123
        - --ldap_bind_user=admin
        - --ldap_group_ou=Groups
        - --ldap_host=ldap-openldap.dex
        - --ldap_port=389
        - --ldap_user_ou=People
        - --listin_addr=0.0.0.0
        - --listin_port=80
        - --oauth_client_id=oidc-auth-client
        - --oauth_client_secrect=ZXhhbXBsZS1hcHAtc2VjcmV0
        - --oauth_redirect_url=http://localhost/idp/v1/token/callback
        - --oauth_scopes=openid,profile,email,offline_access,groups
        - --oauth_token_url=http://dex.dex:5556/dex/token
        - --oauth_url=http://dex.dex:5556/dex/auth
        command:
        - /app
        image: harbor.inner.galaxy.ksyun.com/luban/idp
        imagePullPolicy: Always
        name: server
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: idp
  name: idp
  namespace: luban
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: idp
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: idp
  namespace: luban
spec:
  gateways:
  - idp-gateway
  hosts:
  - luban.idp.galaxy.cloud
  http:
  - corsPolicy:
      allowCredentials: false
      allowHeaders:
      - authorization
      allowMethods:
      - GET
      - POST
      - PATCH
      - PUT
      - DELETE
      - OPTIONS
      allowOrigins:
      - exact: '*'
      maxAge: 24h
    route:
    - destination:
        host: idp
---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: idp-gateway
  namespace: luban
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - luban.idp.galaxy.cloud
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
  - hosts:
    - luban.idp.galaxy.cloud
    port:
      name: http
      number: 80
      protocol: HTTP

```



### 安装鲁班CmdbApi

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: cmdb
  name: cmdb
  namespace: luban
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: cmdb
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: cmdb
    spec:
      containers:
      - env:
        - name: OS_USERNAME
          value: admin
        - name: OS_TENANT_NAME
          value: admin
        - name: OS_PASSWORD
          value: ksc
        - name: OS_AUTH_URL
          value: http://10.177.147.1:35357/v2.0
        - name: OS_REGION_NAME
          value: SHPBSRegionOne
        image: harbor.inner.galaxy.ksyun.com/luban/cmdb-api
        imagePullPolicy: Always
        name: cmdb
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cmdb
  name: cmdb
  namespace: luban
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: cmdb
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cmdb-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "luban.cmdb.galaxy.cloud"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: cmdb
spec:
  hosts:
    - "luban.cmdb.galaxy.cloud"
  gateways:
    - cmdb-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            port:
              number: 80
            host: cmdb
      corsPolicy:
        allowOrigins:
          - exact: "*"
        allowMethods:
          - GET
          - POST
          - PATCH
          - PUT
          - DELETE
          - OPTIONS
        allowCredentials: false
        allowHeaders:
          - authorization
        maxAge: "24h"
```



### 安装鲁班Swagger

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: swagger-ui
  name: swagger-ui
  namespace: luban
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: swagger-ui
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: swagger-ui
    spec:
      containers:
      - image: harbor.inner.galaxy.ksyun.com/luban/swagger-ui
        imagePullPolicy: Always
        name: swagger-ui
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: swagger-ui
  name: swagger-ui
  namespace: luban
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: swagger-ui
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: swagger-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "luban.swagger.galaxy.cloud"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: swagger
spec:
  hosts:
    - "luban.swagger.galaxy.cloud"
  gateways:
    - swagger-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            port:
              number: 8080
            host: swagger-ui
```



```
#!/bin/sh
kubectl apply -f cmdb-api.yaml -f luban-idp.yaml -f luban-server.yaml -f swagger.yaml -n luban
```



### 安装告警对接ElasticSearch

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: alertmanager-warning
  name: alertmanager-warning
  namespace: monitoring
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: alertmanager-warning
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: alertmanager-warning
    spec:
      containers:
      - image: harbor.inner.galaxy.ksyun.com/luban/alertmanager-output
        imagePullPolicy: Always
        name: alertmanager-output
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: alertmanager-warning
  name: alertmanager-warning
  namespace: monitoring
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: alertmanager-warning
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  labels:
    alertmanagerConfig: warning
    release: pm
  name: warning
  namespace: monitoring
spec:
  receivers:
  - name: warning-hook
    webhookConfigs:
    - url: http://alertmanager-warning:8080/webhook
  route:
    groupBy:
    - alertname
    groupInterval: 1m
    groupWait: 30s
    receiver: warning-hook
    repeatInterval: 1m
```



### 安装告警对接飞书

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: alertmanager-feishu
  name: alertmanager-feishu
  namespace: monitoring
spec:
  progressDeadlineSeconds: 600
  replicas: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: alertmanager-feishu
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: alertmanager-feishu
    spec:
      containers:
      - image: harbor.inner.galaxy.ksyun.com/luban/feishu-webhook:latest
        imagePullPolicy: Always
        name: feishu-webhook
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: alertmanager-feishu
  name: alertmanager-feishu
  namespace: monitoring
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: alertmanager-feishu
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  labels:
    alertmanagerConfig: feshu
    release: pm
  name: feishu
  namespace: monitoring
spec:
  receivers:
  - name: feishu-hook
    webhookConfigs:
    - url: http://alertmanager-feishu:8080/alertmanager-alert
  route:
    groupBy:
    - alertname
    groupInterval: 1m
    groupWait: 30s
    receiver: feishu-hook
    repeatInterval: 1m
```



### 导入银河告警规则

```
kubectl apply -f prometheus-rules -n monitoring
```

### 安装监控物理机

目前阶段需要手动安装prometheus监控的物理机列表，后续会自动发现物理机以及部署银河服务的虚拟机列表（暂时手动安装）

```
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: external-nodes
  name: external-nodes
  namespace: monitoring
spec:
  ports:
  - name: metrics
    port: 9100
    protocol: TCP
    targetPort: 9100
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: external-nodes
  name: external-nodes
  namespace: monitoring
subsets:
- addresses:
  - ip: 10.177.16.2
  - ip: 10.177.16.3
  - ip: 10.177.16.4
  - ip: 10.177.16.5
  - ip: 10.177.16.6
  - ip: 10.177.16.7
  - ip: 10.177.9.11
  - ip: 10.178.225.17
  ports:
  - name: metrics
    port: 9100
    protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: external-nodes
    release: pm
  name: external-nodes
  namespace: monitoring
spec:
  endpoints:
  - interval: 60s
    port: metrics
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      k8s-app: external-nodes
```



## 安装鲁班控制台

### 安装鲁班Base

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: console-base
  name: console-base
  namespace: luban-fe
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: console-base
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: console-base
    spec:
      containers:
      - image: harbor.inner.galaxy.ksyun.com/watt/luban-fe/console-base:latest
        imagePullPolicy: Always
        name: console-base
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: console-base
  name: console-base
  namespace: luban-fe
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: console-base
  sessionAffinity: None
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: console-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "luban.console.galaxy.cloud"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: console
spec:
  hosts:
    - "luban.console.galaxy.cloud"
  gateways:
    - console-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            port:
              number: 80
            host: console-base
      corsPolicy:
        allowOrigins:
          - exact: "*"
        allowMethods:
          - GET
          - POST
          - PATCH
          - PUT
          - DELETE
          - OPTIONS
        allowCredentials: false
        allowHeaders:
          - authorization
        maxAge: "24h"
```



### 安装鲁班System

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: console-system
  name: console-system
  namespace: luban-fe
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: console-system
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: console-system
    spec:
      containers:
      - image: harbor.inner.galaxy.ksyun.com/watt/luban-fe/console-system:latest
        imagePullPolicy: Always
        name: console-system
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: console-system
  name: console-system
  namespace: luban-fe
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: console-system
  sessionAffinity: None
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: system-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "luban.system.galaxy.cloud"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: system
spec:
  hosts:
    - "luban.system.galaxy.cloud"
  gateways:
    - system-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            port:
              number: 80
            host: console-system
      corsPolicy:
        allowOrigins:
          - exact: "*"
        allowMethods:
          - GET
          - POST
          - PATCH
          - PUT
          - DELETE
          - OPTIONS
        allowCredentials: false
        allowHeaders:
          - authorization
        maxAge: "24h"
```



### 安装鲁班Monitor

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: console-monitor
  name: console-monitor
  namespace: luban-fe
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: console-monitor
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: console-monitor
    spec:
      containers:
      - image: harbor.inner.galaxy.ksyun.com/watt/luban-fe/console-monitor:latest
        imagePullPolicy: Always
        name: console-monitor
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: console-monitor
  name: console-monitor
  namespace: luban-fe
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: console-monitor
  sessionAffinity: None
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: monitor-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "luban.monitor.galaxy.cloud"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: monitor
spec:
  hosts:
    - "luban.monitor.galaxy.cloud"
  gateways:
    - monitor-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            port:
              number: 80
            host: console-monitor
      corsPolicy:
        allowOrigins:
          - exact: "*"
        allowMethods:
          - GET
          - POST
          - PATCH
          - PUT
          - DELETE
          - OPTIONS
        allowCredentials: false
        allowHeaders:
          - authorization
        maxAge: "24h"
```



### 安装鲁班Demo

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: console-demo
  name: console-demo
  namespace: luban-fe
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: console-demo
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: console-demo
    spec:
      containers:
      - image: harbor.inner.galaxy.ksyun.com/watt/luban-fe/console-demo:latest
        imagePullPolicy: Always
        name: console-demo
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: console-demo
  name: console-demo
  namespace: luban-fe
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: console-demo
  sessionAffinity: None
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: demo-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "luban.demo.galaxy.cloud"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: demo
spec:
  hosts:
    - "luban.demo.galaxy.cloud"
  gateways:
    - demo-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            port:
              number: 80
            host: console-demo
      corsPolicy:
        allowOrigins:
          - exact: "*"
        allowMethods:
          - GET
          - POST
          - PATCH
          - PUT
          - DELETE
          - OPTIONS
        allowCredentials: false
        allowHeaders:
          - authorization
        maxAge: "24h"
```

安装命令

```shell
#!/bin/sh
kubectl apply -f base.yaml -f demo.yaml -f monitor.yaml -f system.yaml -n luban-fe
```



# 操作手册

## 监控

#### 配置prometheus采集

目前研发阶段，需要手动配置prometheus采集，初次创建的时候，需要在k8s中创建ServiceMonitor的crd资源，后续会配合cmdb做服务自动发现，注册到k8s的endpoint。

首次配置采集执行以下脚本：

```
cat <<EOF | kubectl -n monitoring apply -f -
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: external-nodes
  name: external-nodes
spec:
  type: ClusterIP
  ports:
    - name: metrics
      port: 9100
      targetPort: 9100
---
kind: Endpoints
apiVersion: v1
metadata:
  labels:
    k8s-app: external-nodes
  name: external-nodes
subsets:
  - addresses:
    - ip: 10.177.9.11
    ports:
      - name: metrics
        port: 9100
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: external-nodes
  name: external-nodes
spec:
  endpoints:
    - interval: 60s
      port: metrics
  namespaceSelector:
    matchNames:
      - monitoring
  selector:
    matchLabels:
      k8s-app: external-nodes
EOF
 
kubectl label servicemonitor external-nodes release=pm -n monitoring
```



如果后续需要增加监控资源，需要修改已经创建的endpoint，登录鲁班的Master节点，执行以下命令：

```
 kubectl edit ep external-nodes -n monitoring
 ## 增加或修改subsets字段，注意yaml格式
```

登录prometheus界面查看时候添加成功target ： http://10.177.152.168:9090/targets，搜索IP，状态为up

![image-20210325144059470](introduction.assets/image-20210325144059470.png)

登录graph，查看监控：http://10.177.152.168:9090/graph?g0.expr=crash_check&g0.tab=0&g0.stacked=0&g0.range_input=1h

输入监控项：比如crash_check

![image-20210325144237292](introduction.assets/image-20210325144237292.png)

支持label查询，参照wiki：[PromQL](https://wiki.op.ksyun.com/display/~WANGFENGTENG/PromQL)

#### 配置数据存放时间

```
 kubectl edit prometheuses.monitoring.coreos.com -n monitoring pm-kube-prometheus-stack-prometheus -o yaml
```

![image-20210325172208009](introduction.assets/image-20210325172208009.png)

## 告警

### 配置告警

目前prometheus部署方式采用operator部署，创建告警策略PrometheusRule

```
cat <<EOF | kubectl -n monitoring apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    app: kube-prometheus-stack
    release: pm
  name: test-alert-rules
  namespace: monitoring
spec:
  groups:
    - name: test   
      rules:
        - alert: customize-webhook-rule
          annotations:
            summary: summary '{{ $labels.target }}'  crash_check
            description: description '{{ $labels.target }}' crash_check
            message: message '{{ $labels.target }}' crash_check
          expr: |
            crash_check <10
          for: 1m
          labels:
            severity: critical
            alertTag: vip
EOF
```

告警规则也支持prometheus的各种label以及表达式，参照wiki：[PromQL](https://wiki.op.ksyun.com/display/~WANGFENGTENG/PromQL)

登录prometheus界面查看rule是否创建 http://10.177.152.168:9090/rules

![image-20210325155822798](introduction.assets/image-20210325155822798.png)

### 告警规则操作

如果对已经创建的告警规则进行修改，登录kubectl命令操作k8s的中PrometheusRule资源，命名空间为monitoring

```
## 获取全部告警规则
kubectl get PrometheusRule -n monitoring
## 修改告警规则
kubectl edit PrometheusRule {RuleName} -n monitoring
```



### 查看告警

登录alertmanager界面 http://10.177.152.168:9093/#/alerts查看告警

![image-20210325160509660](introduction.assets/image-20210325160509660.png)

登录elasticsearch界面[http://10.177.152.168:9601，查看告警历史，index为alertmanager*

![image-20210325160529568](introduction.assets/image-20210325160529568.png)



### 配置WebHook

目前青岛5环境已经配置2个webhook，一个写入elasticsearch，用于告警历史查询；一个写入飞书，用于运维人员实时处理。

配置命令如下

```
kubectl create deploy alertmanager-warning --image=harbor.inner.galaxy.ksyun.com/luban/alertmanager-output -n monitoring
kubectl create deploy alertmanager-feishu --image=harbor.inner.galaxy.ksyun.com/luban/feishu-webhook:latest -n monitoring
kubectl expose deploy/alertmanager-warning -n monitoring --port=8080 --target-port=8080
kubectl expose deploy/alertmanager-feishu -n monitoring --port=8080 --target-port=8080

cat <<EOF | kubectl -n monitoring apply -f -
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: warning
  labels:
    alertmanagerConfig: warning
    release: pm
spec:
  route:
    groupBy: ['alertname']
    groupWait: 30s
    groupInterval: 1m
    repeatInterval: 1m
    receiver: 'warning-hook'
  receivers:
  - name: 'warning-hook'
    webhookConfigs:
    - url: 'http://alertmanager-warning:8080/webhook'
---
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: feishu
  labels:
    alertmanagerConfig: feshu
    release: pm
spec:
  route:
    groupBy: ['alertname']
    groupWait: 30s
    groupInterval: 1m
    repeatInterval: 1m
    receiver: 'feishu-hook'
  receivers:
  - name: 'feishu-hook'
    webhookConfigs:
    - url: 'http://alertmanager-feishu:8080/alertmanager-alert'
EOF
```



## 日志

#### 配置filebeat

修改/etc/filebeat/filebeat.yml，根据不同日志，传入kafka不同topic中

```
logging.level: debug
 
filebeat.inputs:                      # 从这里开始定义每个日志的路径、类型、收集方式等信息
- type: log                           # 指定收集的类型为 log
  paths:
   - /var/log/secure        # 设置 access.log 的路径
  fields:                             # 设置一个 fields，用于标记这个日志
    log_topic: topic-for-secure      # 为 fields 设置一个关键字 topic，值为 kafka 中已经设置好的 topic 名称
- type: log
  paths:
   - /var/log/messages          # 设置 info.log 的路径
  fields:                            # 设置一个 fields，用于标记这个日志
    log_topic: topic-for-messages       # 为 fields 设置一个关键字 topic，值为 kafka 中已经设置好的 topic 名称
 
output.kafka:
  # 是否启动
  enable: true
  hosts: ["10.177.152.168:30092","10.177.152.168:30093","10.177.152.168:30094"]
  partition.round_robin: #开启kafka的partition分区
    reachable_only: true
  worker: 2
  # 代理要求的ACK可靠性级别
  # 0=无响应，1=等待本地提交，-1=等待所有副本提交
  # 默认值是1
  # 注意:如果设置为0,Kafka不会返回任何ack。出错时，消息可能会悄无声息地丢失。
  required_acks: 1
  compression: gzip      #压缩格式
  max_message_bytes: 10000000    #压缩格式字节大小
  # topic: '%{[fields.topic]}'        # 根据每个日志设置的 fields.topic 来输出到不同的 topic
  topic: '%{[fields.log_topic]}'
```

#### 配置logstash

在logging的安装目录，根据不同日志，获取kafka不同topic中,修改values.yaml

```
logstashConfig:
  logstash.yml: |
    http.host: 0.0.0.0
    monitoring.elasticsearch.hosts: http://elasticsearch-master:9200
 
# Allows you to add any pipeline files in /usr/share/logstash/pipeline/
### ***warn*** there is a hardcoded logstash.conf in the image, override it first
logstashPipeline:
  logstash.conf: |
    input { kafka { bootstrap_servers => "kafka-0.kafka-headless.elastic-system.svc.cluster.local:9092,kafka-1.kafka-headless.elastic-system.svc.cluster.local:9092,kafka-2.kafka-headless.elastic-system.svc.cluster.local:9092" topics => ["topic-for-secure"] } }
    output { elasticsearch { hosts => ["http://elasticsearch-master:9200"] index => "topic-for-secure-%{+YYYY.MM.dd}" } }
#  logstash.conf: |
#    input {
#      exec {
#        command => "uptime"
#        interval => 30
#      }
#    }
#    output { stdout { } }
```

在k8s中创建新的logstash

```
kubectl create ns elastic-system
export REGISTRY="harbor.inner.galaxy.ksyun.com"
helm install {logstash--new-name} ./logstash -n elastic-system  --set imageTag=7.10.1 --set image=$REGISTRY/luban/logstash
```

#### 配置kibana

登录kibana界面，查看es中是否创建新的index

http://10.177.152.168:9601/app/management/data/index_management/indices

创建kibana的indexPatterns

http://10.177.152.168:9601/app/management/kibana/indexPatterns

#### 配置日志清理

需要注意，es中的index需要有@timestamp

在logging的安装目录，修改

```
 kk get cm -n elastic-system curator-elasticsearch-curator-config -o yaml
```



![image-20210325171929064](introduction.assets/image-20210325171929064.png)



# 存储容量规划

## 监控

磁盘容量规划，首先明确几个概念:

- **监控节点**: 一个 exporter 进程被认为是一个监控节点。Manager 在安装 AQUILA时，默认每个节点都会安装一个 node-exporter 收集节点信息(CPU, Memory 等), 每个节点安装一个 tdh-exporter 收集 TDH Services metrics (目前有: HDFS, YARN, ZOOKEEPER, KAFKA, HYPERBASE, INCEPTOR). 故在 TDH 集群上, 每个节点有两个 exporter 进程.
- **测量点**: 一个测量点代表了某监控节点上的一个观测对象. 从某测量点采集到的一组样本数据构成一条时间序列（time series).
- **抓取间隔**: Promtheus 对某个监控节点采集 metrics 的时间间隔. 一般为同类监控节点设置相同的抓取间隔. AQUILA对应的配置值为: prometheus.node.exporter.scrape_interval(默认15s) 和 prometheus.tdh.exporter.scrape_interval(默认60s)
- **保留时间**: 样本数据在磁盘上保存的时间,超过该时间限制的数据就会被删除. 存储在磁盘上的样本都是经过编码之后的样本(对样本进行过数据编码, 一般为 double-delta 编码). AQUILA对应的配置值为 prometheus.storage.retention.time(默认15天)
- **活跃样本留存时间**: 留存于内存的活跃样本（已经被编码）在内存保留时间. 在内存中的留存数据越多，查询过往数据的性能越高，但是消耗内存也会增加. 在实际应用中，需要根据所监控的业务的性质，设定合理的内存留存时间. AQUILA对应的配置值为 prometheus.min-block-duration (默认2h), prometheus.max-block-duration(默认26h). Facebook 在论文 《Gorilla: A Fast, Scalable, In-Memory Time Series DataBase》 （Prometheus实现参考论文）中给出了留存内存时间的一般经验: 26 h.
- **样本(测量点)大小**: 根据 Prometheus 官方文档说明, 每一个编码后的样本大概占用1-2字节大小

## 日志

需要根据不同的日志，以及日志的大小规划

登录kibana查看index使用大小，规划日志的存储

http://10.177.152.168:9601/app/management/data/index_management/indices

# 附录

### 青岛1发布环境hosts信息

https://wiki.op.ksyun.com/pages/viewpage.action?pageId=157002294

### 告警产生流程

1. Prometheus Server监控目标主机上暴露的http接口（这里假设接口A），通过上述Promethes配置的'scrape_interval'定义的时间间隔，定期采集目标主机上监控数据。
2. 当接口A不可用的时候，Server端会持续的尝试从接口中取数据，直到"scrape_timeout"时间后停止尝试。这时候把接口的状态变为“DOWN”。
3. Prometheus同时根据配置的"evaluation_interval"的时间间隔，定期（默认1min）的对Alert Rule进行评估；当到达评估周期的时候，发现接口A为DOWN，即UP=0为真，激活Alert，进入“PENDING”状态，并记录当前active的时间；
4. 当下一个alert rule的评估周期到来的时候，发现UP=0继续为真，然后判断警报Active的时间是否已经超出rule里的‘for’ 持续时间，如果未超出，则进入下一个评估周期；如果时间超出，则alert的状态变为“FIRING”；同时调用Alertmanager接口，发送相关报警数据。
5. AlertManager收到报警数据后，会将警报信息进行分组，然后根据alertmanager配置的“group_wait”时间先进行等待。等wait时间过后再发送报警信息。
6. 属于同一个Alert Group的警报，在等待的过程中可能进入新的alert，如果之前的报警已经成功发出，那么间隔“group_interval”的时间间隔后再重新发送报警信息。比如配置的是邮件报警，那么同属一个group的报警信息会汇总在一个邮件里进行发送。
7. 如果Alert Group里的警报一直没发生变化并且已经成功发送，等待‘repeat_interval’时间间隔之后再重复发送相同的报警邮件；如果之前的警报没有成功发送，则相当于触发第6条条件，则需要等待group_interval时间间隔后重复发送。
同时最后至于警报信息具体发给谁，满足什么样的条件下指定警报接收人，设置不同报警发送频率，这里有alertmanager的route路由规则进行配置。













