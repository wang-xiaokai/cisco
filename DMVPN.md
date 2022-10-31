DMVPN





表三 DMVPN第二阶段和第三阶段协议配置区别

| EIGRP     | DMVPN第三阶段配置                                            | DMVPN第三阶段配置                                            |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Hub配置   | interface Tunnel0<br/> no ip next-hop-self eigrp 100<br/> no ip split-horizon eigrp 100 | interface Tunnel0<br/> ip next-hop-self eigrp 100 （默认配置）<br/> ip split-horizon eigrp 100 （默认配置）<br/> ip nhrp redirect<br/> ip summary-address eigrp 100 192.168.0.0 255.255.0.0 |
| Spoke配置 | 无                                                           | interface Tunnel0<br/> ip nhrp shortcut                      |





#### 1. 单云单HUB配置实例

<p align='left'>
    <img src="https://cdn.jsdelivr.net/gh/wang-xiaokai/images/202210292035761.png" alt="单云单HUB拓扑图" title="单云单HUB拓扑图" width="70%" height="50%"  />
</p>



##### 1.1 第二阶段DMVPN实验

步骤1：基本配置初始化

（1）Hub-1路由器配置初始化

```
hostname Hub-1
interface FastEthernet1/0
 ip address 61.128.1.100 255.255.255.0
 no shutdown
interface Loopback0
 ip address 192.168.100.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（2）Spoke-1路由器配置初始化

```
hostname Spoke-1
interface Loopback0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（3）Spoke-2路由器配置初始化

```
hostname Spoke-2
interface Loopback0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（4）ASA防火墙配置初始化

```
hostname ASA
interface GigabitEthernet0/0
 nameif Outside
 security-level 0
 ip address 202.100.1.10 255.255.255.0 
 no shutdown
interface GigabitEthernet0/1
 nameif Inside
 security-level 100
 ip address 61.128.1.10 255.255.255.0
 no shutdown
access-list out extended permit icmp any any
access-group out in interface Outside
```

步骤2：配置mGRE和NHRP

（1）在ASA防火墙上放行mGRE流量

在配置IPSec VPN之前需要在ASA防火墙上放行mGRE流量。在配置IPSec VPN时，需要将下面命令删除。

```
access-list out extended permit gre host 202.100.1.1 host 61.128.1.100 
access-list out extended permit gre host 202.100.1.2 host 61.128.1.100
```

（2）Hub-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.100 255.255.255.0
 tunnel source 61.128.1.100  （Tunnel源地址，公网地址）
 tunnel mode gre multipoint  （多点GRE模式）
 tunnel key 12345  （建议配置，老版本不配置Tunnel key不会UP。用于标识隧道接口）
 ip nhrp network-id 10  （激活nhrp，所有的设备需要配置相同的ID）
 ip nhrp authentication cisco  （可选配置：启用nhrp认证）
 ip nhrp map multicast dynamic  （动态接收NHRP的组播映射）
show run interface tunnel 0  （查看已经完成的配置）
```

（3）Spoke-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel source 202.100.1.1
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100 (所有spokes需要静态配置hub映射，映射中心站点的隧道虚拟IP地址到中心站点的公网地址IP地址。有了这个映射，分支站点才能访问中心站点)
 ip nhrp map multicast 61.128.1.100  （所有spokes需要手动映射组播到hub的公网地址，便于后续spoke和hub之间建立动态路由协议邻居）
 ip nhrp nhs 172.16.1.100  （配置nhrp server地址，spoke启动以后会到这个服务器上注册自己的虚拟隧道地址到公网地址。nhs就是NHRP服务器）
show run interface tunnel 0  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-1通过tunnel口已经可以成功访问到HUB-1）
```

（4）Spoke-2路由器配配置

```
第一种配置 （hub-1为静态Global地址的情况）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map multicast 61.128.1.100  
 ip nhrp nhs 172.16.1.100  
show run interface tunnel 0  
ping 172.16.1.100 

第二种配置 （hub-1为动态Global地址的情况。模拟使用DDNS。）
ip domain lookup 
ip name-server 114.114.114.114
ip host hub-1.cloudpbc.cn 61.128.1.100
ping hub-1.cloudpbc.cn （测试）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
show run interface tunnel 0  
ping 172.16.1.100

*DMVPN静态Global和动态域名解析配置命令的区别：
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
取代传统的如下三行命令配置
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map multicast 61.128.1.100  
 ip nhrp nhs 172.16.1.100 
```

（5）测试NHRP

```
show ip nhrp  （在HUB，Spoke中查看NHRP的注册信息）
确认了NHRP在中心站点和分支站点已经正常工作以后，在分支站点一（Spoke1）通过ping测试访问分支站点二（Spoke2），主要目的是检查NHRP的动态解析功能。
Spoke-1#ping 172.16.1.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.2, timeout is 2 seconds:
!.!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 8/27/80 ms
```

步骤3：配置动态路由协议

（1）Hub-1路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.100.0 0.0.0.255
```

（2）Spoke-1路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.1.0 0.0.0.255
```

（3）Spoke-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

步骤4：调整和测试EIGRP

（1）在Hub-1路由器上调整EIGRP

```
interface Tunnel0
 no ip split-horizon eigrp 100 （关闭水平分割特性。注意只配置no ip split-horizon，则关闭RIP而不是EIGRP）
 no ip next-hop-self eigrp 100 （关闭EIGRP的next-hop-self特性）
```

（2）在分支站点一（Spoke1）上查看EIGRP学习的路由

```
Spoke-1#show ip route eigrp 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 202.100.1.10 to network 0.0.0.0

D     192.168.2.0/24 [90/28288000] via 172.16.1.2, 00:05:12, Tunnel0
D     192.168.100.0/24 [90/26882560] via 172.16.1.100, 00:05:28, Tunnel0
```

步骤5：配置IPSec VPN

（1）在ASA防火墙上放行mGRE流量

当配置IPSec VPN时，mGRE流量会被加密。因此仅仅需要放行isakmp UDP 500和esp的流量。

```
no access-list out extended permit gre host 202.100.1.1 host 61.128.1.100 
no access-list out extended permit gre host 202.100.1.2 host 61.128.1.100
access-list out extended permit udp host 202.100.1.1 host 61.128.1.100 eq isakmp 
access-list out extended permit udp host 202.100.1.2 host 61.128.1.100 eq isakmp 
access-list out extended permit esp host 202.100.1.1 host 61.128.1.100
access-list out extended permit esp host 202.100.1.2 host 61.128.1.100 
```

（2）在HUB-1，Spoke1和Spoke2上都配置相同的配置

```
crypto isakmp policy 10
 authentication pre-share 
