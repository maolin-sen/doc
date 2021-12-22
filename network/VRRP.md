# 虚拟路由冗余协议(VRRP)

定义
---
---

虚拟路由冗余协议VRRP（Virtual Router Redundancy Protocol）通过把几台路由设备联合组成一台虚拟的路由设备，将虚拟路由设备的IP地址作为用户的默认网关实现与外部网络通信。当网关设备发生故障时，VRRP机制能够选举新的网关设备承担数据流量，从而保障网络的可靠通信。

目的
---
---

VRRP能够在不改变组网的情况下，采用将多台路由设备组成一个虚拟路由器，通过配置虚拟路由器的IP地址为默认网关，实现默认网关的备份。当网关设备发生故障时，VRRP机制能够选举新的网关设备承担数据流量，从而保障网络的可靠通信。

收益
---
---

在具有多播或广播能力的局域网（如以太网）中，借助VRRP能在网关设备出现故障时仍然提供高可靠的缺省链路，无需修改主机及网关设备的配置信息便可有效避免单一链路发生故障后的网络中断问题。

## VRRP原理

### VRRP概述

* VRRP路由器（VRRP Router）：运行VRRP协议的设备，它可能属于一个或多个虚拟路由器。
* 虚拟路由器（Virtual Router）：又称VRRP备份组，由一个Master设备和多个Backup设备组成，被当作一个共享局域网内主机的缺省网关。
* Master路由器（Virtual Router Master）：承担转发报文任务的VRRP设备。
* Backup路由器（Virtual Router Backup）：一组没有承担转发任务的VRRP设备，当Master设备出现故障时，它们将通过竞选成为新的Master设备。
* VRID：虚拟路由器的标识。
* 虚拟IP地址(Virtual IP Address)：虚拟路由器的IP地址，一个虚拟路由器可以有一个或多个IP地址，由用户配置。
* IP地址拥有者（IP Address Owner）：如果一个VRRP设备将虚拟路由器IP地址作为真实的接口地址，则该设备被称为IP地址拥有者。如果IP地址拥有者是可用的，通常它将成为Master。
* 虚拟MAC地址（Virtual MAC Address）：虚拟路由器根据虚拟路由器ID生成的MAC地址。一个虚拟路由器拥有一个虚拟MAC地址，格式为：00-00-5E-00-01-{VRID}(VRRP for IPv4)；00-00-5E-00-02-{VRID}(VRRP for IPv6)。当虚拟路由器回应ARP请求时，使用虚拟MAC地址，而不是接口的真实MAC地址。

### VRRP报文

VRRP协议报文用来将Master设备的优先级和状态通告给同一备份组的所有Backup设备。VRRP协议报文封装在IP报文中，发送到分配给VRRP的IP组播地址。在IP报文头中，源地址为发送报文接口的主IP地址（不是虚拟IP地址），目的地址是224.0.0.18，TTL是255，协议号是112。

### VRRP认证

* 无认证方式：设备对要发送的VRRP通告报文不进行任何认证处理，收到通告报文的设备也不进行任何认证，认为收到的都是真实的、合法的VRRP报文。
* 简单字符（Simple）认证方式：发送VRRP通告报文的设备将认证方式和认证字填充到通告报文中，而收到通告报文的设备则会将报文中的认证方式和认证字与本端配置的认证方式和认证字进行匹配。如果相同，则认为接收到的报文是合法的VRRP通告报文；否则认为接收到的报文是一个非法报文，并丢弃这个报文。
* MD5认证方式：发送VRRP通告报文的设备利用MD5算法对认证字进行加密，加密后保存在Authentication Data字段中。收到通告报文的设备会对报文中的认证方式和解密后的认证字进行匹配，检查该报文的合法性。

### VRRP工作原理

VRRP状态机
---

VRRP协议中定义了三种状态机：初始状态（Initialize）、活动状态（Master）、备份状态（Backup）。其中，只有处于Master状态的设备才可以转发那些发送到虚拟IP地址的报文。

* Initialize
  1. 该状态为VRRP不可用状态，在此状态时设备不会对VRRP报文做任何处理。
  2. 通常刚配置VRRP时或设备检测到故障时会进Initialize状态。
  3. 收到接口Up的消息后，如果设备的优先级为255，则直接成为Master设备；如果设备的优先级小于255，则会先切换至Backup状态。
* Masster：
    当VRRP设备处于Master状态时，它将会做下列工作：
  1. 定时（Advertisement Interval）发送VRRP通告报文。
  2. 以虚拟MAC地址响应对虚拟IP地址的ARP请求。
  3. 转发目的MAC地址为虚拟MAC地址的IP报文。
  4. 如果它是这个虚拟IP地址的拥有者，则接收目的IP地址为这个虚拟IP地址的IP报文。否则，丢弃这个IP报文。
  5. 如果收到比自己优先级大的报文，立即成为Backup。
  6. 如果收到与自己优先级相等的VRRP报文且本地接口IP地址小于对端接口IP，立即成为Backup。
* Backup：
    当VRRP设备处于Backup状态时，它将会做下列工作：
  1. 接收Master设备发送的VRRP通告报文，判断Master设备的状态是否正常。
  2. 对虚拟IP地址的ARP请求，不做响应。
  3. 丢弃目的IP地址为虚拟IP地址的IP报文。
  4. 如果收到优先级和自己相同或者比自己大的报文，则重置Master_Down_Interval定时器，不进一步比较IP地址。
     1. Master_Down_Interval定时器：Backup设备在该定时器超时后仍未收到通告报文，则会转换为Master状态。计算公式如下：
   Master_Down_Interval=(3*Advertisement_Interval) + Skew_time。其中，Skew_Time=(256–Priority)/256。
  5. 如果收到比自己优先级小的报文且该报文优先级是0时，定时器时间设置为Skew_time（偏移时间），如果该报文优先级不是0，丢弃报文，立刻成为Master。

