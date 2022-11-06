## GETVPN



#### 1.GETVPN组播实验

<img src="https://cdn.jsdelivr.net/gh/wang-xiaokai/images/202211051842522.png" alt="GEVPN拓扑图" title="GEVPN拓扑图" width="70%" height="30%" />

##### 1.1 GETVPN组播更新配置实例

步骤1：基本配置初始化

（1）KS-1路由器配置初始化

```
hostname KS-1
interface Loopback0
 ip address 172.16.100.1 255.255.255.0
 ip ospf network point-to-point
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.1 255.255.255.0
 no shutdown
router ospf 1
 network 172.16.100.0 0.0.0.255 area 0
 network 202.100.1.0 0.0.0.255 area 0
```

（2）KS-2路由器配置初始化

```
hostname KS-2
interface FastEthernet0/0
 ip address 202.100.1.2 255.255.255.0
 no shutdown
router ospf 1
 network 202.100.1.0 0.0.0.255 area 0
```

（3）ASA配置初始化

```
interface GigabitEthernet0/0
 ip address 202.100.1.10 255.255.255.0 
 nameif Inside
 security-level 100
 no shutdown          
interface GigabitEthernet0/1
 ip address 202.100.2.10 255.255.255.0
 nameif Outside
 security-level 0
 no shutdown
router ospf 1
 network 202.100.1.0 255.255.255.0 area 0
 network 202.100.2.0 255.255.255.0 area 0
access-list out extended permit icmp any any
access-list out extended permit udp any any eq 848
access-group out in interface Outside
```

（4）GM-1路由器配置初始化

```
hostname GM-1
interface Loopback0
 ip address 172.16.1.1 255.255.255.0
 ip ospf network point-to-point
 no shutdown
interface FastEthernet1/0
 ip address 202.100.2.1 255.255.255.0
 no shutdown
router ospf 1
 network 172.16.1.0 0.0.0.255 area 0
 network 202.100.2.0 0.0.0.255 area 0
```

（5）GM-2路由器配置初始化

```
hostname GM-2
interface Loopback0
 ip address 172.16.2.1 255.255.255.0
 ip ospf network point-to-point
 no shutdown
interface FastEthernet1/0
 ip address 202.100.2.2 255.255.255.0
 no shutdown
router ospf 1
 network 172.16.2.0 0.0.0.255 area 0
 network 202.100.2.0 0.0.0.255 area 0
```

（6）在GM-2和GM-2上测试全网可路由

```
GM-1#ping 172.16.2.1 source 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.2.1, timeout is 2 seconds:
Packet sent with a source address of 172.16.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/13/28 ms
GM-1#ping 172.16.100.1 source 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.100.1, timeout is 2 seconds:
Packet sent with a source address of 172.16.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 20/20/20 ms
```

步骤2：组播配置

（1）KS-1路由器配置

```
ip multicast-routing
ip pim rp-address 202.100.1.2
interface FastEthernet0/0
 ip pim sparse-mode （在OSPF宣告的接口上都需要配置）
```

（2）KS-2路由器配置

```
ip multicast-routing
ip pim rp-address 202.100.1.2
interface FastEthernet0/0
 ip pim sparse-mode 
```

（3）GM-1路由器配置

```
ip multicast-routing
ip pim rp-address 202.100.1.2
interface FastEthernet1/0
 ip pim sparse-mode 
```

（4）GM-2路由器配置

```
ip multicast-routing
ip pim rp-address 202.100.1.2
interface FastEthernet1/0
 ip pim sparse-mode
```

（5）ASA配置

```
multicast-routing  （当enable组播时，ASA的接口默认sparse-mode）
警告：WARNING: Interfaces part of explicit zone will not participate in multicast-routing
pim rp-address 202.100.1.2

查看pim邻居关系
ciscoasa(config)# show pim neighbor 
Neighbor Address  Interface          Uptime    Expires DR pri Bidir
202.100.1.1       Inside             00:10:29  00:01:37 1     
202.100.1.2       Inside             00:10:29  00:01:36 1     
202.100.2.1       Outside            00:10:29  00:01:33 1     
202.100.2.2       Outside            00:10:29  00:01:36 1
```

（6）SW-1和SW-2的配置

```
no ip igmp snooping  （模拟交换机bug,真实交换机不需要配置）
```

步骤3：产生和查看密钥

KS1产生RSA密钥

```
ip domain name cloudpbc.cn
crypto key generate rsa label getvpnkey modulus 1024 exportable

KS-1#show crypto key mypubkey rsa getvpnkey
% Key pair was generated at: 03:56:15 UTC Nov 5 2022
Key name: getvpnkey
Key type: RSA KEYS
 Storage Device: not specified
 Usage: General Purpose Key
 Key is exportable.
 Key Data:
  30819F30 0D06092A 864886F7 0D010101 05000381 8D003081 89028181 00A23259 
  E7FA9604 7C965170 D96DED7B CAB4C1FD FB8AAC7E 67C12368 9D1E1ABE 98D53FA1 
  9C5D8642 719C24CB 30944426 72B24440 852FE845 40FC8768 B2AE8113 9760C7DF 
  E456710B D183178D F3DC80A6 25643D9D 961D0C55 C3490C6E 07A046D8 BACD3390 
  6480DF30 FA23F8EE CEBB920F FA261086 B6719A5F C18631C6 1C5FCE8D 03020301 0001
```

步骤4：配置KS1的策略

KS需要配置的策略：（1）IKE Phase1 Policy（IKE第一阶段策略）2. Rekey SA Policy（密钥更新安全关联策略）3.IPSec SA Policy（IPSec安全关联策略）

（1）配置ISAKMP第一阶段策略

```
crypto isakmp policy 10
 authentication pre-share
crypto isakmp key cisco address 202.100.1.2
crypto isakmp key cisco address 202.100.2.1
crypto isakmp key cisco address 202.100.2.2
```

（2）配置感兴趣流

```
ip access-list extended GETVPN-Traffic （配置感兴趣流，后期如果需要更改必须在KS-1和KS-2上配置相同）
 permit ip 172.16.0.0 0.0.255.255 172.16.0.0 0.0.255.255
ip access-list extended Multicast （定义组播流量，239.0.1.2为任意一个组播地址）
 permit udp host 202.100.1.1 eq 848 host 239.0.1.2 eq 848
```

（3）配置IPSec Profile

```
crypto ipsec transform-set getvpn-set esp-3des esp-md5-hmac
crypto ipsec profile getvpn-profile
 set transform-set getvpn-set
```

（4）配置KS1为密钥服务器

```
KS-1(config)#crypto gdoi group mygroup
KS-1(config-gdoi-group)#identity number 66666
KS-1(config-gdoi-group)#server local
KS-1(gdoi-local-server)#address ipv4 202.100.1.1
```

（5）配置密钥更新（组播）

```
KS-1(gdoi-local-server)#rekey authentication mypubkey rsa getvpnkey
KS-1(gdoi-local-server)#rekey algorithm aes 256 （默认加密算法为3des，HASH算法为SHA-1。此配置为可选，配置密钥更新信息的加密算法为aes）
KS-1(gdoi-local-server)#rekey address ipv4 Multicast
```

（6）配置IPSec 安全关联

```
KS-1(gdoi-local-server)#sa ipsec 1
KS-1(gdoi-sa-ipsec)#match address ipv4 GETVPN-Traffic
KS-1(gdoi-sa-ipsec)#profile getvpn-profile
KS-1(gdoi-sa-ipsec)#replay time window-size 3 （配置为选项，启用基于时间的防重放，窗口大小为3秒）
```

步骤4：配置GM-1和GM-2的策略

（1）配置GM-X的IKE Phase1 Policy（IKE第一阶段策略）

