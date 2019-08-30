# kubernetes-redis-5.0.5-cluster

用于记录kubernetes与redis5.0.5集群的整合

所有相关配置文件都已经存储到addons中

本文选择的是redis-5.0.5的集群版本，相关镜像地址:

redis-image

redis创建集群首先需要将所需要的redis节点创建起来，并且所有redis节点状态的配置文件无特殊需求，所以可以共同使用相同的配置文件。

# 部署

部署之前，我们创建一个独立的namespace

`kubectl create namespace redis-cluster`

本章实例使用的是statefulset进行pod的管理，其实redis使用deployment一样可以正常使用，只不过需要额外解决持久化存储的问题，redis的data目录在运行中需要进行持久化存储，使用statefulset可以便捷的为每个pod都分配一个存储盘。

执行如下命令，完成redis集群节点搭建

`kubectl create -f addons/.`

若您本地环境没有可直接使用的storeageclass，请先为自己的集群部署一个storageclass，本文就不再赘述。不了解storageclass点击这里。

部署成功后，查看集群状态

`[root@kubemaster ~]# kubectl get pod,service --namespace=redis-cluster -owide`

`NAME              READY     STATUS    RESTARTS   AGE       IP             NODE`

`pod/redis-app-0   1/1       Running   0          16m       10.244.2.245   slave2`

`pod/redis-app-1   1/1       Running   0          16m       10.244.0.187   kubemaster`

`pod/redis-app-2   1/1       Running   0          16m       10.244.1.49    slave1`

`pod/redis-app-3   1/1       Running   0          16m       10.244.1.50    slave1`

`pod/redis-app-4   1/1       Running   0          15m       10.244.2.246   slave2`

`pod/redis-app-5   1/1       Running   0          15m       10.244.0.188   kubemaster`

`  
`

`NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE       SELECTOR`

`service/redis-access-service   NodePort    10.103.198.106   <none>        6379:30379/TCP   16m       app=redis,appCluster=redis-cluster`

`service/redis-service          ClusterIP   None             <none>        6379/TCP         16m       app=redis,appCluster=redis-cluster`

可以看到所有的pod和service都已经部署好了。

# 创建集群

此时我们需要进入到redis集群中，进行集群初始化操作:

`[root@kubemaster redis-5.0.5-cluster]# kubectl exec -it --namespace=redis-cluster redis-app-0 /bin/bash`

`root@redis-app-0:/data# redis-cli --cluster create 10.244.2.245:6379 10.244.0.187:6379 10.244.1.49:6379 10.244.1.50:6379 10.244.2.246:6379 10.244.0.188:6379 --cluster-replicas 1`

依次将redis对应的pod地址填入，注意此处不可以填写域名，redis自身对域名解析不好，我们填写为pod的ip地址。

创建成功后可以看到类似如下信息

`>>> Performing Cluster Check (using node 10.244.2.245:6379)`

`M: 8434b3117c9e3304c51a7f1b463a8b5a3a6dcfaa 10.244.2.245:6379`

`slots:[0-5460] (5461 slots) master`

`1 additional replica(s)`

`M: cda0c5b72a8cde9e93e01089c7092a9a710431c5 10.244.0.187:6379`

`slots:[5461-10922] (5462 slots) master`

`1 additional replica(s)`

`S: a66c30807d2857cfd0d50e4c10587744045aa0be 10.244.0.188:6379`

`slots: (0 slots) slave`

`replicates cda0c5b72a8cde9e93e01089c7092a9a710431c5`

`S: bfb98a4d55c77b86686c8900ffa105cde7020c13 10.244.1.50:6379`

`slots: (0 slots) slave`

`replicates c05c483527bdbcaf7636e275dc8976e02dc41619`

`S: c0fcd18205a3455f9877d3c36ea0a53e47091619 10.244.2.246:6379`

`slots: (0 slots) slave`

`replicates 8434b3117c9e3304c51a7f1b463a8b5a3a6dcfaa`

`M: c05c483527bdbcaf7636e275dc8976e02dc41619 10.244.1.49:6379`

`slots:[10923-16383] (5461 slots) master`

`1 additional replica(s)`

`[OK] All nodes agree about slots configuration.`

`>>> Check for open slots...`

`>>> Check slots coverage...`

`[OK] All 16384 slots covered.`

此时集群已经创建完毕，我们查看集群状态

`root@redis-app-0:/data# redis-cli`

`127.0.0.1:6379> CLUSTER INFO`

`cluster_state:ok`

`cluster_slots_assigned:16384`

`cluster_slots_ok:16384`

`cluster_slots_pfail:0`

`cluster_slots_fail:0`

`cluster_known_nodes:6`

`cluster_size:3`

`cluster_current_epoch:6`

`cluster_my_epoch:1`

`cluster_stats_messages_ping_sent:3155`

`cluster_stats_messages_pong_sent:3280`

`cluster_stats_messages_sent:6435`

`cluster_stats_messages_ping_received:3275`

`cluster_stats_messages_pong_received:3155`

`cluster_stats_messages_meet_received:5`

`cluster_stats_messages_received:6435`

查看节点状态:

