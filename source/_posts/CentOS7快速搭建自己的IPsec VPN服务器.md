---
title: CentOS7快速搭建自己的IPsec VPN服务器
tags: vpn
categories: linux
---

#   简述 

   由于工(xiu)作(xian)需要很多时候需要科学上网，尝试了很多免费的科学上网软件（要么就是限时，要么就是疯狂看广告），
但是这些软件都十分的不稳定（毕竟人家就是要让你出钱啊喂）。这里记录下一键式的搭建技巧：）
   
<!--more-->

#   准备工作

*   一台国外的VPS
    
    由于原来使用的某里，某讯的国内的服务器，备案这个事情折腾了我很久，很烦，最后还没好...趁着这个机会刚好换一台 哦嚯嚯嚯~
    价格大概4.9刀每个月，可月付哦。[不知道怎么购买的点这里](https://www.vps234.com/hostwinds-purchase-tutorial/ "hostwinds")
    
*   没有了

#   一键式搭建 IPsec VPN

*   如果是CentOS，直接更改下面的密钥VPN_IPSEC_PSK，用户名VPN_USER，，密码VPN_PASSWORD，回车即可等待完成
    ```
    wget https://git.io/vpnsetup-centos -O vpnsetup.sh && sudo VPN_IPSEC_PSK='XXX' VPN_USER='XXX' VPN_PASSWORD='XXX' sh vpnsetup.sh
    ```
    
*   当然详情请点 [这里哦](https://github.com/duanshaojie/setup-ipsec-vpn/blob/master/README-zh.md "hostwinds")
    我fork的，可以给原作者点个star