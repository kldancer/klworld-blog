+++
title = '从kube-ovn中vpc网关EIP到Netfilter Hook优先级探讨'
date = 2024-09-26T14:45:04+08:00
draft = false
tags= ["网络","Linux kernel","kube-ovn"]
categories= ["网络"]
author= ["kl"]
+++


# 前言

  博主近期工作上需要和kube-ovn打交道，后续可能会涉及到基于kube-ovn的二开，因此需要详细了解其特性。昨天在工作中接到了一个关于**“k8s平台中容器部署的虚拟机中，先ping着外网，然后绑定EIP成功后还是不通，需要把ping关了重新ping才生效，解绑EIP也是如此“**的问题分析。

# VPC

kube-ovn中关于VPC的设计，引用原文：

> VPC 主要用于有多租户网络强隔离的场景，部分 Kubernetes 网络功能在多租户网络下存在冲突。 例如节点和 Pod 互访，NodePort 功能，基于网络访问的健康检查和 DNS 能力在多租户网络场景暂不支持。 为了方便常见 Kubernetes 的使用场景，Kube-OVN 默认 VPC 做了特殊设计，该 VPC 下的 Subnet 可以满足 Kubernetes 规范。用户自定义 VPC 支持本文档介绍的静态路由，EIP 和 NAT 网关等功能。 **常见隔离需求可通过默认 VPC 下的网络策略和子网 ACL 实现**，在使用自定义 VPC 前请明确是否需要 VPC 级别的隔离，并了解自定义 VPC 下的限制。 **在 Underlay 网络下，物理交换机负责数据面转发，VPC 无法对 Underlay 子网进行隔离**。
> 

可见kube-ovn中VPC的设计是针对Overlay网络的多租户隔离需求，像单租户下的Overlay网络、Underlay网络分别可以通过k8s中的网络策略、Underlay的Vlan实现隔离需求。

自定义VPC主要是围绕一个VPC网关实现EIP、自定义路由、自定义内部负载均衡、自定义vpc-dns等功能，本次主要设计其EIP功能，拓扑架构如图：

![image.png](/img/kube-ovn-eip-netfilter.png)

# VPC Nat Gateway

从具体的CR资源来看其对应关系：

```bash
[root@cd56 ~]# kubectl get vpc-nat-gateways.kubeovn.io 
NAME           VPC            SUBNET            LANIP
eip-p7cs4r5y   vpc-iwm2hkn4   subnet-skwgfk6s   13.13.0.253
[root@cd56 ~]#  kubectl get vpc-nat-gateways.kubeovn.io eip-p7cs4r5y -oyaml 
apiVersion: kubeovn.io/v1
kind: VpcNatGateway
metadata:
  name: eip-p7cs4r5y
spec:
  **externalSubnets: # VPC网关使用的外部网络
  - net-g0q7ypsh**
  subnet: subnet-skwgfk6s **# VPC网关使用的subnet**
  lanIp: 13.13.0.253 **# VPC subnet内某个未使用的IP，作为该VPC内的网关IP**
  vpc: vpc-iwm2hkn4 **# 所属VPC**
  qosPolicy: ""
  selector:
  - 'nodeType: controller'
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
    operator: Exists

# 该 Subnet 用来管理可用的外部地址，网段内的地址将会通过 Macvlan 分配给 VPC 网关 
[root@cd56 ~]# kubectl get subnets.kubeovn.io net-g0q7ypsh -oyaml 
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: net-g0q7ypsh
spec:
  cidrBlock: 192.169.100.0/24
  default: false
  enableLb: true
  excludeIps:
  - 192.169.100.1
  gateway: 192.169.100.1
  gatewayNode: ""
  gatewayType: distributed
  natOutgoing: false
  private: false
  protocol: IPv4
  **provider: net-g0q7ypsh.kube-system**
  
# VPC 网关功能依赖 Multus-CNI 的多网卡功能
[root@cd55 ~]# kubectl get network-attachment-definitions.k8s.cni.cncf.io -n kube-system net-g0q7ypsh -oyaml 
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: net-g0q7ypsh
  namespace: kube-system
spec:
  config:'{
		  "cniVersion": "0.3.0",
		  "ipam": {
		    "provider": "net-g0q7ypsh.kube-system",
		    "server_socket": "/run/openvswitch/kube-ovn-daemon.sock",
		    "type": "kube-ovn"
			},
		  "master": "bond5.200",
		  "mode": "bridge",
		  "type": "macvlan"
		}'
		
		
		
[root@cd56 ~]# kubectl get iptables-fip-rules.kubeovn.io eip-p7cs4r5y 
NAME           EIP            V4IP             INTERNALIP      V6IP   READY   NATGWDP
eip-p7cs4r5y   eip-p7cs4r5y   **192.169.100.2    13.13.0.4**              true    eip-p7cs4r5y

# 网关pod中的net1就是bond5.200通过macvlan分出去的子网卡
[root@cd56 ~]# kubectl exec -it -n kube-system  vpc-nat-gw-eip-p7cs4r5y-0  /bin/bash 
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
Defaulted container "vpc-nat-gw" out of: vpc-nat-gw, vpc-nat-gw-init (init)
vpc-nat-gw-eip-p7cs4r5y-0:/kube-ovn# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: net1@if424: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 8e:1d:ec:57:6c:34 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.169.100.3/24 brd 192.169.100.255 scope global net1
       valid_lft forever preferred_lft forever
    inet **192.169.100.2/24** scope global secondary net1
       valid_lft forever preferred_lft forever
    inet6 fe80::8c1d:ecff:fe57:6c34/64 scope link 
       valid_lft forever preferred_lft forever
431: eth0@if432: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default 
    link/ether aa:cc:17:e5:61:ca brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 13.13.0.253/24 brd 13.13.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a8cc:17ff:fee5:61ca/64 scope link 
       valid_lft forever preferred_lft forever
vpc-nat-gw-eip-p7cs4r5y-0:/kube-ovn# ip r
default via 192.169.100.1 dev net1 
10.96.0.0/12 via 13.13.0.254 dev eth0 
13.13.0.0/24 dev eth0 proto kernel scope link src 13.13.0.253 
192.169.100.0/24 dev net1 proto kernel scope link src 192.169.100.3
# 生成的SNAT、DNAT规则
vpc-nat-gw-eip-p7cs4r5y-0:/kube-ovn# iptables-legacy -S -t nat 
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N DNAT_FILTER
-N EXCLUSIVE_DNAT
-N EXCLUSIVE_SNAT
-N SHARED_DNAT
-N SHARED_SNAT
-N SNAT_FILTER
-A PREROUTING -j DNAT_FILTER
-A POSTROUTING -j SNAT_FILTER
-A DNAT_FILTER -j EXCLUSIVE_DNAT
-A DNAT_FILTER -j SHARED_DNAT
**-A EXCLUSIVE_DNAT -d 192.169.100.2/32 -j DNAT --to-destination 13.13.0.4
-A EXCLUSIVE_SNAT -s 13.13.0.4/32 -j SNAT --to-source 192.169.100.2**
-A SNAT_FILTER -j EXCLUSIVE_SNAT
-A SNAT_FILTER -j SHARED_SNAT
```

