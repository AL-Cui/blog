---
title: 使用Keepalived挂载的VIP+Nodeport无法正常访问的解决方案
categories: Kubernetes
---

# 使用Keepalived挂载的VIP+Nodeport无法正常访问
## 问题的产生
之前在解决kube-apiserver高可用性的时候，采用了Keepalived来挂载VIP的方式，供客户通过这个VIP和相应的端口号来访问k8s集群服务和用户自己的服务。kube-proxy的proxy-mode很自然的选择了ipvs，这也是官方推荐的模式，没想到却给自己挖了坑。

事情是这样的，我们通过Keepalived给集群节点挂载了一个VIP，想让用户把自己的服务通过NodePort暴露出来，然后用户就可以通过VIP+NodePort的方式访问自己的服务。这样就没必要通过物理节点的IP+NodePort的方式访问。因为集群的节点有多个，而VIP只有一个。在自己测试的时候发现并没有问题，但是后来提交测试的时候却出了问题，kube-proxy并没有给VIP增加相应的ipvs rules规则，流量也就不能通过VIP进入集群，这是为什么呢？
## 问题分析
经过还原测试环境，发现了一个微小的变化。众所周知，***我们在使用Keepalived配置VIP的时候，不但可以配置一个IP地址，还可以给IP加上掩码***。**如果你没有配置，默认掩码就是32位。碰巧那位测试人员，把VIP的掩码配置成了24位**。那么问题就来了为什么在32位掩码情况下，kube-proxy就可以负载而24位情况小就不能了呢？问题的真相就要回归到源码上。我查看了kube-proxy的源码后，终于找出了问题的原因。原来，**kube-proxy在ipvs模式下**，发现IP并为其增加ipvs rules的方法是：

```powershell
ip route show table local type local proto kernel
```
这条shell指令被用来发现本机的IPAddress，具体源码如下：
```go
// GetLocalAddresses lists all LOCAL type IP addresses from host based on filter device.
// If dev is not specified, it's equivalent to exec:
// $ ip route show table local type local proto kernel
// 10.0.0.1 dev kube-ipvs0  scope host  src 10.0.0.1
// 10.0.0.10 dev kube-ipvs0  scope host  src 10.0.0.10
// 10.0.0.252 dev kube-ipvs0  scope host  src 10.0.0.252
// 100.106.89.164 dev eth0  scope host  src 100.106.89.164
// 127.0.0.0/8 dev lo  scope host  src 127.0.0.1
// 127.0.0.1 dev lo  scope host  src 127.0.0.1
// 172.17.0.1 dev docker0  scope host  src 172.17.0.1
// 192.168.122.1 dev virbr0  scope host  src 192.168.122.1
// Then cut the unique src IP fields,
// --> result set: [10.0.0.1, 10.0.0.10, 10.0.0.252, 100.106.89.164, 127.0.0.1, 192.168.122.1]

// If dev is specified, it's equivalent to exec:
// $ ip route show table local type local proto kernel dev kube-ipvs0
// 10.0.0.1  scope host  src 10.0.0.1
// 10.0.0.10  scope host  src 10.0.0.10
// Then cut the unique src IP fields,
// --> result set: [10.0.0.1, 10.0.0.10]

// If filterDev is specified, the result will discard route of specified device and cut src from other routes.
func (h *netlinkHandle) GetLocalAddresses(dev, filterDev string) (sets.String, error) {
	chosenLinkIndex, filterLinkIndex := -1, -1
	if dev != "" {
		link, err := h.LinkByName(dev)
		if err != nil {
			return nil, fmt.Errorf("error get device %s, err: %v", filterDev, err)
		}
		chosenLinkIndex = link.Attrs().Index
	} else if filterDev != "" {
		link, err := h.LinkByName(filterDev)
		if err != nil {
			return nil, fmt.Errorf("error get filter device %s, err: %v", filterDev, err)
		}
		filterLinkIndex = link.Attrs().Index
	}

	routeFilter := &netlink.Route{
		Table:    unix.RT_TABLE_LOCAL,
		Type:     unix.RTN_LOCAL,
		Protocol: unix.RTPROT_KERNEL,
	}
	filterMask := netlink.RT_FILTER_TABLE | netlink.RT_FILTER_TYPE | netlink.RT_FILTER_PROTOCOL

	// find chosen device
	if chosenLinkIndex != -1 {
		routeFilter.LinkIndex = chosenLinkIndex
		filterMask |= netlink.RT_FILTER_OIF
	}
	routes, err := h.RouteListFiltered(netlink.FAMILY_ALL, routeFilter, filterMask)
	if err != nil {
		return nil, fmt.Errorf("error list route table, err: %v", err)
	}
	res := sets.NewString()
	for _, route := range routes {
		if route.LinkIndex == filterLinkIndex {
			continue
		}
		if h.isIPv6 {
			if route.Dst.IP.To4() == nil && !route.Dst.IP.IsLinkLocalUnicast() {
				res.Insert(route.Dst.IP.String())
			}
		} else if route.Src != nil {
			res.Insert(route.Src.String())
		}
	}
	return res, nil
}
```
当我们给挂载的VIP配置了24位掩码时，就会发现，通过这条指令无法找到配置的VIP，找到的反而是当前网卡设备上原有的IP地址，所以Kube-proxy无法帮其增加ipvs rules。

