﻿# 如何配置 IKEv2 VPN: Windows 7 和更新版本

*其他语言版本: [English](ikev2-howto.md), [简体中文](ikev2-howto-zh.md).*

**重要提示：** 本指南仅适用于**高级用户**。

Windows 7 和更新版本支持 IKEv2 和 MOBIKE 标准，通过 Microsoft 的 Agile VPN 功能来实现。因特网密钥交换 （英语：Internet Key Exchange，简称 IKE 或 IKEv2）是一种网络协议，归属于 IPsec 协议族之下，用以创建安全联结 （Security association，SA）。与 IKE 版本 1 相比较，IKEv2 带来许多<a href="https://en.wikipedia.org/wiki/Internet_Key_Exchange#Improvements_with_IKEv2" target="_blank">功能改进</a>，比如通过 MOBIKE 实现 standard mobility support，以及支持在同一个 NAT （比如家用路由器）后面的多个设备同时连接。

Libreswan 支持通过使用 RSA 签名算法的 X.509 Machine Certificates 来对 IKEv2 客户端进行身份验证。该方法无需 IPsec PSK, 用户名或密码。除了 Windows 之外，它也可用于 <a href="https://wiki.strongswan.org/projects/strongswan/wiki/AndroidVpnClient" target="_blank">strongSwan Android VPN 客户端</a> （未测试）。以下的步骤举例说明如何在 Libreswan 上配置 IKEv2。

1. 获取服务器的公共和私有 IP 地址，并确保它们的值非空。注意，这两个 IP 地址可以相同。

   ```bash
   $ PUBLIC_IP=$(dig @resolver1.opendns.com -t A -4 myip.opendns.com +short)
   $ PRIVATE_IP=$(ip -4 route get 1 | awk '{print $NF;exit}')
   $ echo "$PUBLIC_IP"
   (Your public IP is displayed)
   $ echo "$PRIVATE_IP"
   (Your private IP is displayed)
   ```

1. 在 `/etc/ipsec.conf` 文件中添加一个新的 IKEv2 连接:

   ```bash
   $ cat >> /etc/ipsec.conf <<EOF

   conn ikev2-cp
     left=$PRIVATE_IP
     leftcert=$PUBLIC_IP
     leftid=@$PUBLIC_IP
     leftsendcert=always
     leftsubnet=0.0.0.0/0
     leftrsasigkey=%cert
     right=%any
     rightaddresspool=192.168.43.10-192.168.43.250
     rightca=%same
     rightrsasigkey=%cert
     modecfgdns1=8.8.8.8
     modecfgdns2=8.8.4.4
     narrowing=yes
     dpddelay=30
     dpdtimeout=120
     dpdaction=clear
     auto=add
     ikev2=insist
     rekey=no
     fragmentation=yes
     forceencaps=yes
     ike=3des-sha1,aes-sha1,aes256-sha2_512,aes256-sha2_256
     phase2alg=3des-sha1,aes-sha1,aes256-sha2_512,aes256-sha2_256
   EOF
   ```

