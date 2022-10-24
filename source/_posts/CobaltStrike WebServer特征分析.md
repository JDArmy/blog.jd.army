---
title: CobaltStrike WebServer特征分析
tags: [cs,C2,流量]
categories: 渗透工具
date: 2022-06-01 10:08:16
---

[toc]

# WebServer特征

本文简单介绍了Cobalt Stike 4.4版本的一些特征以及缓解措施。



## 证书特征以及流量特征

三种方式都可以，一定要改。

```
https-certificate {
    
    ## Option 1) Trusted and Signed Certificate
    ## Use keytool to create a Java Keystore file. 
    ## Refer to https://www.cobaltstrike.com/help-malleable-c2#validssl
    ## or https://github.com/killswitch-GUI/CobaltStrike-ToolKit/blob/master/HTTPsC2DoneRight.sh
   
    ## Option 2) Create your own Self-Signed Certificate
    ## Use keytool to import your own self signed certificates

    set keystore "./cobaltstrike.store";
    set password "112233";

    ## Option 3) Cobalt Strike Self-Signed Certificate
    #set C   "CN";
    #set CN  "*.jd.com";
    #set O   "BEIJING JINGDONG SHANGKE INFORMATION TECHNOLOGY CO., LTD.";
    #set OU  "Mobile";
    #set ST  "beijing";
    #set L   "beijing";
    #set validity "365";
}
```

证书修改之后，数据包也要改。



## webserver处理逻辑漏洞

### 请求状态码异常

正常的服务器对于uri的开头不为/的情况，一般都会产生400的状态。

![image-20220526222654379](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/9725e6b1c895ec53b6efa38688a8b351.png)

而teamserver在处理的时候，返回了404，

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/961c5c717bb2571e7d69a8dd5354915d.png)

在处理OPTIONS请求时候，更是uri都不看，直接返回200，并且在后面会加上Allow: OPTIONS,GET,HEAD,POST

![image-20220527125856996](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/fcfcc669c7476c0f7f709f33dbc330fc.png)

正常的网站感觉还是干不出来这种事情的。

![image-20220527130106960](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/ea4501d24315fecaee493db80d3aa719.png)



### beacon&stager uri异常访问

对于更特殊的请求，类似的情况如下，更是直接暴露了profile的http配置以及beacon。

profile的流量情况：

![image-20220527130449921](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/57a30b1c286fd0f6661c9311c7ca01b3.png)

beacon：

![image-20220526184239394](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/c26f1b02f46e0df71b8e2b78bfb4031b.png)

### uri匹配问题

可以看到对于beacon.http-get、beacon.http-post的uri后面可以随意增加的，profile很灵活导致了webserver没办法做精准的匹配。

![image-20220527153113217](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/569f67139e21c44fc6cb5738f7c30565.png)

## checksum8特征

checksum8算法可以匹配多个值，如aaa9等，个人理解本意是在profile没有设置uri的时候，具有一定的随机性，多个uri都可以获取beacon，但是问题在于profile中 set uri之后，该算法依然可用（可能为了兼容msf的请求），配合默认解密方法可以获取完整的配置。

![image-20220527161242273](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/41b64e2d48db35c8fa1d6315d3cca5fc.png)

# WebServer流程、特征分析

## UA校验

我们对核心逻辑_serve进行简单的分析。可以看到先经过了一个UA的黑白名单，可以在profile中进行配置。默认黑名单有curl* lynx* wget*。

![image-20220527132117861](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/d6b4316c2b29be019cdb41c178a26154.png)如果不符合UA检测，则返回404，并在console中输出。

![image-20220527132432748](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/d08761189988e125a483b0241ad38b24.png)

## 处理OPTIONS请求

这块发现teamserver是没有对uri做校验的，直接返回200，并添加了一个Allow的header。

![image-20220527125758640](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/64d75e5f873232f9e7ebab5aabf59974.png)



## webserver核心逻辑

Webserver封装了一个名字叫hook的Map，里面push了多个WebService的实现，Map的key为uri，在监听创建的时候，默认会push上述4个WebService进去。

```
beacon.http-get
stager
stager64
beacon.http-post
```

其中beacon\*类型为MalleableHOOK，负责处理beacon的通讯，如心跳，命令执行等，stager\*类型为MalleableStager，负责推送beacon。

当通过UA检测后区分了7种情况：

