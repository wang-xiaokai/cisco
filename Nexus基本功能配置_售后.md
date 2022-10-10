## Nexus交换机基本功能

- 开启SSH服务器功能

no feature ssh

ssh key rsa 2048 force

**feature ssh**

ssh login-attempts 5

show run security

username  admin password *yourpassword*

ssh admin@*ip_adress* vrf management 

ping *ip_adress* vrf management 

- 开启Telnet功能

**feature telnet**

telnet *ip_adress* vrf management 

ping *ip_adress* vrf management 