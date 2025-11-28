+++
date = '2025-11-28T03:07:22Z'
draft = false
title = 'HE Tunnel添加IPV6踩坑记'
+++

因为我的服务器没有ipv6，之前用过warp添加v6出口，本质上就是给服务器套了个代理，但我想要v6入口，昨天晚上听说 tunnelbroker 添加的v6 in v4 tunnel是公网出入ipv6，今天起来立马折腾了一下，参考了教程[HE tunnelbroker添加ipv6][1]

### 我的服务器环境

* **服务器为**：华为云Flexus L实例，以前参赛送的，非官方系统，自己重装的，记不清装的什么稀奇古怪的版本了。
* **系统版本**：自己重装的奇奇怪怪的版本，Debian 12，一直用的还行，不纠结这个，SSH提示为
  `6.1.0-35-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.137-1 (2025-05-07) x86_64`
* **网络配置**：用的不是参考教程中的 `netplan`，而是传统的 `/etc/network/interfaces`，主网卡为 `ens3`。

### 配置过程

添加的步骤很简单，在 `/etc/network/interfaces` 文件中加入以下内容，然后重启

```bash
auto he-ipv6
iface he-ipv6 inet6 v4tunnel
    # --- 隧道基础配置 ---
    mode sit
    endpoint 216.218.221.42  # HE.net 隧道服务器的IPv4
    ttl 255
    local xxxx              # 服务器公网ip
  
    # --- 隧道本身的IPv6地址配置 ---
    address xxxxx           # 分配给你的IPv6地址
    netmask 64
    gateway xxxx            # HE.net 隧道的IPv6网关
```

### 排错之路

添加之后，重启网络服务进行测试，发现：

* ping服务器v6 不通
* tcp 服务器v6 不通
* `curl ipv6.ip.sb` 不通

出口不通我能理解，我把IPv6出口禁了，设置了IPv4优先（在 `/etc/gai.conf` 里面取消 `precedence ::ffff:0:0/96 100` 的注释），所以我也没有设置v6 DNS服务器，因为v6出口对我来说意义不大，有入口就行。

#### 1. 防火墙排查

首先从`ufw`开始排查，我记得我是没设置禁止ping和v6入站的，`ufw status` 看一眼，确实是。在服务器里ping了一下自己的IPv6地址和v4隧道服务器，都没问题。

#### 2. 安全组排查

恍惚间想起来，华为云这种大厂在云服务器之前都有一层**安全组**。看了一下，果然，我的入站只允许了IPv4。于是立马调整成服务器所有协议和端口都允许v4+v6入站（平时习惯服务器上自行配置防火墙，不过还是用了安全组禁用了一些我不需要的协议入站，毕竟方便）。

调整完之后，还是不行。

#### 3. 抓包分析

继续排查，想到可以抓包看一下。之前的测试应该可以排除防火墙的问题，和我连接不到v4隧道的问题。之前在windows用wireshark和reqable抓包，但linux下还真不会，问了下AI，Debian中可以用以下命令进行抓包：

```bash
tcpdump -i ens3 proto 41  
```

开启`tcpdump`，本地`ping6`发包测试，抓包结果如下：

```
09:40:55.236556 IP tserv1.sin1.he.net > 192.168.12.2: IP6 xxxx > xxxx ICMP6, echo request, id 1, seq 12, length 40
```

乍一看没问题，这v6数据包在v4隧道里包装，拿到本机里拆成v6包，IP也是HE的。看了半天还没看出问题，`192.168.12.2` 也确实是我服务器的局域网IP。

### 找到问题与反思

思考了半天没整明白，问了下AI：

> “he-ipv6隧道接口配置里，`local`地址写的是你的公网IP，但这里数据包的目的地是局域网IP，这通常意味着你的服务器位于一个NAT设备后面（这是云服务器的标准结构）”

这下知道原因了，以前对云服务器没多少了解，或许因为以前用的都是些“小学生云”，哪有这些东西。

了解了原因就简单了，**将**  **`/etc/network/interfaces`** **中的** **`local`** **地址改成服务器的局域网IP**，问题解决。

之后去了解学习了一下云服务器为什么这么搞，因为一开始觉得“我花钱买了公网IP，云服务商加一层NAT干什么，又不是家庭宽带共享出口”。学习后知道是为了公网IP和服务器解耦，实现动态绑定与解绑，也就是现在**弹性IP**的概念，确实挺好。

但是......华为云的Flexus L实例，是捆绑套餐，IP不能变，也不能解绑，和机器是绑定的，这么看似乎又有点多此一举......

[1]: https://appscross.com/blog/how-to-configure-he-ipv6-tunnel.html