crypto isakmp key cisco address 0.0.0.0 0.0.0.0
crypto ipsec transform-set cisco esp-des esp-md5-hmac 
 mode transport 
crypto ipsec  profile dmvpn-profile
 set transform-set cisco
interface Tunnel0
 ip mtu 1400
 tunnel protection ipsec profile dmvpn-profile
```

步骤6：测试并查看DMVPN的状态

```
（1）在Spoke1上使用ping触发分支站点间的流量
Spoke-1#ping 192.168.2.1 source 192.168.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 16/20/24 ms
（2）查看IPsec的状态
show crypto ipsec sa
show crypto session
```



##### 1.2 第三阶段DMVPN实验

步骤1：基本配置初始化

（1）Hub-1路由器配置初始化

```
hostname Hub-1
interface FastEthernet1/0
 ip address 61.128.1.100 255.255.255.0
 no shutdown
interface Loopback0
 ip address 192.168.100.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（2）Spoke-1路由器配置初始化

```
hostname Spoke-1
interface Loopback0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（3）Spoke-2路由器配置初始化

```
hostname Spoke-2
interface Loopback0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（4）ASA防火墙配置初始化

```
hostname ASA
interface GigabitEthernet0/0
 nameif Outside
 security-level 0
 ip address 202.100.1.10 255.255.255.0 
 no shutdown
interface GigabitEthernet0/1
 nameif Inside
 security-level 100
 ip address 61.128.1.10 255.255.255.0
 no shutdown
access-list out extended permit icmp any any
access-group out in interface Outside
```

步骤2：配置mGRE和NHRP

（1）在ASA防火墙上放行mGRE流量

在配置IPSec VPN之前需要在ASA防火墙上放行mGRE流量。在配置IPSec VPN时，需要将下面命令删除。

```
access-list out extended permit gre host 202.100.1.1 host 61.128.1.100 
access-list out extended permit gre host 202.100.1.2 host 61.128.1.100
```

（2）Hub-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.100 255.255.255.0
 tunnel source 61.128.1.100  （Tunnel源地址，公网地址）
 tunnel mode gre multipoint  （多点GRE模式）
 tunnel key 12345  （建议配置，老版本不配置Tunnel key不会UP。用于标识隧道接口）
 ip nhrp network-id 10  （激活nhrp，所有的设备需要配置相同的ID）
 ip nhrp authentication cisco  （可选配置：启用nhrp认证）
 ip nhrp map multicast dynamic  （动态接收NHRP的组播映射）
 ip nhrp redirect   （第三阶段的DMVPN需要在中心站点启用NHRP重定向，这样中心站点才会给分支站点发送NHRP重定向信息来优化下一跳）
show run interface tunnel 0  （查看已经完成的配置）
```

（3）Spoke-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel source 202.100.1.1
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100 (所有spokes需要静态配置hub映射，映射中心站点的隧道虚拟IP地址到中心站点的公网地址IP地址。有了这个映射，分支站点才能访问中心站点)
 ip nhrp map multicast 61.128.1.100  （所有spokes需要手动映射组播到hub的公网地址，便于后续spoke和hub之间建立动态路由协议邻居）
 ip nhrp nhs 172.16.1.100  （配置nhrp server地址，spoke启动以后会到这个服务器上注册自己的虚拟隧道地址到公网地址。nhs就是NHRP服务器）
 ip nhrp shortcut  （第三阶段的DMVPN需要在所有分支站点启用NHRP短路，这样才能在分支站点间直接建立隧道）
show run interface tunnel 0  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-1通过tunnel口已经可以成功访问到HUB-1）
```

（4）Spoke-2路由器配配置

```
第一种配置 （hub-1为静态Global地址的情况）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map multicast 61.128.1.100  
 ip nhrp nhs 172.16.1.100
 ip nhrp shortcut
show run interface tunnel 0  
ping 172.16.1.100 

第二种配置 （hub-1为动态Global地址的情况。模拟使用DDNS。）
ip domain lookup 
ip name-server 114.114.114.114
ip host hub-1.cloudpbc.cn 61.128.1.100
ping hub-1.cloudpbc.cn （测试）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp shortcut
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
show run interface tunnel 0  
ping 172.16.1.100

*DMVPN静态Global和动态域名解析配置命令的区别：
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
取代传统的如下三行命令配置
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map multicast 61.128.1.100  
 ip nhrp nhs 172.16.1.100 
```

（5）测试NHRP

```
show ip nhrp  （在HUB，Spoke中查看NHRP的注册信息）
确认了NHRP在中心站点和分支站点已经正常工作以后，在分支站点一（Spoke1）通过ping测试访问分支站点二（Spoke2），主要目的是检查NHRP的动态解析功能。
Spoke-1#ping 172.16.1.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.2, timeout is 2 seconds:
!.!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 8/27/80 ms
```

步骤3：配置动态路由协议

（1）Hub-1路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.100.0 0.0.0.255
```

（2）Spoke-1路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.1.0 0.0.0.255
```

（3）Spoke-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

步骤4：调整和测试EIGRP

（1）在Hub-1路由器上调整EIGRP

```
interface Tunnel0
 ip summary-address eigrp 100 192.168.0.0 255.255.0.0
第三阶段的DMVPN不再需要关闭水平分割（no ip split-horizon eigrp 100），也不在需要关闭关闭EIGRP的next-hop-self特性来优化路由（no ip next-hop-self eigrp 100）。只需要中心给所有的分支站点发送一条汇总路由。
```

（2）在分支站点一（Spoke1）上查看EIGRP学习的路由

```
Spoke-1#show ip route eigrp 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 202.100.1.10 to network 0.0.0.0

D     192.168.2.0/24 [90/28288000] via 172.16.1.2, 00:05:12, Tunnel0
D     192.168.100.0/24 [90/26882560] via 172.16.1.100, 00:05:28, Tunnel0
```

步骤5：配置IPSec VPN

（1）在ASA防火墙上放行mGRE流量