```
crypto isakmp policy 10
 authentication pre-share
crypto isakmp key 0 cisco address 202.100.1.1
crypto isakmp key 0 cisco address 202.100.1.2
crypto gdoi group mygroup
 identity number 66666
 server address ipv4 202.100.1.1
 server address ipv4 202.100.1.2
crypto map cisco 10 gdoi
 set group mygroup
interface FastEthernet1/0
 crypto map cisco
```

（1）在KS-1上查看加解密的状态

```
KS-1#show crypto gdoi group mygroup
    Group Name               : mygroup (Multicast)
    Group Identity           : 66666
    Group Members            : 2
    IPSec SA Direction       : Both
    Group Rekey Lifetime     : 86400 secs
    Group Rekey
        Remaining Lifetime   : 84098 secs
    Rekey Retransmit Period  : 10 secs
    Rekey Retransmit Attempts: 2
    Group Retransmit
        Remaining Lifetime   : 0 secs

      IPSec SA Number        : 1
      IPSec SA Rekey Lifetime: 3600 secs
      Profile Name           : getvpn-profile
      Replay method          : Time Based
      Replay Window Size     : 3
      SA Rekey
         Remaining Lifetime  : 1299 secs
      ACL Configured         : access-list GETVPN-Traffic

     Group Server list       : Local
```

（3）在GM-1上查看加解密的状态

```
命令1:
GM-1#show crypto engine connections active 
Crypto Engine Connections

   ID  Type    Algorithm           Encrypt  Decrypt LastSeqN IP-Address
    1  IPsec   3DES+MD5                  0        0        0 172.16.0.0
    2  IPsec   3DES+MD5                  0        0        0 172.16.0.0
 1001  IKE     SHA+DES                   0        0        0 202.100.2.1
 1002  IKE     SHA+AES256                0        0        0 
解释：
ID:1001为IKE Phase1 SA（IKE第一阶段SA），使用的算法为DES，HASH为SHA。
ID:1002为Rekey SA（密钥更新安全关联策略），使用的算法为AES256，HASH为SHA。
ID:1和ID:2为IPSec SA（IPSec安全关联）

命令2:
GM-1#show crypto gdoi
GROUP INFORMATION

    Group Name               : mygroup （GDOI组名）
    Group Identity           : 66666   （GDOI ID名）
    Rekeys received          : 0
    IPSec SA Direction       : Both

     Group Server list       : 202.100.1.1  （有两个KS服务器）
                               202.100.1.2
                               
    Group member             : 202.100.2.1      vrf: None  （组成员自己的IP地址）
       Registration status   : Registered  （已经注册状态）
       Registered with       : 202.100.1.1 （注册在KS1上）
       Re-registers in       : 2579 sec
       Succeeded registration: 1
       Attempted registration: 5
       Last rekey from       : 0.0.0.0
       Last rekey seq num    : 0
       Multicast rekey rcvd  : 0
       allowable rekey cipher: any
       allowable rekey hash  : any
       allowable transformtag: any ESP

    Rekeys cumulative
       Total received        : 0
       After latest register : 0
       Rekey Received        : never

 ACL Downloaded From KS 202.100.1.1:  （从KS1中心获取到的感兴趣流）
   access-list  permit ip 172.16.0.0 0.0.255.255 172.16.0.0 0.0.255.255

KEK POLICY:
    Rekey Transport Type     : Multicast （组播更新）
    Lifetime (secs)          : 86399
    Encrypt Algorithm        : AES  （加密算法）
    Key Size                 : 256     
    Sig Hash Algorithm       : HMAC_AUTH_SHA （完整性校验算法）
    Sig Key Length (bits)    : 1024    （长度）

TEK POLICY for the current KS-Policy ACEs Downloaded:
  FastEthernet1/0:
    IPsec SA:
        spi: 0xB8841F0F(3095666447)
        transform: esp-3des esp-md5-hmac 
        sa timing:remaining key lifetime (sec): (2753)
        Anti-Replay(Time Based) : 3 sec interval
命令3:
GM-1# show crypto gdoi ipsec sa 
SA created for group mygroup:
  FastEthernet1/0:
    protocol = ip
      local ident  = 172.16.0.0/16, port = 0
      remote ident = 172.16.0.0/16, port = 0
      direction: Both, replay(method/window): Time/3 sec
      
命令4:触发感兴趣流查看加解密状态
GM-1#ping 172.16.2.1 source 172.16.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 172.16.2.1, timeout is 2 seconds:
Packet sent with a source address of 172.16.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 8/10/56 ms
GM-1#show crypto engine connections active 
Crypto Engine Connections

   ID  Type    Algorithm           Encrypt  Decrypt LastSeqN IP-Address
    1  IPsec   3DES+MD5                  0      100        0 172.16.0.0
    2  IPsec   3DES+MD5                100        0        0 172.16.0.0
 1001  IKE     SHA+DES                   0        0        0 202.100.2.1
 1002  IKE     SHA+AES256                0        0        0 
```



##### 1.2 GETVPN组成员访问控制列表配置

​    首先需要在GM-1上通过ping测试访问KS-1身后的网络。测试是失败的。原因是源172.16.1.1目的为172.16.100.1的流量满足GETVPN的感兴趣流，因此GM-1会对这个流量进行加密。但是密钥服务器KS-1不存在IPSec SA，所以不能对此流量进行解密。

```
GM-1#ping 172.16.100.1 source 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.100.1, timeout is 2 seconds:
Packet sent with a source address of 172.16.1.1 
.....
Success rate is 0 percent (0/5)
```

​    解决方案是在组成员GM-1上配置组成员访问控制列表，旁路掉从172.16.1.0/24到172.16.100.0/24的流量。

```
ip access-list extended bypass 
 deny ip 172.16.1.0 0.0.0.255 172.16.100.0 0.0.0.255
crypto map cisco 10 gdoi 
 match address bypass 
```

​    本地列表将优先于来自KS推送的列表。

```
GM-1#show crypto gdoi gm acl
Group Name: mygroup
 ACL Downloaded From KS 202.100.1.1:
   access-list  permit ip 172.16.0.0 0.0.255.255 172.16.0.0 0.0.255.255
 ACL Configured Locally: 
  Map Name: cisco
   access-list bypass deny ip 172.16.1.0 0.0.0.255 172.16.100.0 0.0.0.255
```



##### 1.3 GETVPN 双KS之间的协作

步骤1：在KS-1上导出RSA密钥

