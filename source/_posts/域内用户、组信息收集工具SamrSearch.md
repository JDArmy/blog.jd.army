---
title: 域内用户、组信息收集工具SamrSearch
tags: [域渗透,工具]
categories: 渗透工具
date: 2022-02-17 10:08:16
---

 在impacket进行域渗透中，通过MS-SAMR协议实现net user和net group的功能，能方便在渗透过程中，如果域内没有可控的windows主机，但能通过web手段获取到域内用户账号的情况下，来对用户权限、用户信息和组信息进行收集。

### Install

Python 3.5+impacket

### Usage

```
usage: samrsearch.py [-h] [-csv] [-ts] [-debug] [-username USERNAME] [-groupname GROUPNAME] [-dc-ip ip address] [-target-ip ip address] [-port [destination port]] [-hashes LMHASH:NTHASH] [-no-pass] [-k] [-aesKey hex key] target

This script downloads the list of users for the target system.

positional arguments:
  target                [[domain/]username[:password]@]<targetName or address>

optional arguments:
  -h, --help            show this help message and exit
  -csv                  Turn CSV output
  -ts                   Adds timestamp to every logging output
  -debug                Turn DEBUG output ON
  -username USERNAME    Username you want to search
  -groupname GROUPNAME  Group you want to search

connection:
  -dc-ip ip address     IP Address of the domain controller. If ommited it use the domain part (FQDN) specified in the target parameter
  -target-ip ip address
                        IP Address of the target machine. If ommited it will use whatever was specified as target. This is useful when target is the NetBIOS name and you cannot resolve it
  -port [destination port]
                        Destination port to connect to SMB Server

authentication:
  -hashes LMHASH:NTHASH
                        NTLM hashes, format is LMHASH:NTHASH
  -no-pass              don't ask for password (useful for -k)
  -k                    Use Kerberos authentication. Grabs credentials from ccache file (KRB5CCNAME) based on target parameters. If valid credentials cannot be found, it will use the ones specified in the command line
  -aesKey hex key       AES key to use for Kerberos Authentication (128 or 256 bits)
```



net user windows8 /domain: `python3 samrsearch.py windows.local/test:aaa@172.16.178.9 -username "windows8"`

![图片](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/02/63c70147e793e893e7d7298a0456d3b6.png)



net group "Domain Admins" /domain:`python3 samrsearch.py windows.local/test:aaa@172.16.178.9 -groupname "Domain Admins"`

![图片](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/02/a956f4fd3d18953dba27b9cb96ad498e.png)



如果未添加参数，将对dump所有域内的用户信息。

```
python3 samrsearch.py windows.local/test:aaa@172.16.178.9
```

![图片](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/02/3465f2847676e73fa6d5c46e76d02fee.png)

### github地址

https://github.com/knightswd/SamrSearch

欢迎大家使用，有什么问题尽管提 (•‾̑⌣‾̑•)