`127.0.0.1:6379> CLUSTER NODES`

`cda0c5b72a8cde9e93e01089c7092a9a710431c5 10.244.0.187:6379@16379 master - 0 1567150579000 2 connected 5461-10922`

`a66c30807d2857cfd0d50e4c10587744045aa0be 10.244.0.188:6379@16379 slave cda0c5b72a8cde9e93e01089c7092a9a710431c5 0 1567150579077 6 connected`

`8434b3117c9e3304c51a7f1b463a8b5a3a6dcfaa 10.244.2.245:6379@16379 myself,master - 0 1567150579000 1 connected 0-5460`

`bfb98a4d55c77b86686c8900ffa105cde7020c13 10.244.1.50:6379@16379 slave c05c483527bdbcaf7636e275dc8976e02dc41619 0 1567150580581 4 connected`

`c0fcd18205a3455f9877d3c36ea0a53e47091619 10.244.2.246:6379@16379 slave 8434b3117c9e3304c51a7f1b463a8b5a3a6dcfaa 0 1567150579578 5 connected`

`c05c483527bdbcaf7636e275dc8976e02dc41619 10.244.1.49:6379@16379 master - 0 1567150579077 3 connected 10923-16383`

# 访问集群

我们需要远程链接redis，首先查看暴露出来的redis-svc地址

\[root@kubemaster ~\]\# kubectl get svc --namespace=redis-cluster redis-access-service

NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT\(S\)          AGE

redis-access-service   NodePort   10.103.198.106   &lt;none&gt;        6379:30379/TCP   39m

# 高可用实现

由于k8s本身对容器的运行已经具备较强的可用性保障，可以保障容器一直按照我们期望的状态或实例数运行，在这里我们就不再阐述容器自身存活的可用性保障。

理解k8s的朋友应该已经发现，我们使用的是podip而非服务名，k8s中statefulset是可以保障每个pod的服务访问地址唯一的，但由于redis自身对域名解析不好的原因，无法填写域名，故pod出现各类异常问题的情况，k8s将对容器重新调度，重新调度的容器大概率会更换ip地址，此时可能会对集群的高可用造成影响。

## 非全体节点故障

redis的nodes.conf配置文件，维护了redis集群的状态信息，其中重点维护了redis自身节点的id和redis集群的ip信息，在redis自身实例启动过程中，将读取该文件，本实例的nodes.conf文件以持久化的方式存储在了/var/lib中。

redis实例在启动过程中，会根据nodes.conf信息寻找集群地址，只要满足一个节点可以被寻址到，并匹配成功相关的id信息，此redis实例就将加入到该集群中，这种机制极大满足了在kubernetes场景中的ip更换问题。

因为在nodes.conf中维护了所有node节点的状态，即便大多数节点ip变更出现节点丢失，但在集群加入的过程中， 只要保障有一个节点和原本节点相同，即可保障所有节点的加入。

基于这种 理论，我们可以模拟容器故障的场景，测试下非全体节点故障的集群可用性。

首先我们查看所有节点的容器ip地址：

`[root@kubemaster redis-5.0.5-cluster]# kubectl get pod -owide --namespace=redis-cluster |awk '{print $6}' |grep -v IP`

`10.244.2.245`

`10.244.0.187`

`10.244.1.49`

`10.244.1.50`

`10.244.2.246`

`10.244.0.188`

记录这些ip地址后，我们手动删除5个pod，仅保留ip为10.244.0.188的pod:redis-app-5

`kubectl delete pod -n redis-cluster redis-app-0`

`kubectl delete pod -n redis-cluster redis-app-1`

`kubectl delete pod -n redis-cluster redis-app-2`

`kubectl delete pod -n redis-cluster redis-app-3`

`kubectl delete pod -n redis-cluster redis-app-4`

全部删除成功后查看下现在的容器状态：

`[root@kubemaster redis-5.0.5-cluster]# kubectl get pod -owide -n redis-cluster`

`NAME          READY     STATUS    RESTARTS   AGE       IP             NODE`

`redis-app-0   1/1       Running   0          1m        10.244.2.247   slave2`

`redis-app-1   1/1       Running   0          1m        10.244.0.190   kubemaster`

`redis-app-2   1/1       Running   0          57s       10.244.1.67    slave1`

`redis-app-3   1/1       Running   0          47s       10.244.1.68    slave1`

`redis-app-4   1/1       Running   0          36s       10.244.2.248   slave2`

`redis-app-5   1/1       Running   0          1h        10.244.0.188   kubemaster`

处理发现除了redis-app-5 之外，所有的podip都已经改变

此时我们在进入容器内查看集群的状态:

`127.0.0.1:6379> CLUSTER INFO`

`cluster_state:ok`

`127.0.0.1:6379> CLUSTER NODES`

`cda0c5b72a8cde9e93e01089c7092a9a710431c5 10.244.0.190:6379@16379 slave a66c30807d2857cfd0d50e4c10587744045aa0be 0 1567154253529 8 connected`

`a66c30807d2857cfd0d50e4c10587744045aa0be 10.244.0.188:6379@16379 master - 0 1567154254030 8 connected 5461-10922`