```
KS-1(config)#crypto key export rsa getvpnkey pem terminal 3des cisco123
导出名字为getvpnkey的RSA密钥对，使用3DES加密算法来加密导出后的私钥，加密密码为cisco123
% Key name: getvpnkey
   Usage: General Purpose Key
   Key data:
-----BEGIN PUBLIC KEY-----  （公钥）
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCiMlnn+pYEfJZRcNlt7XvKtMH9
+4qsfmfBI2idHhq+mNU/oZxdhkJxnCTLMJREJnKyRECFL+hFQPyHaLKugROXYMff
5FZxC9GDF43z3ICmJWQ9nZYdDFXDSQxuB6BG2LrNM5BkgN8w+iP47s67kg/6JhCG
tnGaX8GGMcYcX86NAwIDAQAB
-----END PUBLIC KEY-----
-----BEGIN RSA PRIVATE KEY----- （使用3DES算法加密后的私钥，密码为cisco123）
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,E1818CFE16C197D6

hF4cLbYQRLScoHrzwEF7cFcf36kID9NMGcLhD3IatIEHdJHcmILYNJF9HF9/no9+
4rcwAO8OEpLSZ7Hvj4RRxU3AGiJ8eeoFu54Flpyp7YFE2ZZWwTuUJWCaFVaCOv7E
N3aNrJi08dkgRr7jNfP0oBQouvidKXjZQuIVvoVpXqrwWpwOEirwvfNUB+833zAU
O9Mih/hHEFjyeq7Oa4kkri6QquLoqd3tJsrDTFUZ78Jc4/LC9dXqaXD/EgXLgK8w
owBQgIvsKieog3rW5KsZ+DVr9Qh4ljQW1rfLSc/kGrcfIyJNHR5UHb+0Gi3+PVdG
tmjjOG72o25hrFACfb3NDdVsoEotlJ7WNaQE3SahbP7Hy4f/Bm8PAYEaVtg4CRmL
iDt4ho8UdNdQUkXoFmXOXEBT/eTTl6U6Rad7mK0K9M57/xVt+RajTiKCLzWwrYa/
bwShL45VI9aI9Y3hv33um5KSHboJZ2aJBH3e9L3WJlvQT+WS/91lse/59MoOzc1o
TT/uOHgZV286qqYdorKYO5kaWBWVpAwwKu1DOdKnCoavheOYeCZD1atHrq2PO6we
/rx5Xilrbe/IRlYKW2HQM+v1PFUbudwKkk8Rhuten7rkFltTkePmE7fctLwijLMY
+E4rn57Z+o9wdr1QwVmhK2bdKV7gms58Hjl82ZZdqB8LluuiRs1tSoBTazEfu6p+
3cJEcffIEON0ws5B3/csep8uWfd32F/CQuK3GolKNbFlxJLFh184mtONDVNl8Dg8
mHy1IM0JMQmmwvBpoTYbK1v/EJp9Gtg9vxZhcH0Xl5E=
-----END RSA PRIVATE KEY-----
```

建议：导出RSA后修改RSA为不可导出。但此操作不可逆。

```
crypto key move rsa getvpnkey non-exportable
```

步骤2：在KS-2上导入RSA密钥

```
KS-2(config)#crypto key import rsa getvpnkey terminal cisco123

% Enter PEM-formatted public General Purpose key or certificate.
% End with a blank line or "quit" on a line by itself.
拷贝KS-1的公钥在下面
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCiMlnn+pYEfJZRcNlt7XvKtMH9
+4qsfmfBI2idHhq+mNU/oZxdhkJxnCTLMJREJnKyRECFL+hFQPyHaLKugROXYMff
5FZxC9GDF43z3ICmJWQ9nZYdDFXDSQxuB6BG2LrNM5BkgN8w+iP47s67kg/6JhCG
tnGaX8GGMcYcX86NAwIDAQAB
-----END PUBLIC KEY----- 在这里第一次回车
在这里第二次回车
% Enter PEM-formatted encrypted private General Purpose key.
% End with "quit" on a line by itself.
拷贝KS-1的私钥在下面
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,E1818CFE16C197D6

hF4cLbYQRLScoHrzwEF7cFcf36kID9NMGcLhD3IatIEHdJHcmILYNJF9HF9/no9+
4rcwAO8OEpLSZ7Hvj4RRxU3AGiJ8eeoFu54Flpyp7YFE2ZZWwTuUJWCaFVaCOv7E
N3aNrJi08dkgRr7jNfP0oBQouvidKXjZQuIVvoVpXqrwWpwOEirwvfNUB+833zAU
O9Mih/hHEFjyeq7Oa4kkri6QquLoqd3tJsrDTFUZ78Jc4/LC9dXqaXD/EgXLgK8w
owBQgIvsKieog3rW5KsZ+DVr9Qh4ljQW1rfLSc/kGrcfIyJNHR5UHb+0Gi3+PVdG
tmjjOG72o25hrFACfb3NDdVsoEotlJ7WNaQE3SahbP7Hy4f/Bm8PAYEaVtg4CRmL
iDt4ho8UdNdQUkXoFmXOXEBT/eTTl6U6Rad7mK0K9M57/xVt+RajTiKCLzWwrYa/
bwShL45VI9aI9Y3hv33um5KSHboJZ2aJBH3e9L3WJlvQT+WS/91lse/59MoOzc1o
TT/uOHgZV286qqYdorKYO5kaWBWVpAwwKu1DOdKnCoavheOYeCZD1atHrq2PO6we
/rx5Xilrbe/IRlYKW2HQM+v1PFUbudwKkk8Rhuten7rkFltTkePmE7fctLwijLMY
+E4rn57Z+o9wdr1QwVmhK2bdKV7gms58Hjl82ZZdqB8LluuiRs1tSoBTazEfu6p+
3cJEcffIEON0ws5B3/csep8uWfd32F/CQuK3GolKNbFlxJLFh184mtONDVNl8Dg8
mHy1IM0JMQmmwvBpoTYbK1v/EJp9Gtg9vxZhcH0Xl5E=
-----END RSA PRIVATE KEY----- 在这里第一次回车
在这里第二次回车
quit （输入quit退出）
% Key pair import succeeded.

KS-2#show crypto key mypubkey rsa getvpnkey
% Key pair was generated at: 11:12:21 UTC Nov 5 2022
Key name: getvpnkey
Key type: RSA KEYS
 Storage Device: not specified
 Usage: General Purpose Key
 Key is not exportable.
 Key Data:
  30819F30 0D06092A 864886F7 0D010101 05000381 8D003081 89028181 00A23259 
  E7FA9604 7C965170 D96DED7B CAB4C1FD FB8AAC7E 67C12368 9D1E1ABE 98D53FA1 
  9C5D8642 719C24CB 30944426 72B24440 852FE845 40FC8768 B2AE8113 9760C7DF 
  E456710B D183178D F3DC80A6 25643D9D 961D0C55 C3490C6E 07A046D8 BACD3390 
  6480DF30 FA23F8EE CEBB920F FA261086 B6719A5F C18631C6 1C5FCE8D 03020301 0001
```

步骤3：配置KS-2的策略

KS需要配置的策略：（1）IKE Phase1 Policy（IKE第一阶段策略）2. Rekey SA Policy（密钥更新安全关联策略）3.IPSec SA Policy（IPSec安全关联策略）

（1）配置ISAKMP第一阶段策略

```
crypto isakmp policy 10
 authentication pre-share
crypto isakmp key cisco address 202.100.1.1
crypto isakmp key cisco address 202.100.2.1
crypto isakmp key cisco address 202.100.2.2
```

（2）配置感兴趣流

```
ip access-list extended GETVPN-Traffic （配置感兴趣流，后期如果需要更改必须在KS-1和KS-2上配置相同）
 permit ip 172.16.0.0 0.0.255.255 172.16.0.0 0.0.255.255
ip access-list extended Multicast （定义组播流量，239.0.1.2为任意一个组播地址）
 permit udp host 202.100.1.2 eq 848 host 239.0.1.2 eq 848
```

（3）配置IPSec Profile

```
crypto ipsec transform-set getvpn-set esp-3des esp-md5-hmac
crypto ipsec profile getvpn-profile
 set transform-set getvpn-set
```

（4）配置KS-2为密钥服务器

```
KS-2(config)#crypto gdoi group mygroup
KS-2(config-gdoi-group)#identity number 66666
KS-2(config-gdoi-group)#server local
KS-2(gdoi-local-server)#address ipv4 202.100.1.2
```

（5）配置密钥更新（组播）

```
KS-2(gdoi-local-server)#rekey authentication mypubkey rsa getvpnkey
KS-2(gdoi-local-server)#rekey algorithm aes 256 （默认加密算法为3des，HASH算法为SHA-1。此配置为可选，配置密钥更新信息的加密算法为aes）
KS-2(gdoi-local-server)#rekey address ipv4 Multicast
```