当配置IPSec VPN时，mGRE流量会被加密。因此仅仅需要放行isakmp UDP 500和esp的流量。

```
no access-list out extended permit gre host 202.100.1.1 host 61.128.1.100 
no access-list out extended permit gre host 202.100.1.2 host 61.128.1.100
access-list out extended permit udp host 202.100.1.1 host 61.128.1.100 eq isakmp 
access-list out extended permit udp host 202.100.1.2 host 61.128.1.100 eq isakmp 
access-list out extended permit esp host 202.100.1.1 host 61.128.1.100
access-list out extended permit esp host 202.100.1.2 host 61.128.1.100 
```

（2）在HUB-1，Spoke1和Spoke2上都配置相同的配置

```
crypto isakmp policy 10
 authentication pre-share 
crypto isakmp key cisco address 0.0.0.0 0.0.0.0
crypto ipsec transform-set cisco esp-des esp-md5-hmac 
 mode transport 
crypto ipsec  profile dmvpn-profile
 set transform-set cisco
interface Tunnel0
 ip mtu 1400
 tunnel protection ipsec profile dmvpn-profile
```

步骤6：测试并查看DMVPN的状态

```
（1）在Spoke1上使用ping触发分支站点间的流量
Spoke-1#ping 192.168.2.1 source 192.168.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 16/20/24 ms
（2）查看IPsec的状态
show crypto ipsec sa
show crypto session
```



#### 2. 单云双HUB配置实例



<img src="https://cdn.jsdelivr.net/gh/wang-xiaokai/images/202210301037607.png" alt="单云双HUB拓扑图" title="单云双HUB拓扑图" width="70%" height="50%" />

<br/>

##### 2.1 第二阶段DMVPN实验

步骤1：基本配置初始化

（1）Hub-1路由器配置初始化

```
hostname Hub-1
interface FastEthernet1/0
 ip address 61.128.1.100 255.255.255.0
 no shutdown
interface FastEthernet2/0
 ip address 192.168.100.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（2）Hub-2路由器配置初始化

```
hostname Hub-2
interface FastEthernet1/0
 ip address 61.128.1.200 255.255.255.0
 no shutdown
interface FastEthernet2/0
 ip address 192.168.100.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（3）Spoke-1路由器配置初始化

```
hostname Spoke-1
interface Loopback0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（4）Spoke-2路由器配置初始化

```
hostname Spoke-2
interface Loopback0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（5）Spoke-3路由器配置初始化

```
hostname Spoke-3
interface Loopback0
 ip address 192.168.3.1 255.255.255.0
 no shutdown
interface FastEthernet1/0
 ip address 61.128.1.3 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（6）ASA防火墙配置初始化

```
hostname ASA
interface GigabitEthernet0/0
 nameif Outside
 security-level 0
 ip address 202.100.1.10 255.255.255.0 
 no shutdown
interface GigabitEthernet0/1
 nameif Inside
 security-level 100
 ip address 61.128.1.10 255.255.255.0
 no shutdown
access-list out extended permit icmp any any
access-group out in interface Outside
```

步骤2：配置mGRE和NHRP

（1）Hub-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.100 255.255.255.0
 tunnel source 61.128.1.100  
 tunnel mode gre multipoint  
 tunnel key 12345  
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic 
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.200
show run interface tunnel 0  
```

（2）Hub-2路由器配配置

```
interface Tunnel0
 ip address 172.16.1.200 255.255.255.0
 tunnel source 61.128.1.200  
 tunnel mode gre multipoint  
 tunnel key 12345  
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic 
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map multicast 61.128.1.100
show run interface tunnel 0  
```

（3）Spoke-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel source 202.100.1.1
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100
 ip nhrp nhs 172.16.1.200
show run interface tunnel 0  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-1通过tunnel口已经可以成功访问到HUB-1）
```

（4）Spoke-2路由器配配置

```
第一种配置 （hub-1为静态Global地址的情况）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100 
 ip nhrp nhs 172.16.1.200
show run interface tunnel 0  
ping 172.16.1.100 

第二种配置 （hub-1为动态Global地址的情况。模拟使用DDNS。）
ip domain lookup 
ip name-server 114.114.114.114
ip host hub-1.cloudpbc.cn 61.128.1.100
ip host hub-2.cloudpbc.cn 61.128.1.200
ping hub-1.cloudpbc.cn （测试）
ping hub-2.cloudpbc.cn （测试）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
 ip nhrp nhs dynamic nbma hub-2.cloudpbc.cn multicast
show run interface tunnel 0  
ping 172.16.1.100

*DMVPN静态Global和动态域名解析配置命令的区别：
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
取代传统的如下三行命令配置
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map multicast 61.128.1.100  
 ip nhrp nhs 172.16.1.100 
```

（5）Spoke-3路由器配配置

```
interface Tunnel0
 ip address 172.16.1.3 255.255.255.0
 tunnel source 61.128.1.3
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100
 ip nhrp nhs 172.16.1.200
show run interface tunnel 0  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-3通过tunnel口已经可以成功访问到HUB-1）
```

步骤3：配置动态路由协议

（1）Hub-1和Hub-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.100.0 0.0.0.255
```

（2）Spoke-1路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.1.0 0.0.0.255
```

（3）Spoke-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

（4）Spoke-3路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.3.0 0.0.0.255
```

步骤4：调整和测试EIGRP

在Hub-1和Hub2路由器上调整EIGRP

```
interface Tunnel0
 no ip split-horizon eigrp 100 （关闭水平分割特性。注意只配置no ip split-horizon，则关闭RIP而不是EIGRP）
 no ip next-hop-self eigrp 100 （关闭EIGRP的next-hop-self特性）
```


步骤5：配置IPSec VPN

（1）在ASA防火墙上放行isakmp UDP 500和esp的流量

```
object-group network Outside-DMVPN-Address
 network-object host 202.100.1.1
 network-object host 202.100.1.2
object-group network Inside-DMVPN-Address
 network-object host 61.128.1.100
 network-object host 61.128.1.200
 network-object host 61.128.1.3
object-group service DMVPN-IPSec
 service-object udp destination eq isakmp 
 service-object esp 
access-list out extended permit object-group DMVPN-IPSec object-group Outside-DMVPN-Address object-group Inside-DMVPN-Address 
```

（2）在HUB-1，HUB-2，Spoke1，Spoke2和Spoke3上都配置相同的配置