`c05c483527bdbcaf7636e275dc8976e02dc41619 10.244.1.67:6379@16379 slave bfb98a4d55c77b86686c8900ffa105cde7020c13 0 1567154254000 9 connected`

`bfb98a4d55c77b86686c8900ffa105cde7020c13 10.244.1.68:6379@16379 master - 0 1567154254000 9 connected 10923-16383`

`c0fcd18205a3455f9877d3c36ea0a53e47091619 10.244.2.248:6379@16379 slave 8434b3117c9e3304c51a7f1b463a8b5a3a6dcfaa 0 1567154254530 11 connected`

`8434b3117c9e3304c51a7f1b463a8b5a3a6dcfaa 10.244.2.245:6379@16379 myself,master - 0 1567154254000 11 connected 0-5460`

发现集群状态为监控，所有节点都已经加入的集群中，同事发现nodes.conf配置文件已经被修改为新的ip地址

root@redis-app-0:/data\# cat /var/lib/redis/nodes.conf

cda0c5b72a8cde9e93e01089c7092a9a710431c5 10.244.0.190:6379@16379 slave a66c30807d2857cfd0d50e4c10587744045aa0be 0 1567154110000 8 connected

a66c30807d2857cfd0d50e4c10587744045aa0be 10.244.0.188:6379@16379 master - 0 1567154110516 8 connected 5461-10922

c05c483527bdbcaf7636e275dc8976e02dc41619 10.244.1.67:6379@16379 slave bfb98a4d55c77b86686c8900ffa105cde7020c13 0 1567154111000 9 connected

bfb98a4d55c77b86686c8900ffa105cde7020c13 10.244.1.68:6379@16379 master - 0 1567154111121 9 connected 10923-16383

c0fcd18205a3455f9877d3c36ea0a53e47091619 10.244.2.248:6379@16379 slave 8434b3117c9e3304c51a7f1b463a8b5a3a6dcfaa 0 1567154111236 11 connected

8434b3117c9e3304c51a7f1b463a8b5a3a6dcfaa 10.244.2.245:6379@16379 myself,master - 0 1567154111000 11 connected 0-5460

## 全体节点故障

当全体节点故障的情况下，任何一个pod不可以对集群进行寻址，此时集群将不能自动恢复，需要手动告诉集群的所有节点，集群的位置在哪里，redis-cli命令可以实现此需求，

在实例启动之后，可以选取任意节点为集群的发现节点，此节点充当上面自动恢复集群过程的中心节点，如上面使用的redis-app-5 节点。

模拟全节点故障

删除所有pod

`[root@kubemaster ~]# kubectl delete pod --namespace=redis-cluster $(kubectl get pod --namespace=redis-cluster | awk '{print $1}'|grep -v NAME)`

等待容器全部恢复后，进入到任一一台节点

`root@redis-app-0:/data# redis-cli`

`127.0.0.1:6379> CLUSTER INFO`

`cluster_state:fail`

可以发现集群已经出现故障:

依次发现其他节点，加入的redis集群

`127.0.0.1:6379> CLUSTER MEET $IP 6379`

全部加入后，查看集群状态

`127.0.0.1:6379> CLUSTER INFO`

`cluster_state:ok`

所以，若想将redis集群在k8s环境下实现高可用的话，在整个机房或中心发生故障后，需要手动或编写相关的脚本执行完成集群加入的操作即可。

# 增加节点:

增加节点非常简单，首先我们将statefulset副本调整为7，addons中的yaml文件原本为6.

调整好后，查看pod状态:

```
[root@kubemaster redis-5.0.5-cluster]# kubectl get pod -owide -n redis-cluster
NAME          READY     STATUS    RESTARTS   AGE       IP             NODE
redis-app-0   1/1       Running   0          13m       10.244.0.191   kubemaster
redis-app-1   1/1       Running   0          13m       10.244.2.8     slave2
redis-app-2   1/1       Running   0          12m       10.244.1.69    slave1
redis-app-3   1/1       Running   0          12m       10.244.0.192   kubemaster
redis-app-4   1/1       Running   0          12m       10.244.1.70    slave1
redis-app-5   1/1       Running   0          12m       10.244.2.9     slave2
redis-app-6   1/1       Running   0          3m        10.244.2.10    slave2
```

此时我们加入新增的pod容器中执行加入集群指令

```
127.0.0.1:6379> CLUSTER MEET 10.244.0.191 6379
```

加入成功后，我们可以查看集群状态信息观察是否加入:

```
127.0.0.1:6379> CLUSTER INFO
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:7
cluster_size:3
cluster_current_epoch:11
cluster_my_epoch:0
cluster_stats_messages_ping_sent:451
cluster_stats_messages_pong_sent:463
cluster_stats_messages_meet_sent:6
cluster_stats_messages_sent:920
cluster_stats_messages_ping_received:463
cluster_stats_messages_pong_received:457
cluster_stats_messages_received:920
```

# 清理环境

```
kubectl delete -f addons/.
kubectl delete pvc -n redis-cluster $(kubectl get pvc -n redis-cluster |awk '{print $1}' |grep -v NAME)
```



