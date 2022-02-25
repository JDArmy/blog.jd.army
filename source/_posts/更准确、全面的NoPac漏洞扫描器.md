---
title: 更准确、全面的NoPac漏洞扫描器
tags: [NoPac,提权]
categories: 渗透工具
date: 2022-02-16 19:20:16
---


NoPac域提权漏洞扫描器，相比于网上公开的扫描器，它能自动扫描更多的域控，并且能更精确的识别漏洞。

<!--more -->

### 原理

为什么能查询到更多域控？

  通过DNS查询`_msdcs.aaa.com`来获取域控的地址,相比LDAP或者SAMR协议查询获取到的域控地址，这种方法能查询到更多域控。（因为DNS查询能查到已经脱域但未在dns中注销的域控）漏洞原作者的查询方法是通过LDAP进行查询。

为什么能更精确识别漏洞？

  通过分析微软对此漏洞的主要补丁，能发现它修改的方法主要是通过在PAC认证中，增加了一个类型为0x10的结构体，在TGS获取阶段增加对0x10结构体的身份检查，来防止NoPac中的身份伪造。因此通过识别PAC中是否存在0x10的结构体就行。至于如何解密PAC文件呢？通过当前用户请求自己的S4U2self票据，从而使用自己的hash来解密获取的PAC。漏洞原作者的查询方法是通过判断返回的tgt的大小来判断漏洞是否存在，如果tgt中有pac，则漏洞存在。但是在微软后面的更新中，有个策略是无论Include Pac的参数是什么，都返回PAC包，这将导致判断失误。

### Install

`Python3.5+impacket`

- pip install aiodns

### Usage

```
usage: NopacScan.py [-h] [-debug] [-hashes LMHASH:NTHASH] [-dc-ip ip address] [-dns-ip dns ip address] [-dns-port dns port] credentials

positional arguments:
  credentials           domain/username[:password]

optional arguments:
  -h, --help            show this help message and exit
  -debug                Turn DEBUG output ON

authentication:
  -hashes LMHASH:NTHASH
                        NTLM hashes, format is LMHASH:NTHASH
  -dc-ip ip address     IP Address of the domain controller.
  -dns-ip dns ip address
                        Dns search Ip(default is DC-IP)
  -dns-port dns port    Dns search port (default is 53) scan with 
```

### scan

```
python3 NopacScan.py windows.local/test:aaaa -dc-ip 172.16.178.9
```

  通过dc-ip传入域控地址后，将默认通过DC的dns服务器来进行dns查询，并自动扫描所有查询到的DC，后进行漏洞探测。

![图片](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/02/f1fd272c807f0186a3f8c52a85a5ec22.png)

### github地址

https://github.com/knightswd/NoPacScan 

欢迎大家使用，有什么问题尽管提 (•‾̑⌣‾̑•)