```
crypto isakmp policy 10
 authentication pre-share 
crypto isakmp key cisco address 0.0.0.0 0.0.0.0
crypto ipsec transform-set cisco esp-des esp-md5-hmac 
 mode transport 
crypto ipsec  profile dmvpn-profile
 set transform-set cisco
interface Tunnel0
 ip mtu 1400
 tunnel protection ipsec profile dmvpn-profile
```

步骤6：测试并查看DMVPN的状态

（1）测试NHRP

```
show ip nhrp  （在HUB，Spoke中查看NHRP的注册信息）
确认了NHRP在中心站点和分支站点已经正常工作以后，在分支站点一（Spoke1）通过ping测试访问分支站点二（Spoke3），主要目的是检查NHRP的动态解析功能。
Spoke-1#ping 172.16.1.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.2, timeout is 2 seconds:
!.!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 8/27/80 ms
```

（2）在分支站点一（Spoke1）上查看EIGRP学习的路由

```
Spoke-1#show ip route eigrp 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 202.100.1.10 to network 0.0.0.0

D     192.168.2.0/24 [90/28288000] via 172.16.1.2, 00:02:01, Tunnel0
                     [90/28288000] via 172.16.1.2, 00:02:01, Tunnel0
D     192.168.3.0/24 [90/28288000] via 172.16.1.3, 00:01:58, Tunnel0
                     [90/28288000] via 172.16.1.3, 00:01:58, Tunnel0
D     192.168.100.0/24 [90/26882560] via 172.16.1.200, 00:02:04, Tunnel0
                       [90/26882560] via 172.16.1.100, 00:02:04, Tunnel0
```

（3）在分支站点一（Spoke1）上查看IPSec状态
```
1）在Spoke1上使用ping触发分支站点间的流量
Spoke-1#ping 192.168.3.1 source 192.168.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 16/20/24 ms
2）查看IPSec的状态
Spoke-1#show crypto ipsec sa
 local  ident (addr/mask/prot/port): (202.100.1.1/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (61.128.1.3/255.255.255.255/47/0)
   current_peer 61.128.1.3 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 94, #pkts encrypt: 94, #pkts digest: 94
    #pkts decaps: 95, #pkts decrypt: 95, #pkts verify: 95
    
Spoke-1#show crypto session
Interface: Tunnel0
Session status: UP-ACTIVE     
Peer: 61.128.1.3 port 500 
  IKEv1 SA: local 202.100.1.1/500 remote 61.128.1.3/500 Active 
  IKEv1 SA: local 202.100.1.1/500 remote 61.128.1.3/500 Active 
  IPSEC FLOW: permit 47 host 202.100.1.1 host 61.128.1.3 
        Active SAs: 2, origin: crypto map

```

##### 2.2 第三阶段DMVPN实验

步骤1：基本配置初始化

（1）Hub-1路由器配置初始化

```
hostname Hub-1
interface FastEthernet1/0
 ip address 61.128.1.100 255.255.255.0
 no shutdown
interface FastEthernet2/0
 ip address 192.168.100.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（2）Hub-2路由器配置初始化

```
hostname Hub-2
interface FastEthernet1/0
 ip address 61.128.1.200 255.255.255.0
 no shutdown
interface FastEthernet2/0
 ip address 192.168.100.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（3）Spoke-1路由器配置初始化

```
hostname Spoke-1
interface Loopback0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（4）Spoke-2路由器配置初始化

```
hostname Spoke-2
interface Loopback0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（5）Spoke-3路由器配置初始化

```
hostname Spoke-3
interface Loopback0
 ip address 192.168.3.1 255.255.255.0
 no shutdown
interface FastEthernet1/0
 ip address 61.128.1.3 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（6）ASA防火墙配置初始化

```
hostname ASA
interface GigabitEthernet0/0
 nameif Outside
 security-level 0
 ip address 202.100.1.10 255.255.255.0 
 no shutdown
interface GigabitEthernet0/1
 nameif Inside
 security-level 100
 ip address 61.128.1.10 255.255.255.0
 no shutdown
access-list out extended permit icmp any any
access-group out in interface Outside
```

步骤2：配置mGRE和NHRP

（1）Hub-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.100 255.255.255.0
 tunnel source 61.128.1.100  
 tunnel mode gre multipoint  
 tunnel key 12345  
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic 
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.200
 ip nhrp redirect
show run interface tunnel 0  
```

（2）Hub-2路由器配配置

```
interface Tunnel0
 ip address 172.16.1.200 255.255.255.0
 tunnel source 61.128.1.200  
 tunnel mode gre multipoint  
 tunnel key 12345  
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic 
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map multicast 61.128.1.100
 ip nhrp redirect
show run interface tunnel 0  
```

（3）Spoke-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel source 202.100.1.1
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100
 ip nhrp nhs 172.16.1.200
 ip nhrp shortcut
show run interface tunnel 0  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-1通过tunnel口已经可以成功访问到HUB-1）
```

（4）Spoke-2路由器配配置

```
第一种配置 （hub-1为静态Global地址的情况）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100 
 ip nhrp nhs 172.16.1.200
 ip nhrp shortcut
show run interface tunnel 0  
ping 172.16.1.100 

第二种配置 （hub-1为动态Global地址的情况。模拟使用DDNS。）
ip domain lookup 
ip name-server 114.114.114.114
ip host hub-1.cloudpbc.cn 61.128.1.100
ip host hub-2.cloudpbc.cn 61.128.1.200
ping hub-1.cloudpbc.cn （测试）
ping hub-2.cloudpbc.cn （测试）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
 ip nhrp nhs dynamic nbma hub-2.cloudpbc.cn multicast
 ip nhrp shortcut
show run interface tunnel 0  
ping 172.16.1.100

*DMVPN静态Global和动态域名解析配置命令的区别：
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
取代传统的如下三行命令配置
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map multicast 61.128.1.100  
 ip nhrp nhs 172.16.1.100 
```

（5）Spoke-3路由器配配置

```
interface Tunnel0
 ip address 172.16.1.3 255.255.255.0
 tunnel source 61.128.1.3
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100
 ip nhrp nhs 172.16.1.200
 ip nhrp shortcut
show run interface tunnel 0  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-3通过tunnel口已经可以成功访问到HUB-1）
```

步骤3：配置动态路由协议

（1）Hub-1和Hub-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.100.0 0.0.0.255
 
```