# 内核Netfilter hook优先级

Linux内核中，数据包在网络层中需要经过netfilter进行处理，其中注册了若干hook函数，这些钩子函数通过 `nf_hook_ops` 结构体数组进行注册，用于在网络栈的不同阶段执行特定的操作。比如对于conntrack连接跟踪模块hook函数的注册取自 `linux v5.19 net/netfilter/nf_conntrack_proto.c` 如下：

```c
static const struct nf_hook_ops ipv4_conntrack_ops[] = {
	{
		.hook		= ipv4_conntrack_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_conntrack_local,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
	{
		.hook		= ipv4_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
};

```

其中的参数解释：

- pf (Protocol Family): 协议族，这里为 NFPROTO_IPV4，表示 IPv4 协议。
- hooknum: 钩子编号，标识网络栈中的处理点位置。例如：
    - NF_INET_PRE_ROUTING: 数据包进入系统前。
    - NF_INET_POST_ROUTING: 数据包离开系统前。
    - NF_INET_LOCAL_OUT: 本地发送的数据包。
    - NF_INET_LOCAL_IN: 接收到的本地数据包。
- priority (优先级): 决定钩子函数执行顺序，数值越小优先级越高，所有的优先级取自`include/uapi/linux/netfilter_ipv4.h`。如下：

```c
enum nf_ip_hook_priorities {
	NF_IP_PRI_FIRST = INT_MIN,
	NF_IP_PRI_RAW_BEFORE_DEFRAG = -450,
	NF_IP_PRI_CONNTRACK_DEFRAG = -400,
	NF_IP_PRI_RAW = -300,
	NF_IP_PRI_SELINUX_FIRST = -225,
	NF_IP_PRI_CONNTRACK = -200,
	NF_IP_PRI_MANGLE = -150,
	NF_IP_PRI_NAT_DST = -100,
	NF_IP_PRI_FILTER = 0,
	NF_IP_PRI_SECURITY = 50,
	NF_IP_PRI_NAT_SRC = 100,
	NF_IP_PRI_SELINUX_LAST = 225,
	NF_IP_PRI_CONNTRACK_HELPER = 300,
	NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
	NF_IP_PRI_LAST = INT_MAX,
};
```

关于IP NAT的hook函数注册在`net/netfilter/nf_nat_proto.c` ，如下：

```c
static const struct nf_hook_ops nf_nat_ipv4_ops[] = {
	/* Before packet filtering, change destination */
	{
		.hook		= nf_nat_ipv4_pre_routing,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_NAT_DST,
	},
	/* After packet filtering, change source */
	{
		.hook		= nf_nat_ipv4_out,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_NAT_SRC,
	},
	/* Before packet filtering, change destination */
	{
		.hook		= nf_nat_ipv4_local_fn,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_NAT_DST,
	},
	/* After packet filtering, change source */
	{
		.hook		= nf_nat_ipv4_local_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_NAT_SRC,
	},
};
```

可以看到conntrack hook优先级是在nat hook优先级之上的，因此linux内核中数据包在经过netfilter处理时，是先去匹配conntrack模块，如果是已建立的连接，则根据已记录跟踪的连接详情来处理数据包；如果没有相应连接，才会往下匹配NAT规则来处理数据包。

# 结论

综上所诉可以得出结论，kube-ovn中为容器内部IP绑定EIP的行为，本质上是在VPC gateway pod的iptables nat表中创建SNAT、DNAT规则，这些规则的更新并不会影响已存在的连接（比如该文情况下的icmp连接）。

# 写在最后

本文主要是记录了针对这个问题的研究过程，其中涉及kube-ovn的特性、linux kernal nat、conntrack具体的数据包处理函数，后续还会有更深入的研究，到时候再细细说来～