（6）配置IPSec 安全关联

```
KS-2(gdoi-local-server)#sa ipsec 1
KS-2(gdoi-sa-ipsec)#match address ipv4 GETVPN-Traffic
KS-2(gdoi-sa-ipsec)#profile getvpn-profile
KS-2(gdoi-sa-ipsec)#replay time window-size 3 （配置为选项，启用基于时间的防重放，窗口大小为3秒）
```

步骤4：配置GETVPN的高可用性

（1）在首要密钥服务器KS-1上配置冗余密钥服务器

```
KS-1(config)#crypto gdoi group mygroup
KS-1(config-gdoi-group)#server local
KS-1(gdoi-local-server)#redundancy 
KS-1(gdoi-coop-ks-config)#peer address ipv4 202.100.1.2
KS-1(gdoi-coop-ks-config)#local priority 100  （默认为1，越高越能成为Primary）
```

（2）在次要密钥服务器KS-2上配置冗余密钥服务器

```
KS-2(config)#crypto gdoi group mygroup
KS-2(config-gdoi-group)#server local
KS-2(gdoi-local-server)#redundancy 
KS-2(gdoi-coop-ks-config)#peer address ipv4 202.100.1.1 
KS-2(gdoi-coop-ks-config)#local priority 75
```

（4）查看协作密钥服务器的状态

```
KS-2#show crypto gdoi ks coop 
Crypto Gdoi Group Name :mygroup 
        Group handle: 2147483650, Local Key Server handle: 2147483650

        Local Address: 202.100.1.2 （本地KS地址）
        Local Priority: 75       （本地KS优先级）
        Local KS Role: Secondary , Local KS Status: Alive   （本地KS角色和当前的状态）
        Secondary Timers: 
                Sec Primary Periodic Time: 30 
                Remaining Time: 25, Retries: 0
                Invalid ANN PST recvd: 0
                New GM Temporary Blocking Enforced?: No
                Antireplay Sequence Number: 1

        Peer Sessions:
        Session 1:
                Server handle: 2147483651
                Peer Address: 202.100.1.1  （对端KS地址）
                Peer Priority: 100         （对端KS优先级）
                Peer KS Role: Primary   , Peer KS Status: Alive （本地KS角色和当前的状态）
                Antireplay Sequence Number: 53

                IKE status: Established
                Counters:
                    Ann msgs sent: 0
                    Ann msgs sent with reply request: 1
                    Ann msgs recv: 41 
                    Ann msgs recv with reply request: 0
                    Packet sent drops: 0 
                    Packet Recv drops: 0 
                    Total bytes sent: 56 
                    Total bytes recv: 22755
```



#### 2.GETVPN单播更新在VRF环境中的应用



<img src="https://cdn.jsdelivr.net/gh/wang-xiaokai/images/202211061204863.png" alt="GEVPN VRF环境拓扑图" title="GEVPN VRF环境拓扑图" width="100%" height="80%" />



##### 2.1 GETVPN单播配置实例

步骤1：基本配置初始化

（1）KS-1路由器配置初始化

```
hostname KS-1
interface Loopback0
 ip address 172.16.100.1 255.255.255.0
 ip ospf network point-to-point
 no shutdown
interface FastEthernet0/0
 ip address 202.100.1.1 255.255.255.0
 no shutdown
router ospf 1
 network 172.16.100.0 0.0.0.255 area 0
 network 202.100.1.0 0.0.0.255 area 0
```

（2）KS-2路由器配置初始化

```
hostname KS-2
interface FastEthernet0/0
 ip address 202.100.1.2 255.255.255.0
 no shutdown
router ospf 1
 network 202.100.1.0 0.0.0.255 area 0
```

（3）ASA配置初始化

```
hostname ASA
interface GigabitEthernet0/0
 ip address 202.100.1.10 255.255.255.0 
 nameif Inside
 security-level 100
 no shutdown          
interface GigabitEthernet0/1
 ip address 202.100.2.10 255.255.255.0
 nameif Outside
 security-level 0
 no shutdown
router ospf 1
 network 202.100.1.0 255.255.255.0 area 0
 network 202.100.2.0 255.255.255.0 area 0
access-list out extended permit icmp any any
access-list out extended permit udp any any eq 848
access-group out in interface Outside
```

（4）GM-1路由器配置初始化

```
hostname GM-1
interface Loopback0
 ip address 172.16.1.1 255.255.255.0
 ip ospf network point-to-point
 no shutdown
interface FastEthernet1/0
 ip address 202.100.2.1 255.255.255.0
 no shutdown
router ospf 1
 network 172.16.1.0 0.0.0.255 area 0
 network 202.100.2.0 0.0.0.255 area 0
```

（5）GM-2路由器配置初始化

```
hostname GM-2
interface Loopback0
 ip address 172.16.2.1 255.255.255.0
 ip ospf network point-to-point
 no shutdown
interface FastEthernet1/0
 ip address 202.100.2.2 255.255.255.0
 no shutdown
router ospf 1
 network 172.16.2.0 0.0.0.255 area 0
 network 202.100.2.0 0.0.0.255 area 0
```

（6）在GM-2和GM-2上测试全网可路由

```
GM-1#ping 172.16.2.1 source 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.2.1, timeout is 2 seconds:
Packet sent with a source address of 172.16.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/13/28 ms
GM-1#ping 172.16.100.1 source 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.100.1, timeout is 2 seconds:
Packet sent with a source address of 172.16.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 20/20/20 ms
```

步骤2：首要和次要密钥服务器同步RSA密钥

（1）KS1产生RSA密钥

```
ip domain name cloudpbc.cn
crypto key generate rsa label getvpnkey modulus 1024 exportable

KS-1#show crypto key mypubkey rsa getvpnkey
% Key pair was generated at: 16:16:14 UTC Nov 5 2022
Key name: getvpnkey
Key type: RSA KEYS
 Storage Device: not specified
 Usage: General Purpose Key
 Key is exportable.
 Key Data:
  30819F30 0D06092A 864886F7 0D010101 05000381 8D003081 89028181 008C4D36 
  B7860DCD 9892590A 9D1A0A75 EED3C995 C2051CEC 3F3F04D4 84B7FE46 63B10315 
  087309CD F7C7B271 F7623879 AA764B4D A36EB3D3 59354607 8BB81404 B02C85AF 
  D55156CC 87515943 F41CE6A4 60A3E563 D60833CA 4672D965 EE1E6D8F DA7F0CA1 
  2467FAD2 113670F6 89D66746 94BBA2BD 3F8EADDF B79D8846 BDE22603 87020301 0001
```

（2）在KS-1上导出RSA密钥

