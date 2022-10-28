## DMVPN







![DMVPN Phase2拓扑图](https://cdn.jsdelivr.net/gh/wang-xiaokai/images/202210282031650.png "DMVPN Phase2拓扑图")

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

```