```
1.匹配响应一般的uri，用于host file，powershell script等一些情况 （beacon&stager uri异常访问的原因）
2.匹配响应用户配置的beacon.post/get,stager/stagerx64的请求
3.匹配响应目录的uri，自动补全结尾的/，没有找到场景
4.匹配以http://开始的uri，没有找到场景
5.匹配stager64的uri，主要用于响应64位场景下checksum8算法生成的uri（uri长度有限制，checksum8特征的原因）
6.匹配stager的uri，主要用于响应32位场景下checksum8算法生成的uri（uri长度有限制，checksum8特征的原因）
7.所有条件轮空的处理
```

当第匹配到uri为hooks的key时，就会返回对应的响应，也就产生了beacon&stager uri异常访问的问题。

![image-20220527132811915](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/c04eb46fca32f74113804f4f3cfa15c7.png)

当所有条件轮空时，也就是第7种情况，会再次通过checksum8算法匹配uri是否返回beacon的响应，与上文相比，去掉了uri长度的限制。此外，也会判断是否stager关闭导致异常。如果i遍历完成，返回404，由于对uri有特殊的情况，没有判断uri是否需要以/开头。因此产生了一系列的特征。

![image-20220527151319454](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/d82b6dc35f4de10fb9c3ff9900dbbd34.png)

这里比较有趣的是，while的条件是startsWith与isFuzzy判断，通过对WebService所有实现类进行分析。MalleableHook的isFuzzy为true。也就是说WebServer对于beacon的交互的uri在后面随便加东西都可以匹配、响应profile配置。个人感觉这也算是teamserver的特征吧。

# 特征修改

主要处理了/的问题和checksum8的问题，其他问题暂时不处理了，头大。

## webserver处理逻辑漏洞

请求状态码异常、beacon&stager uri异常访问都是由于没有校验/的问题导致的，由于我使用的是javaagent，对于大段的代码修改比较麻烦，我选择在WebServer中serve进行修改。增加了一个/的检验，不过http://开头的请求可能会收到影响，目前还清楚是什么功能，还需要进一步测试一下。

![image-20220526225223369](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/047f0fe37839421a2911a22c39259daf.png)

## checksum8特征

checksum8特征有很多缓解的方法。

```
1.修改checkSum8的92L与93L为非默认的值（可破解）
2.更换算法（成本较高）
3.固定URI（容易形成新的特征）
4.kill stager（依赖客户端操作）
5.设置host_stage（无法使用stager）
```

我同样在serve函数中进行了patch，废掉了checksum的匹配，缺点是必须配置profile的，可能也会有其他的问题，待测试。

```
    set uri_x86 "xx";
    set uri_x64 "xx";
```

![image-20220527164717418](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/462be9b76dd16cc3d644c80b50bb94b6.png)

## javaagent实现

```
             else if(className.equals("cloudstrike/WebServer")){
                    ClassPool classPool = ClassPool.getDefault();
                    CtClass cls = classPool.get(className.replace("/","."));
                    CtMethod readResource = cls.getDeclaredMethod("serve");
                    readResource.setBody(
                            "{"+
                            "   if(!$1.startsWith(\"/\")) {" +
                            "       return $0.processResponse($1, $2, $3, $4, false, null, new cloudstrike.Response(\"400 Bad Request\", \"text/plain\", \"\"));" +
                            "   }" +
                            "   if(isStager($1)||isStagerX64($1)){" +
                            "       return $0.processResponse($1, $2, $3, $4, false, null, new cloudstrike.Response(\"404 Not Found\", \"text/plain\", \"\"));" +
                            "   }" +
                            "   return handleRanges($2, $3, $0._serve($1, $2, $3, $4));" +
                            "}");
                    return cls.toBytecode();
                }
```

## 效果

stager不会带出了

![image-20220526225335625](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/ae1fd72922e080855578ca31395d2b63.png)

异常的404也顺带解决了

![image-20220526225304337](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/31c421c33b8335f996341f4737396ad9.png)

aaa9已经无法请求

![image-20220527165602354](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/05/3beadd27511f0f51a3d45128c66b3c7c.png)

# 其他特征

本文主要分析了webserver的几个特征，内存特征就不再这里提了，javaagent也是可以缓解的。4.5虽然对javaagent做了检查，但是依然很好绕过。4.6貌似server和client分开了，可能就比较麻烦了。

参考

```
https://cloud.tencent.com/developer/article/1967094?from=article.detail.1818330
https://github.com/Like0x/0xagent/blob/main/PreMain.java
http://cn-sec.com/archives/300922.html checksum8

```