（2）Spoke-1路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.1.0 0.0.0.255
```

（3）Spoke-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

（4）Spoke-3路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.3.0 0.0.0.255
```

步骤4：调整和测试EIGRP

在Hub-1和Hub2路由器上调整EIGRP

```
interface Tunnel0
 ip summary-address eigrp 100 192.168.0.0 255.255.0.0
第三阶段的DMVPN不再需要关闭水平分割（no ip split-horizon eigrp 100），也不在需要关闭关闭EIGRP的next-hop-self特性来优化路由（no ip next-hop-self eigrp 100）。只需要中心给所有的分支站点发送一条汇总路由。
```


步骤5：配置IPSec VPN

（1）在ASA防火墙上放行isakmp UDP 500和esp的流量

```
object-group network Outside-DMVPN-Address
 network-object host 202.100.1.1
 network-object host 202.100.1.2
object-group network Inside-DMVPN-Address
 network-object host 61.128.1.100
 network-object host 61.128.1.200
 network-object host 61.128.1.3
object-group service DMVPN-IPSec
 service-object udp destination eq isakmp 
 service-object esp 
access-list out extended permit object-group DMVPN-IPSec object-group Outside-DMVPN-Address object-group Inside-DMVPN-Address 
```

（2）在HUB-1，HUB-2，Spoke1，Spoke2和Spoke3上都配置相同的配置

```
crypto isakmp policy 10
 authentication pre-share 
crypto isakmp key cisco address 0.0.0.0 0.0.0.0
crypto ipsec transform-set cisco esp-des esp-md5-hmac 
 mode transport 
crypto ipsec  profile dmvpn-profile
 set transform-set cisco
interface Tunnel0
 ip mtu 1400
 tunnel protection ipsec profile dmvpn-profile
```

步骤6：测试并查看DMVPN的状态

（1）测试NHRP

```
show ip nhrp  （在HUB，Spoke中查看NHRP的注册信息）
确认了NHRP在中心站点和分支站点已经正常工作以后，在分支站点一（Spoke1）通过ping测试访问分支站点二（Spoke3），主要目的是检查NHRP的动态解析功能。
Spoke-1#ping 172.16.1.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.2, timeout is 2 seconds:
!.!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 8/27/80 ms
```

（2）在分支站点一（Spoke1）上查看EIGRP学习的路由

```
Spoke-1#show ip route eigrp 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 202.100.1.10 to network 0.0.0.0

D     192.168.0.0/16 [90/26882560] via 172.16.1.200, 00:02:19, Tunnel0
                     [90/26882560] via 172.16.1.100, 00:02:19, Tunnel0
```

（3）在分支站点一（Spoke1）上查看IPSec状态
```
1）在Spoke1上使用ping触发分支站点间的流量
Spoke-1#ping 192.168.3.1 source 192.168.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 16/20/24 ms
2）查看IPSec的状态
Spoke-1#show crypto ipsec sa
 local  ident (addr/mask/prot/port): (202.100.1.1/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (61.128.1.3/255.255.255.255/47/0)
   current_peer 61.128.1.3 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 94, #pkts encrypt: 94, #pkts digest: 94
    #pkts decaps: 95, #pkts decrypt: 95, #pkts verify: 95

Spoke-1#show crypto session
Interface: Tunnel0
Session status: UP-ACTIVE     
Peer: 61.128.1.3 port 500 
  IKEv1 SA: local 202.100.1.1/500 remote 61.128.1.3/500 Active 
  IKEv1 SA: local 202.100.1.1/500 remote 61.128.1.3/500 Active 
  IPSEC FLOW: permit 47 host 202.100.1.1 host 61.128.1.3 
        Active SAs: 2, origin: crypto map

```

#### 3. 单云树状配置实例



<img src="https://cdn.jsdelivr.net/gh/wang-xiaokai/images/202210311608947.png" alt="树状拓扑图" title="树状拓扑图" width="80%" height="80%" />



##### 3.1 第二阶段DMVPN实验
步骤1：基本配置初始化

（1）Hub-1路由器配置初始化

```
hostname Hub-1
interface FastEthernet1/0
 ip address 61.128.1.100 255.255.255.0
 no shutdown
interface FastEthernet2/0
 ip address 192.168.100.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
ip route 61.128.2.0 255.255.255.0 61.128.1.3
```

（2）Hub-2路由器配置初始化

```
hostname Hub-2
interface FastEthernet1/0
 ip address 61.128.1.200 255.255.255.0
 no shutdown
interface FastEthernet2/0
 ip address 192.168.100.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
ip route 61.128.2.0 255.255.255.0 61.128.1.3
```

（3）Spoke-1路由器配置初始化

```
hostname Spoke-1
interface Loopback0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（4）Spoke-2路由器配置初始化

```
hostname Spoke-2
interface Loopback0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（5）Spoke-3路由器配置初始化

```
hostname Spoke-3
interface Loopback0
 ip address 192.168.3.1 255.255.255.0
 no shutdown
interface FastEthernet1/0
 ip address 61.128.1.3 255.255.255.0
 no shutdown
interface FastEthernet3/0
 ip address 61.128.2.3 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（6）Spoke-4路由器配置初始化

```
hostname Spoke-4
interface Loopback0
 ip address 192.168.4.1 255.255.255.0
 no shutdown
interface FastEthernet3/0
 ip address 61.128.2.4 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.2.3
```

（7）ASA防火墙配置初始化

```
hostname ASA
interface GigabitEthernet0/0
 nameif Outside
 security-level 0
 ip address 202.100.1.10 255.255.255.0 
 no shutdown
interface GigabitEthernet0/1
 nameif Inside
 security-level 100
 ip address 61.128.1.10 255.255.255.0
 no shutdown
access-list out extended permit icmp any any
access-group out in interface Outside
route Inside 61.128.2.0 255.255.255.0 61.128.1.3
```

步骤2：配置mGRE和NHRP

（1）Hub-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.100 255.255.255.0
 tunnel source 61.128.1.100  
 tunnel mode gre multipoint  
 tunnel key 12345  
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic 
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.200
show run interface tunnel 0  
```

（2）Hub-2路由器配配置

