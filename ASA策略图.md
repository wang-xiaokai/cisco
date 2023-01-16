### ASA策略图









```mermaid
graph TB;
a[VPN连接]-->b1(L2L);
a[VPN连接]-->b2(EzVPN);
a[VPN连接]-->b3(SSLVPN);

subgraph  选择Tunnel-Group
  b1(L2L)
  b2(EzVPN)
  b3(SSLVPN)
end

b3(SSLVPN)-->c(Authentication-server-group);
c(Authentication-server-group)-->d(User);
e(username attributes)-->d(User);
d(User)-->f(user group-policy);
f(user group-policy)-->g(tunnel-group default-group-policy);
b2(EzVPN).->g(tunnel-group default-group-policy);
b3(SSLVPN).->g(tunnel-group default-group-policy);
g(tunnel-group default-group-policy)-->h(group-policy DfltGrpPolicy);
h(group-policy DfltGrpPolicy)-->i(tunnel-group policy);
i(tunnel-group policy)-->j[用户最终VPN策略];
```

下面五个策略，越低越优先

- 策略优先级1：username attributes

```
username ssluser attributes
 vpn-framed-ip-address 1.2.3.4 255.255.255.0
```

- 策略优先级2：user group-policy

```
ip local pool POOL2 2.2.2.1-2.2.2.100
group-policy user-group-policy internal
group-policy user-group-policy attributes
 vpn-tunnel-protocol ssl-client ssl-clientless
 address-pools value POOL2
username ssluser attributes
 vpn-group-policy user-group-policy
```

- 策略优先级3：tunnel-group default-group-policy

```
ip local pool POOL3 3.3.3.1-3.3.3.100
group-policy tunnel-group-policy attributes
 address-pools value POOL3
tunnel-group anyconnect-tunnel-group type remote-access
tunnel-group anyconnect-tunnel-group general-attributes
 default-group-policy tunnel-group-policy
```

- 策略优先级4：group-policy DfltGrpPolicy

```
ip local pool POOL4 4.4.4.1-4.4.4.100
group-policy DfltGrpPolicy attributes
 address-pools value POOL4
```

- 策略优先级5：tunnel-group policy  

```
ip local pool POOL5 5.5.5.1-5.5.5.100
tunnel-group anyconnect-tunnel-group type remote-access
tunnel-group anyconnect-tunnel-group general-attributes
 address-pool POOL5
```



https://www.packetswitch.co.uk/cisco-asa-anyconnect-vpn/

