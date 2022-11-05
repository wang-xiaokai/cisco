## GETVPN





#### GETVPN组播更新配置实例

<img src="https://cdn.jsdelivr.net/gh/wang-xiaokai/images/202211051842522.png" alt="GEVPN拓扑图" title="GEVPN拓扑图" width="70%" height="50%" />

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



#### GETVPN组成员访问控制列表配置

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



#### GETVPN 双KS之间的协作

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

