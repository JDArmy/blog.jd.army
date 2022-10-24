---
title: Lsass memory dump
tags: [windows,lsass]
categories: windows
date: 2022-10-10 14:26:16
---

[toc]

lsass是windows中处理本地安全和登录策略的重要进程，几乎所有的windows身份认证程序都离不开lsass进程。因此在lsass的内存中会保存用户相关的凭证。它是windows主机中凭证的重要组成部分，因此获取lsass内存也是MITRE ATT&CK框架中Credential Access战术下的重要技术点。

![image-20220927115856240](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/ba3f4cb1e9552a139297eae6dc5b1313.png)

在整个unified kill chain中Credential Access的环节起着重要的作用。在稳定获得初始化立足点后，需要获取目标主机上的相关凭证，并通过这些凭证进行后续的横向移动，当凭证获取不足时，无疑会加大后续的横向移动难度，因此Credential Access也是攻防对抗的重要战场。

![image-20220927120240172](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/633b85ad13d7ea976c39cdd142bd75ba.png)

在lsass内存获取中，最容易想到的就是mimikatz，直接通过mimikatz就能获取lsass中保存的所有凭证，但是通常杀软会对mimikatz进行严格的监控与防御，在mimikatz运行过程中，主要分为两步，第一步获取lsass内存，第二步解析lsass内存获取密码或者hash。其中第二步可以离线进行，因此对抗点就出现在了第一步获取lsass内存的行为中了。

本文主要介绍dump lsass内存的相关技术、免杀对抗，帮助读者更好的理解lsass内存dump和相关的免杀对抗技巧。

## 一、使用签名程序dump lsass内存

许多带签名的正常程序通常需要用到内存dump功能，让用户能查看进程在内存中的信息。此方法主要是通过签名程序，利用程序的正常功能来获取windows lsass中的内存。

### Procdump

Procdump是由SysInternal团队开发的一个windows进程和cpu监控程序，后由微软对其进行签名。在此工具的正常功能中，需要在cpu峰值期间生成崩溃转储文件，方便管理员来确定峰值原因，通过以下命令就能直接生成转储文件了。

使用命令：`procdump.exe -ma LsassPid`

### Process Explorer

Procexp也是是SysInternal团队开发的windows进程监控工具，此工具主要对windows的进程进行监控，包括进程相关文件、进程属性、进程状态等进行监控。在此工具的正常功能中，需要对进程内存进行dump，来查看进程运行过程中的内存。

此工具直接通过图形界面来创建dump文件。

### SQLDumper

SQLDumper是Microsoft SQL Server程序的一部分，主要是为了调试sqlServer和相关进程的内存。

使用命令：`sqldump.exe LsassPid 0 0x01100`

### ProcessDump

