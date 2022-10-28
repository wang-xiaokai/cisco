## VPN

SVTI仅仅可以用在两个站点之间是静态固定Global地址，无法使用在站点在PAT后面或站点使用动态Global地址的场景。

DMVPN是思科的私有VPN技术。其实就是高级版的GRE over IPsec VPN。HUB可以是静态IP地址或DDNS的动态Global地址。Spoke即可以是动态Global地址也可以在PAT后面。但两个spoke之间需要动态建立spoke-to-spoke隧道的前提条件是一侧必须不在PAT后面。如果两个spoke都在PAT后面，流量将在HUB中心转发，无法动态建立spoke-to-spoke隧道。

DMVPN，GRE over IPSec，L2TP over IPsec 加密点等于通信点，推荐使用传输模式进行封装

GRE over IPSec两端必须是固定Global IP地址















