---
layout:       post
title:        "k8s故障日记：修改linux内核参数导致的pod网络不可用"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - CloudNative
---

## 背景

客户报障：k8s集群上的某个pod无法访问其他pod。

这里集群网络用的是underlay模式，依赖云供应商提供的虚拟网络服务，每个pod申请独立的IP地址。通过veth pair的方式加入到虚拟网络中以实现跨主机的网络互通。

这种underlay网络模式不需要依赖独立的overlay网络插件（例如flannel），由云厂商提供的CNI来维护集群网络，CNI需要负责pod IP的分配、网络配置、销毁等一系列操作。上层用户一般不需要关注底层网络架构，只需要知道所有pod都处于同一个VPC即可。

## 准备

为了复盘整个排查过程，我准备了一个空闲的演示集群，下面是集群的规模情况：

```bash
kubectl get node
# NAME          STATUS                     ROLES    AGE   VERSION
# 10.9.0.138    Ready,SchedulingDisabled   master   35h   v1.22.5
# 10.9.15.187   Ready,SchedulingDisabled   master   35h   v1.22.5
# 10.9.189.51   Ready                      <none>   29h   v1.22.5
# 10.9.51.67    Ready,SchedulingDisabled   master   35h   v1.22.5
# 10.9.82.243   Ready                      <none>   35h   v1.22.5
```

该集群有2个node，我们创建一个DaemonSet，以在两个node上面分别启动一个pod：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: test-nginx
  namespace: default
spec:
  selector:
    matchLabels:
      app: test-nginx
  template:
    metadata:
      labels:
        app: test-nginx
    spec:
      containers:
        - name: test-nginx
          image: nginx:latest
          ports:
            - name: web
              containerPort: 80
              protocol: TCP
```

我对`10.9.189.51`这个node了做了点手脚（后面将揭晓~），现在这个node上面的pod无法访问另一个pod：

```bash
kubectl get po -owide
kubectl exec -it test-nginx-h8wz5 -- sh
ping 10.9.30.253
#（这里卡死，ping不通）
```

## 排查

遇到这种问题，第一个想到的就是云供应商的VPC服务是不是出现了问题。我第一时间做了以下检查：

- 登录出问题的node，在node上ping其它node以及pod，可以ping通。
- 登录出问题的node上面的pod，在里面ping宿主机，可以ping通。
- 登录出问题的node上面的pod，在里面ping其它node和pod，ping不通。

这是一个很奇怪的现象：

- pod跟主机网络之间是通的。
- 主机跟其它主机是通的。
- pod跟其它主机不通。

这就可以基本排除是VPC的问题了，可能是CNI网络配置有问题，需要深入pod查看当前underlay网络是如何配置的了。

到pod上面，查看路由（如果pod比较原始，没有ip，ping等命令，可以使用`nsenter`来完成，详情可以参考我专门介绍`nsenter`的文章）：

```bash
ip ro
# default via 10.9.189.51 dev eth0 onlink
```

可以看到，pod的流量都走到主机`10.9.189.51`上面了，那么流量走到了主机的哪个设备上面呢？这里是通过veth pair技术实现的，可以通过`ip neigh`命令查看流量走到了哪个设备上面：

```bash
ip neigh
# 10.9.189.51 dev eth0 lladdr 56:ec:85:ba:41:00 used 0/0/0 probes 0 PERMANENT
```

流量走到了主机的MAC地址为`56:ec:85:ba:41:00`的设备上，我们回到主机，通过`ip a`命令查看设备列表，寻找该MAC地址的设备：

```bash
ip a
# ...(省略了非目标设备输出)
# 12: ucni88bdbf34ff6@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1452 qdisc noqueue state UP group default
#     link/ether 56:ec:85:ba:41:00 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

这是一个由CNI创建的设备，它实际上是一个veth pair，pod通过这个设备来连接到主机网络。

那么，正常来说，数据包会从pod打到这个设备上面，然后再走主机的eth0设备出去。

这时候我们可以通过抓包来看流量是否打到了对应的设备上面，首先来看veth pair，另外启动一个terminal，进入pod并保持对其它pod的ping。然后回到当前node，使用`tcpdump`对veth pair进行抓包：