1. 生成 CA 和 VPN 服务器证书：

   ```bash
   $ certutil -S -x -n "Example CA" -s "O=Example,CN=Example CA" -k rsa -g 4096 -v 12 -d sql:/etc/ipsec.d -t "CT,," -2

   A random seed must be generated that will be used in the
   creation of your key.  One of the easiest ways to create a
   random seed is to use the timing of keystrokes on a keyboard.

   To begin, type keys on the keyboard until this progress meter
   is full.  DO NOT USE THE AUTOREPEAT FUNCTION ON YOUR KEYBOARD!

   Continue typing until the progress meter is full:

   |************************************************************|

   Finished.  Press enter to continue:

   Generating key.  This may take a few moments...

   Is this a CA certificate [y/N]?
   y
   Enter the path length constraint, enter to skip [<0 for unlimited path]: >
   Is this a critical extension [y/N]?
   N

   $ certutil -S -c "Example CA" -n "$PUBLIC_IP" -s "O=Example,CN=$PUBLIC_IP" -k rsa -g 4096 -v 12 -d sql:/etc/ipsec.d -t ",," -1 -6 -8 "$PUBLIC_IP"

   A random seed must be generated that will be used in the
   creation of your key.  One of the easiest ways to create a
   random seed is to use the timing of keystrokes on a keyboard.

   To begin, type keys on the keyboard until this progress meter
   is full.  DO NOT USE THE AUTOREPEAT FUNCTION ON YOUR KEYBOARD!

   Continue typing until the progress meter is full:

   |************************************************************|

   Finished.  Press enter to continue:

   Generating key.  This may take a few moments...

                   0 - Digital Signature
                   1 - Non-repudiation
                   2 - Key encipherment
                   3 - Data encipherment
                   4 - Key agreement
                   5 - Cert signing key
                   6 - CRL signing key
                   Other to finish
    > 0
                   0 - Digital Signature
                   1 - Non-repudiation
                   2 - Key encipherment
                   3 - Data encipherment
                   4 - Key agreement
                   5 - Cert signing key
                   6 - CRL signing key
                   Other to finish
    > 2
                   0 - Digital Signature
                   1 - Non-repudiation
                   2 - Key encipherment
                   3 - Data encipherment
                   4 - Key agreement
                   5 - Cert signing key
                   6 - CRL signing key
                   Other to finish
    > 8
   Is this a critical extension [y/N]?
   N
                   0 - Server Auth
                   1 - Client Auth
                   2 - Code Signing
                   3 - Email Protection
                   4 - Timestamp
                   5 - OCSP Responder
                   6 - Step-up
                   7 - Microsoft Trust List Signing
                   Other to finish
    > 0
                   0 - Server Auth
                   1 - Client Auth
                   2 - Code Signing
                   3 - Email Protection
                   4 - Timestamp
                   5 - OCSP Responder
                   6 - Step-up
                   7 - Microsoft Trust List Signing
                   Other to finish
    > 8
   Is this a critical extension [y/N]?
   N
   ```

1. 生成客户端证书，并且导出 p12 文件。该文件包含客户端证书，私钥以及 CA 证书：

   ```bash
   $ certutil -S -c "Example CA" -n "winclient" -s "O=Example,CN=winclient" -k rsa -g 4096 -v 12 -d sql:/etc/ipsec.d -t ",," -1 -6 -8 "winclient"

   -- repeat same extensions as above --

   $ pk12util -o winclient.p12 -n "winclient" -d sql:/etc/ipsec.d

   Enter password for PKCS12 file:
   Re-enter password:
   pk12util: PKCS12 EXPORT SUCCESSFUL
   ```

   可以重复该步骤来为更多的客户端生成证书，但必须把所有的 `winclient` 换成 `winclient2`，等等。

1. 证书数据库现在应该包含以下内容：

   ```bash
   $ certutil -L -d sql:/etc/ipsec.d

   Certificate Nickname                               Trust Attributes
                                                      SSL,S/MIME,JAR/XPI

   Example CA                                         CTu,u,u
   ($PUBLIC_IP)                                       u,u,u
   winclient                                          u,u,u
   ```

1. 重启 IPsec 服务：

   ```bash
   $ service ipsec restart
   ```

1. 文件 `winclient.p12` 应该被安全的传送到 Windows 客户端计算机，并且导入到 Computer 证书存储。在导入 CA 证书后，它必须被放在 "Trusted Root Certification Authorities" 目录中。

   详细的操作步骤：   
   https://wiki.strongswan.org/projects/strongswan/wiki/Win7Certs

1. 在 Windows 计算机上添加一个新的 IKEv2 VPN 连接。

   https://wiki.strongswan.org/projects/strongswan/wiki/Win7Config

1. 启用新的 IKEv2 VPN 连接，并且开始使用自己的专属 VPN！

   https://wiki.strongswan.org/projects/strongswan/wiki/Win7Connect

   连接成功后，你可以到<a href="https://www.whatismyip.com" target="_blank">这里</a>检测你的 IP 地址，应该显示为`你的 VPN 服务器 IP`。

## 参考链接

* https://libreswan.org/wiki/VPN_server_for_remote_clients_using_IKEv2
* https://libreswan.org/wiki/HOWTO:_Using_NSS_with_libreswan
* https://libreswan.org/man/ipsec.conf.5.html
* https://wiki.strongswan.org/projects/strongswan/wiki/Windows7