```
interface Tunnel0
 ip address 172.16.1.200 255.255.255.0
 tunnel source 61.128.1.200  
 tunnel mode gre multipoint  
 tunnel key 12345  
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic 
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map multicast 61.128.1.100
show run interface tunnel 0  
```

（3）Spoke-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel source 202.100.1.1
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100
 ip nhrp nhs 172.16.1.200
show run interface tunnel 0  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-1通过tunnel口已经可以成功访问到HUB-1）
```

（4）Spoke-2路由器配配置

```
第一种配置 （hub-1为静态Global地址的情况）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100 
 ip nhrp nhs 172.16.1.200
show run interface tunnel 0  
ping 172.16.1.100 

第二种配置 （hub-1为动态Global地址的情况。模拟使用DDNS。）
ip domain lookup 
ip name-server 114.114.114.114
ip host hub-1.cloudpbc.cn 61.128.1.100
ip host hub-2.cloudpbc.cn 61.128.1.200
ping hub-1.cloudpbc.cn （测试）
ping hub-2.cloudpbc.cn （测试）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
 ip nhrp nhs dynamic nbma hub-2.cloudpbc.cn multicast
show run interface tunnel 0  
ping 172.16.1.100

*DMVPN静态Global和动态域名解析配置命令的区别：
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
取代传统的如下三行命令配置
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map multicast 61.128.1.100  
 ip nhrp nhs 172.16.1.100 
```

（5）Spoke-3路由器配配置

```
interface Tunnel0
 ip address 172.16.1.3 255.255.255.0
 tunnel source 61.128.1.3
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100
 ip nhrp nhs 172.16.1.200
interface Tunnel1
 ip address 172.16.2.3 255.255.255.0
 tunnel source 61.128.2.3
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic
show run interface tunnel 0  （查看已经完成的配置）
show run interface tunnel 1  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-3通过tunnel口已经可以成功访问到HUB-1）
```

（5）Spoke-4路由器配配置

```
interface Tunnel1
 ip address 172.16.2.4 255.255.255.0
 tunnel source 61.128.2.4
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10  
 ip nhrp authentication cisco
 ip nhrp map 172.16.2.3 61.128.2.3
 ip nhrp map multicast 61.128.2.3
 ip nhrp nhs 172.16.2.3
```

步骤3：配置动态路由协议

（1）Hub-1和Hub-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.100.0 0.0.0.255
```

（2）Spoke-1路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.1.0 0.0.0.255
```

（3）Spoke-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

（4）Spoke-3路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 172.16.2.0 0.0.0.255
 network 192.168.3.0 0.0.0.255
```

（4）Spoke-4路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.2.0 0.0.0.255
 network 192.168.4.0 0.0.0.255
```

步骤4：调整和测试EIGRP

（1）在Hub-1和Hub2路由器上调整EIGRP

```
interface Tunnel0
 no ip split-horizon eigrp 100 （关闭水平分割特性。注意只配置no ip split-horizon，则关闭RIP而不是EIGRP）
 no ip next-hop-self eigrp 100 （关闭EIGRP的next-hop-self特性）
```

（2）在Spoke-3路由器上调整EIGRP

```
interface Tunnel1
 no ip split-horizon eigrp 100 
 no ip next-hop-self eigrp 100 
```

步骤5：配置IPSec VPN

（1）在ASA防火墙上放行isakmp UDP 500和esp的流量

```
object-group network Outside-DMVPN-Address
 network-object host 202.100.1.1
 network-object host 202.100.1.2
object-group network Inside-DMVPN-Address
 network-object host 61.128.1.100
 network-object host 61.128.1.200
 network-object host 61.128.1.3
 network-object host 61.128.2.3
 network-object host 61.128.2.4
object-group service DMVPN-IPSec
 service-object udp destination eq isakmp 
 service-object esp 
access-list out extended permit object-group DMVPN-IPSec object-group Outside-DMVPN-Address object-group Inside-DMVPN-Address 
```

（2）在HUB-1，HUB-2，Spoke1，Spoke2，Spoke3和Spoke4上都配置相同的配置

```
crypto isakmp policy 10
 authentication pre-share 
crypto isakmp key cisco address 0.0.0.0 0.0.0.0
crypto ipsec transform-set cisco esp-des esp-md5-hmac 
 mode transport 
crypto ipsec profile dmvpn-profile
 set transform-set cisco
```

（3）将ipsec profile配置调用在Tunnel接口上

在HUB-1，HUB-2，Spoke1，Spoke2和Spoke3上都配置相同的配置

```
interface Tunnel0
 ip mtu 1400
 tunnel protection ipsec profile dmvpn-profile
```

在Spoke3和Spoke4上配置相同的配置

```
interface Tunnel1
 ip mtu 1400
 tunnel protection ipsec profile dmvpn-profile
```

步骤6：测试并查看DMVPN的状态

（1）测试NHRP

```
show ip nhrp  （在HUB，Spoke中查看NHRP的注册信息）
确认了NHRP在中心站点和分支站点已经正常工作以后，在分支站点一（Spoke1）通过ping测试访问分支站点四（Spoke4），主要目的是检查NHRP的动态解析功能。
Spoke-1#ping 172.16.2.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.2, timeout is 2 seconds:
!.!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 8/27/80 ms
```

（2）在分支站点一（Spoke1）上查看EIGRP学习的路由

```
Spoke-1#show ip route eigrp
*Oct 30 11:05:27.619: %SYS-5-CONFIG_I: Configured from console by console
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 202.100.1.10 to network 0.0.0.0

      172.16.0.0/16 is variably subnetted, 3 subnets, 2 masks
D        172.16.2.0/24 [90/29440000] via 172.16.1.3, 00:01:39, Tunnel0
                       [90/29440000] via 172.16.1.3, 00:01:39, Tunnel0
D     192.168.2.0/24 [90/28288000] via 172.16.1.2, 00:01:52, Tunnel0
                     [90/28288000] via 172.16.1.2, 00:01:52, Tunnel0
D     192.168.3.0/24 [90/28288000] via 172.16.1.3, 00:01:47, Tunnel0
                     [90/28288000] via 172.16.1.3, 00:01:47, Tunnel0
D     192.168.4.0/24 [90/29568000] via 172.16.1.3, 00:01:34, Tunnel0
                     [90/29568000] via 172.16.1.3, 00:01:34, Tunnel0
