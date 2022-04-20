---
title: 漏洞风险 | Spring4Shell通用WAF策略存在绕过风险，需尽快升级！
tags: [Spring,漏洞]
categories: 漏洞预警
date: 2022-04-01 10:08:16
---

经团队研究发现，所有针对 Spring4Shell 漏洞的基于关键字的WAF防护策略均存在被绕过风险。
<!-- more -->
![](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/20220420175857.png)

近日爆发的Spring-Core RCE漏洞，又称Spring4Shell漏洞。经团队研究发现，所有针对 Spring4Shell 漏洞的基于关键字的WAF防护策略均存在被绕过风险。现官方补丁已发，仅通过WAF防御尚未进行版本升级的相关公司单位请尽快升级！请勿过度依赖WAF！


注：这不是愚人节玩笑！



附：参考链接：

https://spring.io/blog/2022/03/31/spring-framework-rce-early-announcement

https://github.com/spring-projects/spring-framework/compare/v5.3.17...v5.3.18



相关绕过细节将在未来漏洞完全修复后，视情况进行负责任批漏，请持续关注本公众号。