```bash
tcpdump -i ucni88bdbf34ff6 -n icmp
# tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
# listening on ucni88bdbf34ff6, link-type EN10MB (Ethernet), capture size 262144 bytes
# 23:40:26.723916 IP 10.9.11.187 > 10.9.82.243: ICMP echo request, id 7936, seq 0, length 64
# 23:40:27.724032 IP 10.9.11.187 > 10.9.82.243: ICMP echo request, id 7936, seq 1, length 64
# 23:40:28.724188 IP 10.9.11.187 > 10.9.82.243: ICMP echo request, id 7936, seq 2, length 64
# 23:40:29.724328 IP 10.9.11.187 > 10.9.82.243: ICMP echo request, id 7936, seq 3, length 64
# 23:40:30.724491 IP 10.9.11.187 > 10.9.82.243: ICMP echo request, id 7936, seq 4, length 64
```

可以看到pod正常向该设备发送了数据包，并且设备也尝试将数据包发送给了`10.9.82.243`（目标pod的IP地址），但是没有收到回包。

正常情况下，所有node的出口流量都应该走eth0出去，因此再用同样的方法对eth0进行抓包：

```bash
tcpdump -i eth0 -n icmp
# tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
# listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
```

可以看到，eth0没有发出任何包。这就定位到问题了，说明pod的数据包被正确路由到了veth pair设备`ucni88bdbf34ff6`上面，但是veth pair没有将数据包转给`eth0`以发送出去。

当时定位到这里之后，也是卡了很久的时间，因为本菜鸡对Linux网络不是特别精通，所以不太明白为什么数据包没有被转到eth0上面。直到后面找到了Linux Kernel关于ip_forward的知识：

>ip_forward - BOOLEAN
>
>0 - disabled (default)
>
>not 0 - enabled
>
>Forward Packets between interfaces.

这是一个Linux内核参数，它用来控制是否允许在不同接口之间转发数据包，默认是关闭的。在关闭的情况下，数据包是不允许跨接口转发的。在我们的node上面，可以通过`/proc/sys/net/ipv4/ip_forward`这个文件查看这个参数的值：

```bash
cat /proc/sys/net/ipv4/ip_forward
# 0
```

这个参数是关闭的！这就解释了veth pair的数据包为什么没有被转发到eth0上面。

把这个参数打开，即可解决问题：

```bash
echo "1" > /proc/sys/net/ipv4/ip_forward
```

现在，pod就可以ping通了，并且可以在eth0抓到数据包以及回包：

```bash
tcpdump -i eth0 -n icmp
# tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
# listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
# 00:06:44.930698 IP 10.9.11.187 > 10.9.82.243: ICMP echo request, id 7936, seq 1578, length 64
# 00:06:44.931049 IP 10.9.82.243 > 10.9.11.187: ICMP echo reply, id 7936, seq 1578, length 64
# 00:06:45.930878 IP 10.9.11.187 > 10.9.82.243: ICMP echo request, id 7936, seq 1579, length 64
# 00:06:45.931221 IP 10.9.82.243 > 10.9.11.187: ICMP echo reply, id 7936, seq 1579, length 64
```

这时你可能会想，即然这个参数默认是0，那CNI岂不是完全不可用，因为默认情况下veth pair的数据包就无法转发到eth0上面。

后来经过我们调查，CNI在启动的时候会修改这个参数为1。客户是执行某个其它脚本时，不经意将这个参数改回成了0，导致节点的underlay网络失效。

至此，问题解决。

## 总结

这个问题看似很简单，也确实很简单，只要对Linux网络足够熟悉，应该能很快定位出问题。但是我花了很多时间来定位，说明我对Linux网络基本功还是太差了，要提高处理问题的能力，还是要不断学习Linux运维知识，不能仅停留在研发层面。

另外再总结一下本次排查underlay网络的教训：

- 不同CNI对于underlay网络的实现思路是不同的，在排查问题前，一定要了解整个CNI的运作机制，这样可以更快地定位问题。
- 抓包是排查网络问题的一大利器，可以说现在稍微复杂一些的网络问题不用抓包基本都是排查不出来的，一定要善用抓包。
- Linux内核参数还是要花点时间去熟悉一下，以应对未来潜在的各种情况。