```shell
KS-1(config)#crypto key export rsa getvpnkey pem terminal 3des cisco123
导出名字为getvpnkey的RSA密钥对，使用3DES加密算法来加密导出后的私钥，加密密码为cisco123
% Key name: getvpnkey
   Usage: General Purpose Key
   Key data:
-----BEGIN PUBLIC KEY-----  （公钥）
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCMTTa3hg3NmJJZCp0aCnXu08mV
wgUc7D8/BNSEt/5GY7EDFQhzCc33x7Jx92I4eap2S02jbrPTWTVGB4u4FASwLIWv
1VFWzIdRWUP0HOakYKPlY9YIM8pGctll7h5tj9p/DKEkZ/rSETZw9onWZ0aUu6K9
P46t37ediEa94iYDhwIDAQAB
-----END PUBLIC KEY-----
-----BEGIN RSA PRIVATE KEY-----  （使用3DES算法加密后的私钥，密码为cisco123）
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,54E8AFA5C254479C

pW+6CGVTmf8ygOnpBcHB8GPiO+d0N/xzsTmE/JKYL2L0mCJegsS8QS9/4+rSgA5K
ZB+ony+a1fshBP8VSbh5eWYPYT1zKJNrdmAvDZVaPQjpzNPXqEi/cLP23/N8LzBp
qqrg63s1vIo3pkwejq/BihO2QYyYrZ4t7YYgXJdzCW2XmYfkg4ZYzyGrpYYCplTj
3eteUilWlijFzsOoO0BNwG6VWaSiDrWKVqmWhOE4uSyKJuclQNvDmIfDy50XI8z+
B7eAb1CQsSFnVCHYHdYBv75ZL9txXhDkgbldqvbsBLxwXExhd4uZ99D6lqLun+hr
Fm7ecSK0GIiGax09cWRub6ymI5zb/mp+tB/rYRb5dWlC5F/g53wzOE1JgJEOePWB
oP9G9yBUTA6+dDVjQHxuUUGVBmMCFRYfzmdxmN2ZEz6Wg4pmACGeNXj4BO/Icb1W
+PcbamE2lSJY2Ba5yh/ldNWOgGNHT10iley9V57IPzoXEilIRcMsO9SC9MjzgRDS
7F/A52jDzrmJ1UV6oKefm1H1haMmaljdpnKIfpu2lvAbMtbsw3x0lQ0sKHKLfoyC
aW/r4p2u5mC0TUWlb/RMOc7N8mSszQZ4UwtB7JAnZRfGnMqrugeNNRvYI574qd3i
sH9pLTs6CDFS+uzWP2twBNKNxcsNbHMxEfKXuqvV+oXeV2970I7f3EFtr+h4d3Zf
J8Nept2fEPp8aHAnWCetHKoHjdfN1VdZ7bUD3XcN+mV4yx6DwBy3CuBZsWkspllw
YAXdMKA8GSf5ytLDjif3HvaZ1jErStIy+32P7yUMMZZrSlI7FegGZQ==
-----END RSA PRIVATE KEY-----
```

建议：导出RSA后修改RSA为不可导出。但此操作不可逆。

```
crypto key move rsa getvpnkey non-exportable
```

（3）在KS-2上导入RSA密钥

```shell
KS-2(config)#crypto key import rsa getvpnkey terminal cisco123

% Enter PEM-formatted public General Purpose key or certificate.
% End with a blank line or "quit" on a line by itself.
拷贝KS-1的公钥在下面
-----BEGIN PUBLIC KEY-----  
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCMTTa3hg3NmJJZCp0aCnXu08mV
wgUc7D8/BNSEt/5GY7EDFQhzCc33x7Jx92I4eap2S02jbrPTWTVGB4u4FASwLIWv
1VFWzIdRWUP0HOakYKPlY9YIM8pGctll7h5tj9p/DKEkZ/rSETZw9onWZ0aUu6K9
P46t37ediEa94iYDhwIDAQAB
-----END PUBLIC KEY----- 在这里第一次回车
在这里第二次回车
% Enter PEM-formatted encrypted private General Purpose key.
% End with "quit" on a line by itself.
拷贝KS-1的私钥在下面
-----BEGIN RSA PRIVATE KEY----- 
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,54E8AFA5C254479C

pW+6CGVTmf8ygOnpBcHB8GPiO+d0N/xzsTmE/JKYL2L0mCJegsS8QS9/4+rSgA5K
ZB+ony+a1fshBP8VSbh5eWYPYT1zKJNrdmAvDZVaPQjpzNPXqEi/cLP23/N8LzBp
qqrg63s1vIo3pkwejq/BihO2QYyYrZ4t7YYgXJdzCW2XmYfkg4ZYzyGrpYYCplTj
3eteUilWlijFzsOoO0BNwG6VWaSiDrWKVqmWhOE4uSyKJuclQNvDmIfDy50XI8z+
B7eAb1CQsSFnVCHYHdYBv75ZL9txXhDkgbldqvbsBLxwXExhd4uZ99D6lqLun+hr
Fm7ecSK0GIiGax09cWRub6ymI5zb/mp+tB/rYRb5dWlC5F/g53wzOE1JgJEOePWB
oP9G9yBUTA6+dDVjQHxuUUGVBmMCFRYfzmdxmN2ZEz6Wg4pmACGeNXj4BO/Icb1W
+PcbamE2lSJY2Ba5yh/ldNWOgGNHT10iley9V57IPzoXEilIRcMsO9SC9MjzgRDS
7F/A52jDzrmJ1UV6oKefm1H1haMmaljdpnKIfpu2lvAbMtbsw3x0lQ0sKHKLfoyC
aW/r4p2u5mC0TUWlb/RMOc7N8mSszQZ4UwtB7JAnZRfGnMqrugeNNRvYI574qd3i
sH9pLTs6CDFS+uzWP2twBNKNxcsNbHMxEfKXuqvV+oXeV2970I7f3EFtr+h4d3Zf
J8Nept2fEPp8aHAnWCetHKoHjdfN1VdZ7bUD3XcN+mV4yx6DwBy3CuBZsWkspllw
YAXdMKA8GSf5ytLDjif3HvaZ1jErStIy+32P7yUMMZZrSlI7FegGZQ==
-----END RSA PRIVATE KEY----- 在这里第一次回车
在这里第二次回车
quit （输入quit退出）
% Key pair import succeeded.

KS-2#show crypto key mypubkey rsa getvpnkey
% Key pair was generated at: 16:22:26 UTC Nov 5 2022
Key name: getvpnkey
Key type: RSA KEYS
 Storage Device: not specified
 Usage: General Purpose Key
 Key is not exportable.
 Key Data:
  30819F30 0D06092A 864886F7 0D010101 05000381 8D003081 89028181 008C4D36 
  B7860DCD 9892590A 9D1A0A75 EED3C995 C2051CEC 3F3F04D4 84B7FE46 63B10315 
  087309CD F7C7B271 F7623879 AA764B4D A36EB3D3 59354607 8BB81404 B02C85AF 
  D55156CC 87515943 F41CE6A4 60A3E563 D60833CA 4672D965 EE1E6D8F DA7F0CA1 
  2467FAD2 113670F6 89D66746 94BBA2BD 3F8EADDF B79D8846 BDE22603 87020301 0001
```

步骤3：配置首要密钥服务器KS-1上的GETVPN配置

KS需要配置的策略：（1）IKE Phase1 Policy（IKE第一阶段策略）2. Rekey SA Policy（密钥更新安全关联策略）3.IPSec SA Policy（IPSec安全关联策略）

（1）配置ISAKMP第一阶段策略

```
crypto isakmp policy 10
 authentication pre-share
crypto isakmp key cisco address 202.100.1.2
crypto isakmp key cisco address 202.100.2.1
crypto isakmp key cisco address 202.100.2.2
```

（2）配置感兴趣流

```
ip access-list extended GETVPN-Traffic （配置感兴趣流，后期如果需要更改必须在KS-1和KS-2上配置相同）
 permit ip 172.16.0.0 0.0.255.255 172.16.0.0 0.0.255.255
```

（3）配置IPSec Profile

```
crypto ipsec transform-set getvpn-set esp-3des esp-md5-hmac
crypto ipsec profile getvpn-profile
 set transform-set getvpn-set
```

（4）配置KS1为密钥服务器

```
KS-1(config)#crypto gdoi group mygroup
KS-1(config-gdoi-group)#identity number 66666
KS-1(config-gdoi-group)#server local
KS-1(gdoi-local-server)#address ipv4 202.100.1.1
```

（5）配置密钥更新（单播）

