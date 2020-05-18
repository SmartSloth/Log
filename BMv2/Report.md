*A Report To: 竹鼠*
# ping 现状
- `ping -6 -I eth0  fe80::20c:29ff:fe1b:322`效果   
用`ping -6`时必须加`-I eth0`否则报错`connect: Invalid argument`       

ICMP源 | IPv6地址 | 指令 | 效果 
- | - | - | -
本机 | stratum-eth0 | -6 -I eth0 | 可通
本机 | stratum-eth0 | ping | `connect: Invalid argument`
本机 | stratum-eth0 | -6 -I eth1 | `connect: Network is unreachable`
Leaf1 | stratum-eth0 | -6 -I eth0 | 可通
Leaf1 | stratum-eth0 | ping | `connect: Invalid argument`
Leaf1 | stratum-eth0 | -6 -I eth1 | `connect: Network is unreachable`
Leaf1 | stratum-eth0 | -6 -I eth3 | `ping: unknown iface eth3`
Leaf1 | Spine1-eth1 | -6 -I eth1 | `connect: Network is unreachable`
Spine1 | stratum-eth0 | -6 -I eth0 | 可通
h2 | stratum-eth0 | -6 -I eth0 | `ping: unknown iface eth0`
h2 | h3-eth0 | -6 -I eth0 | `ping: unknown iface eth0`
h2 | h3-eth0 | ping | 可通
h2 | stratum-eth0 | ping | `connect: Invalid argument`
h2 | 本机-eth0 | ping | `connect: Invalid argument`

# Ping研究及待决问题
*Try to explain*
1. 在`mininet>`环境下用指令ping6的时候，`-I`使用的都指的是本机onos-p4-stratum-02虚拟机的网口：eth0是真实存在且打开的，因此可以ping通；eth1是存在但是没有开启的，所以会说`Network is unreachable`；eth2开始是不存在的，所以`unknown iface`；
2. 通常情况下如果强行用`ping`指令ping一个IPv6地址，那么就会报错`Invalid argument`但是有一个神奇且费解的情况可以，见3；
3. 以h2作为一个主机代表进行分析：当`mininet> h2 ping h3`这样用主机ID进行`ping`和`ping6`是可以进行的，见图1。直接写h3的IPv6地址的情况下，不写`-I eth0`是可以通的，写了反而会报错`ping: unknown iface eth0`，见图2，那么出现问题了：
Ⅰ. 这里用的eth0是h2的eth0吗？
继续测试会发现h2不能`ping6`交换机的端口，怎么ping都不行，道理上`h2-eth1<->leaf1-eth6`是不会不通的，见图3，所以基本可以确定用的leaf1-eth6的地址有问题，而这个地址来源是本机`ifconfig`得到的。接下来查看了netcfg.json文件中的leaf1-eth6地址，这里写的是`["2001:1:2::ff/64"]`，问题出现了：
Ⅱ. mininet生成的交换机的各个端口的地址是如何生成的？
如果这时候ping这个地址，问题仍然没有解决，见图4、图5，h2的request没有reply，就是说，leaf1有可能不知道自己的端口叫这个IPv6地址，问题又来了：
Ⅲ. 交换机如何处理来自主机的ICMP或ICMPv6 Request？
Ⅳ. 交换机是否能知道自己在netcfg.json中的配置？
一个简单的总结和猜测，长得很复杂的IPv6地址应该是有全局唯一性的，这个属于全世界都能找到你的地址，但是这个地址无法应用在ping的目的参数中，但是自己配制的简单的IPv6地址是可以应用其参数的。值得注意的一点是`h2 ping leaf1`在工作的时候，源地址是`127.0.0.1`相当于走的IPv4方式，如果用`h2 ping -6 leaf1`会报错`ping: 127.0.0.1: Address family for hostname not supported`，无独有偶，`leaf1 ping leaf2`也是这个报错，这就引出最后一个问题：
Ⅴ. 这个报错产生的原因是什么？

![图1. h2 ping/ping6 h3](https://upload-images.jianshu.io/upload_images/2013499-3c782ff62e1f328a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图2. h2 ping/ping6 h3的IPv6地址](https://upload-images.jianshu.io/upload_images/2013499-7c45aefe041dae25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图3. h2 ping/ping6 leaf1-eth6](https://upload-images.jianshu.io/upload_images/2013499-330e94fa1c56bf15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图4. h2 ping/ping6 leaf1-6 in terminal](https://upload-images.jianshu.io/upload_images/2013499-6c5060be097f6113.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图5. h2 ping/ping6 leaf1-6 in wirehark](https://upload-images.jianshu.io/upload_images/2013499-c8888357e647c52d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*Report Over*
