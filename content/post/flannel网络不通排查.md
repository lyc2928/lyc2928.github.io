---
title: "Flannel网络不通排查"
date: 2019-03-10T14:44:00+08:00
draft: false
categories: ["linux", "k8s"]
tags: ["linux", "flannel", "k8s"]
---

### 最近在做k8s实验
> 使用 kubeadmi 快速构建一个集群环境

```
master: 10.211.55.120
node:   10.211.55.121
```

### 安装网络插件,使用flannel

```bash
kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```bash
[root@k8s-master ~]# kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
nginx-54458cd494-ksw5s   1/1     Running   1          51m   10.244.0.6   k8s-master   <none>           <none>
nginx-54458cd494-xsw8f   1/1     Running   1          51m   10.244.1.3   k8s-node     <none>           <none>
```
### master ping 不通node节点上pod
```bash
[root@k8s-master ~]# ping 10.244.1.3
PING 10.244.1.3 (10.244.1.3) 56(84) bytes of data.
^C
--- 10.244.1.3 ping statistics ---
11 packets transmitted, 0 received, 100% packet loss, time 10004ms

[root@k8s-master ~]# ping 10.244.0.6
PING 10.244.0.6 (10.244.0.6) 56(84) bytes of data.
64 bytes from 10.244.0.6: icmp_seq=1 ttl=64 time=0.119 ms
64 bytes from 10.244.0.6: icmp_seq=2 ttl=64 time=0.091 ms

```


### 检查路由表
```bash
[root@k8s-master ~]# ip r s
default via 10.211.55.1 dev eth0 proto static metric 100
10.211.55.0/24 dev eth0 proto kernel scope link src 10.211.55.120 metric 100
10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
[root@k8s-master ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.211.55.1     0.0.0.0         UG    100    0        0 eth0
10.211.55.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.1.0      10.244.1.0      255.255.255.0   UG    0      0        0 flannel.1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```
> 路由都存在, 是正常的

### node节点上 tcpdum 抓包
```bash
[root@k8s-node ~]# tcpdump icmp -vvv -nn
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
01:08:42.445216 IP (tos 0xc0, ttl 64, id 22760, offset 0, flags [none], proto ICMP (1), length 162)
    10.211.55.121 > 10.211.55.120: ICMP host 10.211.55.121 unreachable - admin prohibited, length 142
	IP (tos 0x0, ttl 64, id 4411, offset 0, flags [none], proto UDP (17), length 134)
    10.211.55.120.38499 > 10.211.55.121.8472: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 1
IP (tos 0x0, ttl 64, id 4825, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.0.0 > 10.244.1.3: ICMP echo request, id 17289, seq 1, length 64
01:08:43.444979 IP (tos 0xc0, ttl 64, id 23715, offset 0, flags [none], proto ICMP (1), length 162)
    10.211.55.121 > 10.211.55.120: ICMP host 10.211.55.121 unreachable - admin prohibited, length 142
	IP (tos 0x0, ttl 64, id 4612, offset 0, flags [none], proto UDP (17), length 134)
    10.211.55.120.38499 > 10.211.55.121.8472: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 1
IP (tos 0x0, ttl 64, id 5013, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.0.0 > 10.244.1.3: ICMP echo request, id 17289, seq 2, length 64
01:08:44.446015 IP (tos 0xc0, ttl 64, id 24297, offset 0, flags [none], proto ICMP (1), length 162)
```

> 没有发现 reply 包响应

### 解决
```
iptable 发现问题

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 KUBE-FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */
2        0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
4        0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
5        0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
6        0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
7        0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0
8        0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
9        0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
10       0     0 FORWARD_direct  all  --  *      *       0.0.0.0/0            0.0.0.0/0
11       0     0 FORWARD_IN_ZONES_SOURCE  all  --  *      *       0.0.0.0/0            0.0.0.0/0
12       0     0 FORWARD_IN_ZONES  all  --  *      *       0.0.0.0/0            0.0.0.0/0
13       0     0 FORWARD_OUT_ZONES_SOURCE  all  --  *      *       0.0.0.0/0            0.0.0.0/0
14       0     0 FORWARD_OUT_ZONES  all  --  *      *       0.0.0.0/0            0.0.0.0/0
15       0     0 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
16       0     0 ACCEPT     all  --  *      *       10.244.0.0/16        0.0.0.0/0
17       0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16
```
> REJECT(或DROP) 后还有规则, 这是有问题
> 马上删除掉这规则, 同理也删除掉INPUT 链的规则 