```
KS-1(gdoi-local-server)#rekey authentication mypubkey rsa getvpnkey
KS-1(gdoi-local-server)#rekey algorithm aes 256 （默认加密算法为3des，HASH算法为SHA-1。此配置为可选，配置密钥更新信息的加密算法为aes）
KS-1(gdoi-local-server)#rekey transport unicast （配置GETVPN使用单播传输密钥更新信息，注意默认为组播）
```

（6）配置IPSec 安全关联

```
KS-1(gdoi-local-server)#sa ipsec 1
KS-1(gdoi-sa-ipsec)#match address ipv4 GETVPN-Traffic
KS-1(gdoi-sa-ipsec)#profile getvpn-profile
KS-1(gdoi-sa-ipsec)#replay time window-size 3 （配置为选项，启用基于时间的防重放，窗口大小为3秒）
```

步骤4：配置次要要密钥服务器KS-2上的GETVPN配置

KS需要配置的策略：（1）IKE Phase1 Policy（IKE第一阶段策略）2. Rekey SA Policy（密钥更新安全关联策略）3.IPSec SA Policy（IPSec安全关联策略）

（1）配置ISAKMP第一阶段策略

```
crypto isakmp policy 10
 authentication pre-share
crypto isakmp key cisco address 202.100.1.1
crypto isakmp key cisco address 202.100.2.1
crypto isakmp key cisco address 202.100.2.2
```

（2）配置感兴趣流

```
ip access-list extended GETVPN-Traffic （配置感兴趣流，后期如果需要更改必须在KS-1和KS-2上配置相同）
 permit ip 172.16.0.0 0.0.255.255 172.16.0.0 0.0.255.255
```

（3）配置IPSec Profile

```
crypto ipsec transform-set getvpn-set esp-3des esp-md5-hmac
crypto ipsec profile getvpn-profile
 set transform-set getvpn-set
```

（4）配置KS-2为密钥服务器

```
KS-2(config)#crypto gdoi group mygroup
KS-2(config-gdoi-group)#identity number 66666
KS-2(config-gdoi-group)#server local
KS-2(gdoi-local-server)#address ipv4 202.100.1.2
```

（5）配置密钥更新（单播）

```
KS-2(gdoi-local-server)#rekey authentication mypubkey rsa getvpnkey
KS-2(gdoi-local-server)#rekey algorithm aes 256 （默认加密算法为3des，HASH算法为SHA-1。此配置为可选，配置密钥更新信息的加密算法为aes）
KS-2(gdoi-local-server)#rekey transport unicast （配置GETVPN使用单播传输密钥更新信息，注意默认为组播）
```

（6）配置IPSec 安全关联

```
KS-2(gdoi-local-server)#sa ipsec 1
KS-2(gdoi-sa-ipsec)#match address ipv4 GETVPN-Traffic
KS-2(gdoi-sa-ipsec)#profile getvpn-profile
KS-2(gdoi-sa-ipsec)#replay time window-size 3 （配置为选项，启用基于时间的防重放，窗口大小为3秒）
```

步骤5：配置GETVPN的高可用性

（1）在首要密钥服务器KS-1上配置冗余密钥服务器

```
KS-1(config)#crypto gdoi group mygroup
KS-1(config-gdoi-group)#server local
KS-1(gdoi-local-server)#redundancy 
KS-1(gdoi-coop-ks-config)#peer address ipv4 202.100.1.2
KS-1(gdoi-coop-ks-config)#local priority 100  （默认为1，越高越能成为Primary）
```

（2）在次要密钥服务器KS-2上配置冗余密钥服务器

```
KS-2(config)#crypto gdoi group mygroup
KS-2(config-gdoi-group)#server local
KS-2(gdoi-local-server)#redundancy 
KS-2(gdoi-coop-ks-config)#peer address ipv4 202.100.1.1 
KS-2(gdoi-coop-ks-config)#local priority 75
```

（4）查看协作密钥服务器的状态

```
KS-2#show crypto gdoi ks coop 
Crypto Gdoi Group Name :mygroup 
        Group handle: 2147483650, Local Key Server handle: 2147483650

        Local Address: 202.100.1.2 （本地KS地址）
        Local Priority: 75       （本地KS优先级）
        Local KS Role: Secondary , Local KS Status: Alive   （本地KS角色和当前的状态）
        Secondary Timers: 
                Sec Primary Periodic Time: 30 
                Remaining Time: 25, Retries: 0
                Invalid ANN PST recvd: 0
                New GM Temporary Blocking Enforced?: No
                Antireplay Sequence Number: 1

        Peer Sessions:
        Session 1:
                Server handle: 2147483651
                Peer Address: 202.100.1.1  （对端KS地址）
                Peer Priority: 100         （对端KS优先级）
                Peer KS Role: Primary   , Peer KS Status: Alive （本地KS角色和当前的状态）
                Antireplay Sequence Number: 53

                IKE status: Established
                Counters:
                    Ann msgs sent: 0
                    Ann msgs sent with reply request: 1
                    Ann msgs recv: 41 
                    Ann msgs recv with reply request: 0
                    Packet sent drops: 0 
                    Packet Recv drops: 0 
                    Total bytes sent: 56 
                    Total bytes recv: 22755
```


步骤6：配置GM-1和GM-2的策略

（1）配置GM-X的IKE Phase1 Policy（IKE第一阶段策略）

```shell
crypto isakmp policy 10
 authentication pre-share
crypto isakmp key 0 cisco address 202.100.1.1
crypto isakmp key 0 cisco address 202.100.1.2
crypto gdoi group mygroup
 identity number 66666
 server address ipv4 202.100.1.1
 server address ipv4 202.100.1.2
crypto map cisco 10 gdoi
 set group mygroup
interface FastEthernet1/0
 crypto map cisco
```

（1）在KS-1上查看加解密的状态

```shell
KS-1#show crypto gdoi group mygroup
    Group Name               : mygroup (Unicast)
    Group Identity           : 66666
    Group Members            : 2
    IPSec SA Direction       : Both
    Redundancy               : Configured
        Local Address        : 202.100.1.1
        Local Priority       : 100
        Local KS Status      : Alive
        Local KS Role        : Primary
    Group Rekey Lifetime     : 86400 secs
    Group Rekey
        Remaining Lifetime   : 86235 secs
    Rekey Retransmit Period  : 10 secs
    Rekey Retransmit Attempts: 2
    Group Retransmit
        Remaining Lifetime   : 0 secs

      IPSec SA Number        : 1
      IPSec SA Rekey Lifetime: 3600 secs
      Profile Name           : getvpn-profile
      Replay method          : Time Based
      Replay Window Size     : 3
      SA Rekey
         Remaining Lifetime  : 3436 secs
      ACL Configured         : access-list GETVPN-Traffic

     Group Server list       : Local
```

（3）在GM-1上查看加解密的状态