## 问题解决方案
为什么我之前说到配置kube-proxy的时候选择了ipvs模式给自己挖了个坑呢？因为，在原先使用的iptables模式下就没有这个问题，因为iptables模式下发现IPAddress使用了另外的方法。k8s之所以推荐ipvs主要原因还是在于，**ipvs采用了hash table来存储规则，因此在规则较多的情况下，Ipvs相对iptables转发效率更高。除此以外，ipvs支持更多的LB算法**。所以，

 1. 解决方案一：**牺牲一部分性能换回iptables模式**
 2. 解决方案二：**在配置Keepalived时不给用户提供掩码配置，默认采用32位掩码**
 3. 解决方案三：**修改kube-proxy代码，一劳永逸**

## 如何修改kube-proxy的代码
问题主要是kube-proxy中ipvs模式下GetLocalAddresses方法导致的，我们主要修改这个方法即可，提供一个方案给大家参考：

```go
func (h *netlinkHandle) GetLocalAddresses(dev, filterDev string) (sets.String, error) {
	var ifaces []net.Interface
	var err error
	var filterIfaceIndex = -1

	if dev != "" {
		// get specified interface by name
		iface, err := net.InterfaceByName(dev)
		if err != nil {
			return nil, fmt.Errorf("error get device %s, err: %v", dev, err)
		}
		ifaces = []net.Interface{*iface}
	} else {
		// list all interfaces
		ifaces, err = net.Interfaces()
		if err != nil {
			return nil, fmt.Errorf("error list all interfaces: %v", err)
		}
		filterLinkIndex = link.Attrs().Index
	}
	if filterDev != "" {
		iface, err := net.InterfaceByName(filterDev)
		if err != nil {
			return nil, fmt.Errorf("error get filter device %s, err: %v", filterDev, err)
		}
		filterIfaceIndex = iface.Index
	}
	// iterate over each interface
	for _, iface := range ifaces {
		// skip filterDev
		if iface.Index == filterIfaceIndex {
			continue
		}
		addrs, err := iface.Addrs()
		if err != nil {
			return nil, fmt.Errorf("error get IP addresses of interface %s: %v", iface.Name, err)
		}
		// iterate over addresses on each interface
		for _, addr := range addrs {
			var ip net.IP
			switch v := addr.(type) {
			case *net.IPNet:
				ip = v.IP
			case *net.IPAddr:
				ip = v.IP
			}
			if ip == nil || ip.IsLinkLocalUnicast() {
				continue
			}
			if h.isIPv6 {
				if ip.To4() == nil {
					// insert IPv6 addresses
					res.Insert(ip.String())
				}
			} else {
				if ip.To4() != nil {
					// insert IPv4 addresses
					res.Insert(ip.String())
				}
			}
		}
	}

	return res, nil
}

```

