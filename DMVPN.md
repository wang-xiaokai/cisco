## DMVPN











<img src="https://cdn.jsdelivr.net/gh/wang-xiaokai/images/202210282125194.png" alt="DMVPN Phase2拓扑图" title="DMVPN Phase2拓扑图" width="70%" height="50%" />

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
interface FastEthernet2/0
 ip address 192.168.100.2 255.255.255.0
 no shutdown
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

（5）ASA防火墙配置初始化

```
hostname ASA
interface GigabitEthernet0/0
 nameif Outside
 security-level 0
 ip address 202.100.1.10 255.255.255.0 
interface GigabitEthernet0/1
 nameif Inside
 security-level 100
 ip address 61.128.1.10 255.255.255.0
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