```shell
命令1:
GM-1#show crypto engine connections active 
Crypto Engine Connections

   ID  Type    Algorithm           Encrypt  Decrypt LastSeqN IP-Address
    1  IPsec   3DES+MD5                  0        0        0 172.16.0.0
    2  IPsec   3DES+MD5                  0        0        0 172.16.0.0
 1001  IKE     SHA+DES                   0        0        0 202.100.2.1
 1002  IKE     SHA+AES256                0        0        0 
解释：
ID:1001为IKE Phase1 SA（IKE第一阶段SA），使用的算法为DES，HASH为SHA。
ID:1002为Rekey SA（密钥更新安全关联策略），使用的算法为AES256，HASH为SHA。
ID:1和ID:2为IPSec SA（IPSec安全关联）

命令2:
GM-1#show crypto gdoi
GROUP INFORMATION

    Group Name               : mygroup  （GDOI组名）
    Group Identity           : 66666    （GDOI ID名）
    Rekeys received          : 0
    IPSec SA Direction       : Both

     Group Server list       : 202.100.1.1  （有两个KS服务器）
                               202.100.1.2
                               
    Group member             : 202.100.2.1      vrf: None （组成员自己的IP地址）
       Registration status   : Registered  （已经注册状态）
       Registered with       : 202.100.1.1 （注册在KS1上）
       Re-registers in       : 3137 sec
       Succeeded registration: 1
       Attempted registration: 1
       Last rekey from       : 0.0.0.0
       Last rekey seq num    : 0
       Unicast rekey received: 0
       Rekey ACKs sent       : 0
       Rekey Received        : never
       allowable rekey cipher: any
       allowable rekey hash  : any
       allowable transformtag: any ESP
          
    Rekeys cumulative
       Total received        : 0
       After latest register : 0
       Rekey Acks sents      : 0

 ACL Downloaded From KS 202.100.1.1:  （从KS1中心获取到的感兴趣流）
   access-list  permit ip 172.16.0.0 0.0.255.255 172.16.0.0 0.0.255.255

KEK POLICY:
    Rekey Transport Type     : Unicast  （组播更新）
    Lifetime (secs)          : 86258
    Encrypt Algorithm        : AES  （加密算法）
    Key Size                 : 256     
    Sig Hash Algorithm       : HMAC_AUTH_SHA  （完整性校验算法）
    Sig Key Length (bits)    : 1024     （长度）

TEK POLICY for the current KS-Policy ACEs Downloaded:
  FastEthernet1/0:
    IPsec SA:
        spi: 0xBA343921(3123984673)
        transform: esp-3des esp-md5-hmac 
        sa timing:remaining key lifetime (sec): (3245)
        Anti-Replay(Time Based) : 3 sec interval

命令3:
GM-1# show crypto gdoi ipsec sa 
SA created for group mygroup:
  FastEthernet1/0:
    protocol = ip
      local ident  = 172.16.0.0/16, port = 0
      remote ident = 172.16.0.0/16, port = 0
      direction: Both, replay(method/window): Time/3 sec
      
命令4:触发感兴趣流查看加解密状态
GM-1#ping 172.16.2.1 source 172.16.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 172.16.2.1, timeout is 2 seconds:
Packet sent with a source address of 172.16.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 8/10/56 ms
GM-1#show crypto engine connections active 
Crypto Engine Connections

   ID  Type    Algorithm           Encrypt  Decrypt LastSeqN IP-Address
    1  IPsec   3DES+MD5                  0      100        0 172.16.0.0
    2  IPsec   3DES+MD5                100        0        0 172.16.0.0
 1001  IKE     SHA+DES                   0        0        0 202.100.2.1
 1002  IKE     SHA+AES256                0        0        0 
```



##### 2.2 GETVPN单播更新在VRF中的应用

步骤1：基本配置初始化

（1）VRF-GM-1路由器配置初始化

```
hostname VRF-GM-1
ip vrf MGMT
 rd 10:10  （rd用来模拟MPLS VPN网络）
ip vrf Site-1
 rd 11:11
ip vrf Site-2
 rd 22:22
key chain ccie （用来做EIGRP MD5验证）
 key 1
  key-string cisco
interface Loopback100
 ip vrf forwarding Site-1
 ip address 192.168.1.1 255.255.255.0
interface Loopback200
 ip vrf forwarding Site-2
 ip address 192.168.1.1 255.255.255.0
interface FastEthernet0/0
 no shutdown
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip vrf forwarding MGMT
 ip address 202.100.3.1 255.255.255.0
interface FastEthernet0/0.11
 encapsulation dot1Q 11
 ip vrf forwarding Site-1
 ip address 10.1.1.1 255.255.255.0
 ip authentication mode eigrp 11 md5
 ip authentication key-chain eigrp 11 ccie
interface FastEthernet0/0.22
 encapsulation dot1Q 22
 ip vrf forwarding Site-2
 ip address 10.1.1.1 255.255.255.0
 ip authentication mode eigrp 22 md5
 ip authentication key-chain eigrp 22 ccie
ip route vrf MGMT 202.100.1.0 255.255.255.0 202.100.3.10
router eigrp 1122
 address-family ipv4 vrf Site-1 autonomous-system 11
  network 10.1.1.0 0.0.0.255
  network 192.168.1.0 0.0.0.255
  exit
 address-family ipv4 vrf Site-2 autonomous-system 22
  network 10.1.1.0 0.0.0.255
  network 192.168.1.0 0.0.0.255
  exit
```
（2）VRF-GM-2路由器配置初始化

```
hostname VRF-GM-2
ip vrf MGMT
 rd 10:10  （rd用来模拟MPLS VPN网络）
ip vrf Site-1
 rd 11:11
ip vrf Site-2
 rd 22:22
key chain ccie （用来做EIGRP MD5验证）
 key 1
  key-string cisco
interface Loopback100
 ip vrf forwarding Site-1
 ip address 192.168.2.1 255.255.255.0
interface Loopback200
 ip vrf forwarding Site-2
 ip address 192.168.2.1 255.255.255.0
interface FastEthernet0/0
 no shutdown
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip vrf forwarding MGMT
 ip address 202.100.3.2 255.255.255.0
interface FastEthernet0/0.11
 encapsulation dot1Q 11
 ip vrf forwarding Site-1
 ip address 10.1.1.2 255.255.255.0
 ip authentication mode eigrp 11 md5
 ip authentication key-chain eigrp 11 ccie
interface FastEthernet0/0.22
 encapsulation dot1Q 22
 ip vrf forwarding Site-2
 ip address 10.1.1.2 255.255.255.0
 ip authentication mode eigrp 22 md5
 ip authentication key-chain eigrp 22 ccie
ip route vrf MGMT 202.100.1.0 255.255.255.0 202.100.3.10
router eigrp 1122
 address-family ipv4 vrf Site-1 autonomous-system 11
  network 10.1.1.0 0.0.0.255
  network 192.168.2.0 0.0.0.255
  exit
 address-family ipv4 vrf Site-2 autonomous-system 22
  network 10.1.1.0 0.0.0.255
  network 192.168.2.0 0.0.0.255
  exit
```

（3）sw3交换机配置初始化

```
vtp mode transparent
vlan 10-11,22
interface Ethernet0/0
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast
interface range Ethernet0/1-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 spanning-tree portfast trunk
```

（4）ASA追加配置

```shell
interface GigabitEthernet0/2
 nameif DMZ
 security-level 50
 ip address 202.100.3.10 255.255.255.0
 no shutdown
router ospf 1
 network 202.100.3.0 255.255.255.0 area 0
access-list DMZ extended permit icmp any any 
access-list DMZ extended permit udp 202.100.3.0 255.255.255.0 202.100.1.0 255.255.255.0 eq 848
access-group DMZ in interface DMZ
```

（5）测试网络连通性

```shell
VRF-GM-1#ping vrf MGMT 202.100.3.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 202.100.3.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 12/18/24 ms
VRF-GM-1#ping vrf MGMT 202.100.3.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 202.100.3.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/9/12 ms
VRF-GM-1#ping vrf Site-1 10.1.1.2  
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 16/19/20 ms
VRF-GM-1#ping vrf Site-2 10.1.1.2  
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 12/18/20 ms

测试VRF EIGRP
VRF-GM-1#show ip route vrf Site-1 eigrp 
Routing Table: Site-1
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is not set

D     192.168.2.0/24 [90/156160] via 10.1.1.2, 00:01:10, FastEthernet0/0.11

VRF-GM-1#show ip route vrf Site-2 eigrp 
Routing Table: Site-2
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is not set

D     192.168.2.0/24 [90/156160] via 10.1.1.2, 00:01:15, FastEthernet0/0.22

VRF-GM-1#ping vrf Site-1 192.168.2.1 source 192.168.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 12/18/20 ms
VRF-GM-1#ping vrf Site-2 192.168.2.1 source 192.168.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/11/16 ms
```
步骤2：在首要密钥服务器KS-1上添加关于VRF Site GETVPN配置