思科Cisco Jabber会在`C:\program file (86)\cisco systems\cisco jabber\x64\`目录下存放processdump.exe，jabber会使用此程序来进行内存转储。

使用命令：`processdump.exe (ps lsass).id c:\lsass.dmp`

## 二、使用windows自带的应用/命令来dump lsass内存

### Task Manager

windows任务管理器，可直接通过图形化界面打开windows任务管理器，后直接右键单击相关进程，可直接dump出lsass的进程。

### Comsvcs.dll

此方法主要是通过rundll32来调用comsvcs.dll中的MiniDump来直接dump相关内存。

MiniDumpW通过OpenProcess+CreateFileW+MiniDumpWriteDump函数来dump内存。

![image-20220927145121881](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/3d353c240d20376c93fcefbbf68924e7.png)

使用命令：`rundll32 C:\windows\system32\comsvcs.dll MiniDump "LsassPid dump.bin full"`

### rdrleakdiag

该程序是windows自带的Microsoft Windows Resource Leak Diagnostic，主要用于windows诊断相关资源泄露，它默认安装在windows7、windows8、windows10、windows server2012以上中。

使用命令：`rdrleakdiag /p LsassPid /O C:\ /fullmemdmp /wait 1`

## 三、使用windows机制来dump内存

### SlientProcessExit

此方法是通过windows api函数让windows的WerFault.exe（windows错误报告进程）程序dump进程内存。

#### Silent Process Exit机制

在windows7中引入了一个Silent Process Exit机制，此机制的目的是为了监控windows的进程终止（通常是两种方式，一种是自己调用ExitProcess()终止，另一种是其他进程调用TerminateProcess()来终止此程序）。并提供在Silent Process Exit之后的处理行为，主要有以下三种。

* 启动一个新的进程
* messagebox展示
* 创建dump文件

此机制的目的是为了方便对windows异常进程debug操作，发现windows异常原因。在windows中的具体进程是WerFault.exe。

#### dump lsass进程

在一个正常的Silent Process Exit机制中，需要进程异常崩溃或者退出，但是对于lsass进程，一但崩溃或者退出，将会导致系统重新启动，因此无法采用正常的windows业务逻辑来dump lsass。

既然无法使用正常的业务逻辑，那如何才能不退出lsass，dump内存呢？在windows中，有个RtlReportSilentProcessExit() API，此api的作用主要是通知windows错误报告服务，哪个进程正在执行静默退出。接受到此信号后，WerFault.exe就会dump该程序的内存。通过此方法，就不需要lsass退出了。

在dump lsass之前，还需要修改以下两个注册表值，来通知计算机对哪个进程执行Silent Process Exit机制。

`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\lsass.exe`
`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\lsass.exe`

相关工具见：

### LsassShtinkering

这个方法是今年defcon大会上分享的一种dump lsass的方法。此方法主要通过ALPC模拟正常的lsass异常信息发送到WerFault.exe来dump lsass内存，相比于SlientProcessExit机制dump内存，它有更好的隐秘性（windows日志记录不会保存相关恶意进程，从日志分析整个过程就像lsass主动dump内存），能够更好的规避杀软。

相关原理见ppt：

原作者的项目是个demo形式，我添加了注册表修改和令牌伪造，变成了自动化工具。

工具地址：



## 四、免杀对抗

了解了上述的相关技术后，我们重新来看下dump lsass技术的关键步骤。

* 1、OpenProcess获取lsass进程句柄。
* 2、通过MiniDumpWriteDump读取lsass进程内存，并将结果保存到文件。

MiniDumpWriteDump的一系列核心流程，如图所示(具体实现在dbgcore.dll中)

![image-20221010150734673](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/e354f558cfbd0c38b8c73c2faec54405.png)

读取内存的函数：

![image-20221008190050807](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/b79c848766d1531553c9b874f9df26c0.png)

![image-20221008190111057](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/1babddbeb8308a35c9ad0f25162f93c2.png)

申请内存的函数：

![image-20221008184723706](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/fda89a02df5aed1fddfb790a2f3ee402.png)

写入文件的函数：

![image-20221008184812620](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/0194d18c123e75ea25b057eb410a038e.png)

因此对抗中存在两个点：

* 对抗点一：通过OpenProcess获取lsass进程句柄的操作，当OpenProcess进程时，会注册句柄回调，通过回调就很容易发现调用lsass的恶意进程。

* 对抗点二：是MiniDumpWriteDump函数dump进程内存的操作中，当dump内存时，会涉及到读取lsass内存和创建文件并将内存写入文件。

下面我们来看看针对这些行为，怎么进行相关攻防。

### Dumpert

此工具仍然使用MiniDumpWriteDump函数来对内存进行dump，但是相比与传统的dump，它先使用hook的方式对NtReadVirtualMemory函数解edr的hook，后通过syscall的方式调用ZwOpenProcess+NtCreateFile+MiniDumpWriteDump来进行内存dump，能有效的规避用户层的杀软。

项目地址：

### PssCaptureSnapshot API

通过PsscaptureSnapshot函数可以创建lsass的进程快照，后通过MiniDumpWriteDump来直接从进程快照中来dump lsass快照内存，规避了对抗点二中edr针对lsass的内存读写保护。

### pypykatz

pypykatz中主要解决了对抗点一的问题，在不直接通过OpenProcess函数获取lsass进程句柄的情况下，通过复制其他进程对lsass进程的访问句柄来间接获取lsass进程句柄。下面是它的具体方法：

1. 获取SeDebugPrivilege权限，让该进程能获取其他进程的句柄
2. 通过NtQuerySystemInformation来获取所有进程打开的句柄及句柄的PID信息。
3. OpenProcess以PROCESS_DUP_HANDLE（复制句柄权限）权限打开进程。
4. 通过NtDuplicateObject来获取其他进程副本。
5. 通过NtQueryObject来判断是否是句柄。
6. 如果是句柄，则通过QueryFullProcessImageName来显示出句柄的可执行文件路径，如果是lsass.exe文件，则调用后续的MiniDumpWriteDump来获取lsass内存。

### MirrorDump

一、针对OpenProcess获取进程句柄的监控，MirrorDump通过将获取lsass进程句柄的操作通过加载SSP的dll来实现。SSP的认证过程是由LSASS进程完成的，在SSP加载完dll后，在dll中获取MirrDump的进程句柄，并将当前进程句柄复制给MirrDump进程。因为加载SSP dll的进程是lsass，因此MirrDump就获取到了lsass的进程句柄。

在系统中的具体行为表现为：**LSASS进程通过OpenProcess获取自己的进程句柄，并将句柄复制给MirrorDump进程。**

二、针对文件的写，MirrorDump通过MinHook来对MiniDumpWriteDump中涉及到的SetFilePointer、GetFileSize、WriteFile函数进行hook，从而改变相关行为，规避杀软查杀。

MirrorDump具体流程：

![image-20221010152533409](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/72f03913438f95379ebc55aa4c7c1365.png)

### MalSeclogon

一、针对OpenProcess获取进程句柄的监控，MalSeclogon采用两种方法来获取lsass进程的句柄，第一种是通过CreateProcessWithLogonW/CreateProcessWithTokenW的父进程欺骗和CreateProcessWithLogonW函数的句柄泄露来从其他进程中寻找lsass泄露的句柄。第二种是在CreateProcessWithLogonW/CreateProcessWithTokenW进行父进程欺骗时，会打开lsass进程句柄，通过条件竞争获取到此lsass进程句柄。下面将详细介绍两种方法。

方法一：

1. 枚举进程句柄并找到lsass打开的句柄。
2. 修改当前进程TEB中的pid为lsass的PID。
3. 将LOGON_NETCREDENTIALS_ONLY指定为dwLogonFlags参数；设置dwFlags为STARTF_USESTDHANDLES；设置hStdInput、hStdOutput、hStdError为lsass打开的句柄。
4. 调用CreateProcessWithLogonW
5. 检查新进程是否是lsass的handle，并复制handle。

![image-20221010153220319](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/27f5ce4f3cc805dccf5d4b9fe329f8aa.png)

方法二：

1. 修改当前进程TEB中的pid为lsass的PID。
2. 在任意文件A中设置OpLock锁（工具中默认license.rtf）。
3. 使用CreateProcessWithLogonW来打开文件A。
4. 通过GetOverlappedResult来捕获CreateProcessWithLogonW访问文件A时的OpLock事件。（此处解决了父进程欺骗打开lsass进程句柄的瞬时问题）
5. 找到seclogon服务PID。
6. 枚举进程句柄并找到seclogon打开的句柄。
7. 通过DuplicateHandle复制seclogon打开lsass的句柄。

![image-20221010153641085](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/a3fa9fcb4f6231b62318c827961f0a02.png)

二、针对lsass内存的监控，MalSeclogon通过调用NtCreateProcessEx来复制进程内存到一个新的进程。（前提是句柄具有PROCESS_CREATE_PROCESS权限，SectionHandle设置为0）

整体代码思维导图：

![image-20221010153749190](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/1ee8182fe5784ad203813bff2a5c82da.png)

### 小结

本节主要总结了在edr对关键进程LSASS的OpenProcess和MiniDumpWriteDump的保护下，如何绕过保护dump内存的方法。

![image-20221010113251335](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/cb043e43a237a02d325b17a279d7f6fe.png)

## 五、防护

针对lsass内存dump的重要防护手段是微软的PPL(Protected Process Light)，当开启lsass的PPL之后，没有特定微软签名的二进制文件将无法访问lsass（上方所有的方法都无法获取lsass的句柄）。但是打开lsass的ppl后，不但恶意程序无法获取lsass句柄，同时非微软签名的程序也无法获取lsass进程句柄，有可能会影响正常业务。

## 六、参考文章
https://googleprojectzero.blogspot.com/2016/03/exploiting-leaked-thread-handle.html
https://skelsec.medium.com/duping-av-with-handles-537ef985eb03
https://splintercod3.blogspot.com/p/the-hidden-side-of-seclogon-part-2.html
https://splintercod3.blogspot.com/p/the-hidden-side-of-seclogon-part-3.html
https://skelsec.medium.com/duping-av-with-handles-537ef985eb03
https://www.matteomalvica.com/blog/2019/12/02/win-defender-atp-cred-bypass/
https://googleprojectzero.blogspot.com/2018/08/windows-exploitation-tricks-exploiting.html