D     192.168.100.0/24 [90/26882560] via 172.16.1.200, 00:01:59, Tunnel0
                       [90/26882560] via 172.16.1.100, 00:01:59, Tunnel0
```

（3）在分支站点一（Spoke1）上查看IPSec状态
```
1）在Spoke1上使用ping触发分支站点间的流量
Spoke-1#ping 192.168.4.1 source 192.168.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.4.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 20/33/160 ms
2）查看IPSec的状态
Spoke-1#show crypto session
Interface: Tunnel0
Session status: UP-ACTIVE     
Peer: 61.128.1.3 port 500 
  IKEv1 SA: local 202.100.1.1/500 remote 61.128.1.3/500 Active 
  IPSEC FLOW: permit 47 host 202.100.1.1 host 61.128.1.3 
        Active SAs: 2, origin: crypto map

Spoke-1#show crypto ipsec sa
   local  ident (addr/mask/prot/port): (202.100.1.1/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (61.128.1.3/255.255.255.255/47/0)
   current_peer 61.128.1.3 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 97, #pkts encrypt: 97, #pkts digest: 97
    #pkts decaps: 99, #pkts decrypt: 99, #pkts verify: 99 
```

##### 3.2 第三阶段DMVPN实验
步骤1：基本配置初始化

（1）Hub-1路由器配置初始化

```
hostname Hub-1
interface FastEthernet1/0
 ip address 61.128.1.100 255.255.255.0
 no shutdown
interface FastEthernet2/0
 ip address 192.168.100.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
ip route 61.128.2.0 255.255.255.0 61.128.1.3
```

（2）Hub-2路由器配置初始化

```
hostname Hub-2
interface FastEthernet1/0
 ip address 61.128.1.200 255.255.255.0
 no shutdown
interface FastEthernet2/0
 ip address 192.168.100.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
ip route 61.128.2.0 255.255.255.0 61.128.1.3
```

（3）Spoke-1路由器配置初始化

```
hostname Spoke-1
interface Loopback0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.1 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（4）Spoke-2路由器配置初始化

```
hostname Spoke-2
interface Loopback0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 202.100.1.10
```

（5）Spoke-3路由器配置初始化

```
hostname Spoke-3
interface Loopback0
 ip address 192.168.3.1 255.255.255.0
 no shutdown
interface FastEthernet1/0
 ip address 61.128.1.3 255.255.255.0
 no shutdown
interface FastEthernet3/0
 ip address 61.128.2.3 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.1.10
```

（6）Spoke-4路由器配置初始化

```
hostname Spoke-4
interface Loopback0
 ip address 192.168.4.1 255.255.255.0
 no shutdown
interface FastEthernet3/0
 ip address 61.128.2.4 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 61.128.2.3
```

（7）ASA防火墙配置初始化

```
hostname ASA
interface GigabitEthernet0/0
 nameif Outside
 security-level 0
 ip address 202.100.1.10 255.255.255.0 
 no shutdown
interface GigabitEthernet0/1
 nameif Inside
 security-level 100
 ip address 61.128.1.10 255.255.255.0
 no shutdown
access-list out extended permit icmp any any
access-group out in interface Outside
route Inside 61.128.2.0 255.255.255.0 61.128.1.3
```

步骤2：配置mGRE和NHRP

（1）Hub-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.100 255.255.255.0
 tunnel source 61.128.1.100  
 tunnel mode gre multipoint  
 tunnel key 12345  
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic 
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.200
 ip nhrp redirect
show run interface tunnel 0  
```

（2）Hub-2路由器配配置

```
interface Tunnel0
 ip address 172.16.1.200 255.255.255.0
 tunnel source 61.128.1.200  
 tunnel mode gre multipoint  
 tunnel key 12345  
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic 
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map multicast 61.128.1.100
 ip nhrp redirect
show run interface tunnel 0  
```

（3）Spoke-1路由器配配置

```
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 tunnel source 202.100.1.1
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100
 ip nhrp nhs 172.16.1.200
 ip nhrp shortcut
show run interface tunnel 0  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-1通过tunnel口已经可以成功访问到HUB-1）
```

（4）Spoke-2路由器配配置

```
第一种配置 （hub-1为静态Global地址的情况）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100 
 ip nhrp nhs 172.16.1.200
 ip nhrp shortcut
show run interface tunnel 0  
ping 172.16.1.100 

第二种配置 （hub-1为动态Global地址的情况。模拟使用DDNS。）
ip domain lookup 
ip name-server 114.114.114.114
ip host hub-1.cloudpbc.cn 61.128.1.100
ip host hub-2.cloudpbc.cn 61.128.1.200
ping hub-1.cloudpbc.cn （测试）
ping hub-2.cloudpbc.cn （测试）
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 tunnel source 202.100.1.2
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
 ip nhrp nhs dynamic nbma hub-2.cloudpbc.cn multicast
 ip nhrp shortcut
show run interface tunnel 0  
ping 172.16.1.100

*DMVPN静态Global和动态域名解析配置命令的区别：
 ip nhrp nhs dynamic nbma hub-1.cloudpbc.cn multicast
取代传统的如下三行命令配置
 ip nhrp map 172.16.1.100 61.128.1.100 
 ip nhrp map multicast 61.128.1.100  
 ip nhrp nhs 172.16.1.100 
```

（5）Spoke-3路由器配配置

```
interface Tunnel0
 ip address 172.16.1.3 255.255.255.0
 tunnel source 61.128.1.3
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10
 ip nhrp authentication cisco
 ip nhrp map 172.16.1.100 61.128.1.100
 ip nhrp map 172.16.1.200 61.128.1.200
 ip nhrp map multicast 61.128.1.100
 ip nhrp map multicast 61.128.1.200
 ip nhrp nhs 172.16.1.100
 ip nhrp nhs 172.16.1.200
 ip nhrp shortcut
 ip nhrp redirect  （必须加）
interface Tunnel1
 ip address 172.16.2.3 255.255.255.0
 tunnel source 61.128.2.3
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10  
 ip nhrp authentication cisco  
 ip nhrp map multicast dynamic
 ip nhrp redirect
show run interface tunnel 0  （查看已经完成的配置）
show run interface tunnel 1  （查看已经完成的配置）
ping 172.16.1.100   （Spoke-3通过tunnel口已经可以成功访问到HUB-1）
```

（5）Spoke-4路由器配配置

```
interface Tunnel1
 ip address 172.16.2.4 255.255.255.0
 tunnel source 61.128.2.4
 tunnel mode gre multipoint 
 tunnel key 12345
 ip nhrp network-id 10  
 ip nhrp authentication cisco
 ip nhrp map 172.16.2.3 61.128.2.3
 ip nhrp map multicast 61.128.2.3
 ip nhrp nhs 172.16.2.3
 ip nhrp shortcut
```

步骤3：配置动态路由协议

（1）Hub-1和Hub-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.100.0 0.0.0.255
```

（2）Spoke-1路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.1.0 0.0.0.255
```

（3）Spoke-2路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

（4）Spoke-3路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.1.0 0.0.0.255
 network 172.16.2.0 0.0.0.255
 network 192.168.3.0 0.0.0.255
```

（4）Spoke-4路由器配配置

```
router eigrp 100
 no auto-summary 
 network 172.16.2.0 0.0.0.255
 network 192.168.4.0 0.0.0.255
```

步骤4：调整和测试EIGRP

在Hub-1和Hub2路由器上调整EIGRP

```
interface Tunnel0
 ip summary-address eigrp 100 192.168.0.0 255.255.0.0
第三阶段的DMVPN不再需要关闭水平分割（no ip split-horizon eigrp 100），也不在需要关闭关闭EIGRP的next-hop-self特性来优化路由（no ip next-hop-self eigrp 100）。只需要中心给所有的分支站点发送一条汇总路由。
```


步骤5：配置IPSec VPN

（1）在ASA防火墙上放行isakmp UDP 500和esp的流量

```
object-group network Outside-DMVPN-Address
 network-object host 202.100.1.1
 network-object host 202.100.1.2
object-group network Inside-DMVPN-Address
 network-object host 61.128.1.100
 network-object host 61.128.1.200
 network-object host 61.128.1.3
 network-object host 61.128.2.3
 network-object host 61.128.2.4
object-group service DMVPN-IPSec
 service-object udp destination eq isakmp 
 service-object esp 
access-list out extended permit object-group DMVPN-IPSec object-group Outside-DMVPN-Address object-group Inside-DMVPN-Address 
```

（2）在HUB-1，HUB-2，Spoke1，Spoke2，Spoke3和Spoke4上都配置相同的配置

```
crypto isakmp policy 10
 authentication pre-share 
crypto isakmp key cisco address 0.0.0.0 0.0.0.0
crypto ipsec transform-set cisco esp-des esp-md5-hmac 
 mode transport 
crypto ipsec profile dmvpn-profile
 set transform-set cisco
```

（3）将ipsec profile配置调用在Tunnel接口上

在HUB-1，HUB-2，Spoke1，Spoke2和Spoke3上都配置相同的配置

```
interface Tunnel0
 ip mtu 1400
 tunnel protection ipsec profile dmvpn-profile
```

在Spoke3和Spoke4上配置相同的配置

```
interface Tunnel1
 ip mtu 1400
 tunnel protection ipsec profile dmvpn-profile
```

步骤6：测试并查看DMVPN的状态

（1）测试NHRP

```
show ip nhrp  （在HUB，Spoke中查看NHRP的注册信息）
确认了NHRP在中心站点和分支站点已经正常工作以后，在分支站点一（Spoke1）通过ping测试访问分支站点四（Spoke4），主要目的是检查NHRP的动态解析功能。
Spoke-1#ping 172.16.2.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.2, timeout is 2 seconds:
!.!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 8/27/80 ms
```

（2）在分支站点一（Spoke1）上查看EIGRP学习的路由

```
Spoke-1#show ip route eigrp 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 202.100.1.10 to network 0.0.0.0

D     192.168.0.0/16 [90/26882560] via 172.16.1.200, 00:00:19, Tunnel0
                     [90/26882560] via 172.16.1.100, 00:00:19, Tunnel0
```

（3）在Hub1上查看EIGRP学习的路由

```
Hub-1#show ip route eigrp 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 61.128.1.10 to network 0.0.0.0

      172.16.0.0/16 is variably subnetted, 3 subnets, 2 masks
D        172.16.2.0/24 [90/28160000] via 172.16.1.3, 00:04:17, Tunnel0
D     192.168.0.0/16 is a summary, 00:04:28, Null0
D     192.168.1.0/24 [90/27008000] via 172.16.1.1, 00:04:24, Tunnel0
D     192.168.2.0/24 [90/27008000] via 172.16.1.2, 00:04:24, Tunnel0
D     192.168.3.0/24 [90/27008000] via 172.16.1.3, 00:04:21, Tunnel0
D     192.168.4.0/24 [90/28288000] via 172.16.1.3, 00:04:14, Tunnel0
```

（4）在分支站点四（Spoke4）上查看EIGRP学习的路由

```
Spoke-4#show ip route eigrp 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 61.128.2.3 to network 0.0.0.0

      172.16.0.0/16 is variably subnetted, 3 subnets, 2 masks
D        172.16.1.0/24 [90/28160000] via 172.16.2.3, 00:06:37, Tunnel1
D     192.168.0.0/16 [90/28162560] via 172.16.2.3, 00:06:37, Tunnel1
D     192.168.3.0/24 [90/27008000] via 172.16.2.3, 00:06:37, Tunnel1
```

（3）在分支站点一（Spoke1）上查看IPSec状态

```
1）在Spoke1上使用ping触发分支站点间的流量
Spoke-1#ping 192.168.4.1 source 192.168.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.4.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 20/33/160 ms
2）查看IPSec的状态
Spoke-1#show crypto session
Interface: Tunnel0
Session status: UP-ACTIVE     
Peer: 61.128.2.4 port 500 
  IKEv1 SA: local 202.100.1.1/500 remote 61.128.2.4/500 Active 
  IKEv1 SA: local 202.100.1.1/500 remote 61.128.2.4/500 Active 
  IPSEC FLOW: permit 47 host 202.100.1.1 host 61.128.2.4 
        Active SAs: 2, origin: crypto map
        
Spoke-1#show crypto ipsec sa
   local  ident (addr/mask/prot/port): (202.100.1.1/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (61.128.2.4/255.255.255.255/47/0)
   current_peer 61.128.2.4 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 94, #pkts encrypt: 94, #pkts digest: 94
    #pkts decaps: 94, #pkts decrypt: 94, #pkts verify: 94
```