（1）配置ISAKMP第一阶段策略

```
crypto isakmp key cisco address 202.100.3.1
crypto isakmp key cisco address 202.100.3.2
```

（2）配置感兴趣流

```
ip access-list extended GETVPN-VRF-Traffic （配置感兴趣流，后期如果需要更改必须在KS-1和KS-2上配置相同）
 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
```

（3）配置KS1为密钥服务器

```
KS-1(config)#crypto gdoi group VRF_Group
KS-1(config-gdoi-group)#identity number 88888
KS-1(config-gdoi-group)#server local
KS-1(gdoi-local-server)#address ipv4 202.100.1.1
```

（5）配置密钥更新（单播）

```
KS-1(gdoi-local-server)#rekey authentication mypubkey rsa getvpnkey
KS-1(gdoi-local-server)#rekey algorithm aes 256 （默认加密算法为3des，HASH算法为SHA-1。此配置为可选，配置密钥更新信息的加密算法为aes）
KS-1(gdoi-local-server)#rekey transport unicast （配置GETVPN使用单播传输密钥更新信息，注意默认为组播）
```

（6）配置IPSec 安全关联

```
KS-1(gdoi-local-server)#sa ipsec 1
KS-1(gdoi-sa-ipsec)#match address ipv4 GETVPN-VRF-Traffic
KS-1(gdoi-sa-ipsec)#profile getvpn-profile
KS-1(gdoi-sa-ipsec)#replay time window-size 3 （配置为选项，启用基于时间的防重放，窗口大小为3秒）
```

步骤3：在次要密钥服务器KS-2上添加关于VRF Site GETVPN配置

（1）配置ISAKMP第一阶段策略

```
crypto isakmp key cisco address 202.100.3.1
crypto isakmp key cisco address 202.100.3.2
```

（2）配置感兴趣流

```
ip access-list extended GETVPN-VRF-Traffic （配置感兴趣流，后期如果需要更改必须在KS-1和KS-2上配置相同）
 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
```

（3）配置KS-2为密钥服务器

```
KS-2(config)#crypto gdoi group VRF_Group
KS-2(config-gdoi-group)#identity number 88888
KS-2(config-gdoi-group)#server local
KS-2(gdoi-local-server)#address ipv4 202.100.1.2
```

（5）配置密钥更新（单播）

```
KS-2(gdoi-local-server)#rekey authentication mypubkey rsa getvpnkey
KS-2(gdoi-local-server)#rekey algorithm aes 256 （默认加密算法为3des，HASH算法为SHA-1。此配置为可选，配置密钥更新信息的加密算法为aes）
KS-2(gdoi-local-server)#rekey transport unicast （配置GETVPN使用单播传输密钥更新信息，注意默认为组播）
```

（6）配置IPSec 安全关联

```
KS-2(gdoi-local-server)#sa ipsec 1
KS-2(gdoi-sa-ipsec)#match address ipv4 GETVPN-VRF-Traffic
KS-2(gdoi-sa-ipsec)#profile getvpn-profile
KS-2(gdoi-sa-ipsec)#replay time window-size 3 （配置为选项，启用基于时间的防重放，窗口大小为3秒）
```

步骤4：配置VRF-GM-1和VRF-GM-2的策略

配置VRF-GM-X的IKE Phase1 Policy（IKE第一阶段策略）

```shell
crypto isakmp policy 10
 authentication pre-share
crypto keyring getvpn vrf MGMT 
  pre-shared-key address 202.100.1.1 key cisco
  pre-shared-key address 202.100.1.2 key cisco
crypto gdoi group VRF_Group
 identity number 88888
 server address ipv4 202.100.1.1
 server address ipv4 202.100.1.2
 client registration interface FastEthernet0/0.10
crypto map cisco 10 gdoi 
 set group VRF_Group
interface FastEthernet0/0.11
 crypto map cisco
interface FastEthernet0/0.22
 crypto map cisco
```

步骤5：查看测试结果

（1）在KS-1上查看组成员信息

```
KS-1#show crypto gdoi ks members 

Group Member Information : 

Number of rekeys sent for group mygroup : 5

Group Member ID    : 202.100.2.1
 Group ID          : 66666
 Group Name        : mygroup
 Key Server ID     : 202.100.1.1
 Rekeys sent       : 5
 Rekeys retries    : 0
 Rekey Acks Rcvd   : 5
 Rekey Acks missed : 0

 Sent seq num : 2       3       4       1
Rcvd seq num :  2       3       4       1

Group Member ID    : 202.100.2.2
 Group ID          : 66666
 Group Name        : mygroup
 Key Server ID     : 202.100.1.1
 Rekeys sent       : 5
 Rekeys retries    : 0
 Rekey Acks Rcvd   : 5
 Rekey Acks missed : 0

 Sent seq num : 2       3       4       1
Rcvd seq num :  2       3       4       1

Number of rekeys sent for group VRF_Group : 0

Group Member ID    : 202.100.3.1
 Group ID          : 88888
 Group Name        : VRF_Group
 Key Server ID     : 202.100.1.1
 Rekeys sent       : 0
 Rekeys retries    : 0
 Rekey Acks Rcvd   : 0
 Rekey Acks missed : 0

 Sent seq num : 0       0       0       0
Rcvd seq num :  0       0       0       0

Group Member ID    : 202.100.3.2
 Group ID          : 88888
 Group Name        : VRF_Group
 Key Server ID     : 202.100.1.1
 Rekeys sent       : 0
 Rekeys retries    : 0
 Rekey Acks Rcvd   : 0
 Rekey Acks missed : 0

 Sent seq num : 0       0       0       0
Rcvd seq num :  0       0       0       0
```

（2）在VRF-GM-1上查看加解密状态

```shell
VRF-GM-1#show crypto gdoi ipsec sa
SA created for group VRF_Group:
  FastEthernet0/0.11:
  FastEthernet0/0.22:
    protocol = ip
      local ident  = 192.168.1.0/24, port = 0
      remote ident = 192.168.2.0/24, port = 0
      direction: Both, replay(method/window): Time/3 sec
    protocol = ip
      local ident  = 192.168.2.0/24, port = 0
      remote ident = 192.168.1.0/24, port = 0
      direction: Both, replay(method/window): Time/3 sec

VRF-GM-1#ping vrf Site-1 192.168.2.1 source 192.168.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 16/20/24 ms

VRF-GM-1#ping vrf Site-2 192.168.2.1 source 192.168.1.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 16/19/24 ms

VRF-GM-1#show crypto ipsec sa 
   protected vrf: Site-1
   local  ident (addr/mask/prot/port): (192.168.1.0/255.255.255.0/0/0)
   remote ident (addr/mask/prot/port): (192.168.2.0/255.255.255.0/0/0)
   current_peer 0.0.0.0 port 848
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 100, #pkts encrypt: 100, #pkts digest: 100
    #pkts decaps: 100, #pkts decrypt: 100, #pkts verify: 100
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

   protected vrf: Site-2
   local  ident (addr/mask/prot/port): (192.168.1.0/255.255.255.0/0/0)
   remote ident (addr/mask/prot/port): (192.168.2.0/255.255.255.0/0/0)
   current_peer 0.0.0.0 port 848
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 100, #pkts encrypt: 100, #pkts digest: 100
    #pkts decaps: 100, #pkts decrypt: 100, #pkts verify: 100
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0
```

