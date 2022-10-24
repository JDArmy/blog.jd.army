---
title: codeql-sql
tags: [Java,代码审计,codeql]
categories: 渗透基础
date: 2022-10-24 14:26:16
---

###  前言


为什么学习CodeQL呢？在学习了一段代码审计，逐渐感觉代码审计是个体力活。而且越大的项目想要较全面的审计起来更是耗时间，还有可能漏掉一些很容易发现的漏洞。而CodeQL就是用来辅助漏洞挖掘，半自动化挖掘+人工辅助审计可大大减少人工成本，也提高了漏洞准确率。随着近几年网上公开的越来越多的严重级漏洞都是通过CodeQL挖掘出来的，所以目前对想学代码审计的人来说，学习CodeQL利大于弊，其目前也渐渐成为国内半自动化代码审计所使用的主流工具了。

###  安装及环境配置


####  **CodeQL安装**


CodeQL本身包含两部分解析引擎+`SDK`。

解析引擎用来解析我们编写的规则，虽然不开源，但是我们可以直接在官网下载二进制文件直接使用。

`SDK`完全开源，里面包含大部分现成的漏洞规则，我们也可以利用其编写自定义规则。

#####  **引擎安装**


下载地址：https://github.com/github/codeql-cli-binaries/releases解压文件夹，复制codeql文件夹到CodeQL(新建的)文件夹下

配置环境变量指向codeql.exe目录即可

![image-20221015190628828](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/0efc4de6fcbae435a5c58e7316b0cacd.png )

配置成功

![image-20221015190727439](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/556b8155b1c6f18acff41d0acd89dbac.png )

#####  SDK安装


移动到CodeQL目录下

```
git clone https://github.com/Semmle/ql
```

安装成功后CodeQL目录下就有两个文件夹codeql和ql

####  **CodeQL插件安装**


在官网下载并安装 Visual Studio Code，并安装CodeQL插件

![image-20221015183528507](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/4deaf44244959452082cac701b0a6668.png )

配置引擎路径

![image-20221015191053486](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/09ba56eb9c353de9f56388acec73969c.png )

到此就完全配置好了CodeQL开发环境了

###  CodeQL测试


靶场环境：https://github.com/l4yn3/micro_service_seclab/（其他也可)

先测试靶场环境是否可以正常调试，先测试下"Hello World"

![image-20221015200657345](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/e8c2843305184245cdc90c3ae761334b.png )

由于`CodeQL`的处理对象并不是源码本身，而是中间生成的AST结构数据库，所以我们先需要把我们的项目源码转换成`CodeQL`能够识别的`CodeDatabase`

```
codeql database create testdemo --language="java" --command="mvn clean install --file pom.xml" --source-root=D:/codeql/testdemo/micro_service_seclab-main/
```

![image-20221015200824228](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/13ee9338f057f96650fa2a8cc9f8397c.png )

成功如下图所示

![image-20221015200854746](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/8686e9a255a0990ce98f38dcd90fa3cb.png )

基础语法解释

```
database create testdemo 指我们要创建的database为testdemo(注:要先创建databases目录)
--language="java" 表示当前程序语言为Java
--command="mvn clean install --file pom.xml" 编译命令（因为Java是编译语言，所以需要使用–command命令先对项目进行编译，再进行转换，python和php这样的脚本语言不需要此命令）
--source-root=D:/codeql/testdemo/micro_service_seclab-main/ 指的是项目路径
```

导入database，选择testdemo文件夹

![image-20221015201136276](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/4833f6231e29d4c0b6f0aaf614a31ca3.png )

导入成功

![image-20221015201217525](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/ea87501155464b383a55337e909902ef.png )

编写查询打开刚才下载的SDK，在ql一一>java一一>ql一一>examples目录下创建demo.ql

![image-20221015201319764](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/453f01ab5da0fef17efa7e27919fbc48.png )

编写好查询语句，右击执行Run Query

![image-20221015201523865](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/39aafd2b93496c866c6b16cce644b13c.png )

出现如下右侧结果说明调试成功

![image-20221015201613098](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/8815a72e9848ae13cca73e26b2a84021.png )

###  CodeQL语法


参考文档：https://codeql.github.com/docs

因为CodeQL是识别不了源码本身的，而是通过CodeQL引擎把源码转换成CodeQL可识别的AST结构数据库，所以想要真正理解CodeQL原理，要学会看懂分析AST抽象语法树。

![image-20221016185328223](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/c796a6b5cdf65db052bfa0a50b7e81d7.png )

选中某个源码文件

![image-20221016185411908](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/3fdabc0049e9736b46e3567db84794b8.png )

点击View AST

![image-20221016185443980](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/4565c3839ab7b52d64517152a4e21a75.png )

通过语法树来学习CodeQL语法，可以更好的理解接下来怎么编写规则，为啥这么编写规则

![image-20221016185541857](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/821b094794c199d1d2f0b8edc731f001.png )

当然了，ql自身也提供了许多每种语言的Demo在snippets目录下供我们学习参考

![image-20221016185943629](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/96e65acab3d77e7fbc48245562148b1f.png )

**接下看开始真正的CodeQL学习之旅！！！**

先看一个小Demo

```
from /* ... variable declarations ... */
where /* ... logical formula ... */
select /* ... expressions ... */
```

```
import java
  
from int i
where i = 1
select i
```

第一行表示我们要引入CodeQL的类库，因为我们分析的项目是java的，所以在ql语句里，必不可少。

`from int i`, 定义一个变量i，它的类型是int，获取所有的int类型的数据

`where i = 1`, 当i等于1的时候，符合条件

`select i`, 输出i

总结：获取项目中所有整形变量，当变量的值等于1时，输出这个变量。

####  **类库**


ql中我们常用到的类库

|     名称     |                             解释                             |
| :----------: | :----------------------------------------------------------: |
|    Method    |      方法类，Method method表示获取当前项目中所有的方法       |
| MethodAccess | 方法调用类，MethodAccess call表示获取当前项目当中的所有被调用的方法 |
|  Parameter   |       参数类，Parameter表示获取当前项目当中所有的参数        |

怎么理解呢？通过AST语法树来理解。

Method method表示获取当前项目中所有的方法

![image-20221016191159214](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/26fdd69905cf0cffe702b67178083d2f.png )

MethodAccess call表示获取当前项目当中的所有被调用的方法

![image-20221016191229498](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/e2dbe24d3303304ea75ef7bcf29d4a89.png )

Parameter表示获取当前项目当中所有的参数

![image-20221016191345693](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/52e7c268625a991e4924cadda0ac96d9.png )

通过demo更好的理解一下

![image-20221016191737709](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/66134a5f1bed3bee8d91288c67b23de3.png )

####  **谓词**


其实就是常说的"函数"

```
predicate isSmall(int i) {
  i in [1 .. 9]
}
  
int getSuccessor(int i) {
  result = i + 1 and
  i in [1 .. 9]
}
```

predicate 表示当前方法没有返回值。

result是CodeQL引入的特殊变量，代表返回的变量

####  重点


如何进行全局污点追踪呢？

通过继承类`DataFlow::Configuration`使用全局数据流库

```
class SqlInjectionConfiguration extends DataFlow::Configuration {
    MyDataFlowConfiguration() { this = "SqlInjectionConfiguration" }
  
    override predicate isSource(DataFlow::Node source) {
    	...
    }
  
    override predicate isSink(DataFlow::Node sink) {
    	...
    }
}
```

下面是关于`DataFlow::Configuration`谓词的介绍

`isSource`-定义数据可能来源(输入点)。比如获取http请求的参数部分，就是非常明显的source。

`isSink`-定义数据可能流向的位置(执行点)。比如SQL注入漏洞，最终执行SQL语句的函数就是sink(这个函数可能叫query或者exeSql，或者其它)

![image-20221016214308470](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/2006e25264778f90934b6f15416058f4.png )

`isSanitizer`—可选，限制数据流(净化点)，代表污点传播到这里就会被阻断。比如SQL注入漏洞，虽然source经过多个Node可到达sink，但是数据类型是Int类型又或者其他原因使得漏洞不存在，则可以通过重写净化函数进行阻断，降低漏洞**误报率**。

![image-20221016214329124](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/febc450f02ae97ae630fcd0c89f78d43.png )

`isAdditionalFlowStep`—可选，添加额外的污点步骤。比如SQL注入Node1到Node2过程，由于QL本身规则没识别出该漏洞，而我们人工审核出是存在漏洞的，则可重写规则强制把Node1与Node2拼接起来，降低漏洞**漏报率**。

![image-20221016214353983](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/04655e1c85b41ea5c3d4081ad66fdd78.png )

####  exists公式讲解


这里有必要讲下`exists`公式，引入一些临时的变量。如果变量可以采用至少一组值来使正文中的公式为真，则它成立。

exists子查询，是CodeQL谓词语法里非常常见的语法结构，它根据内部的子查询返回true or false，来决定筛选出哪些数据。

![image-20221016225053300](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/3bc0b696842edf7b186afcabdbbedc93.png )

####  获取source


靶场环境使用的是`Spring Boot`框架，可以根据spring控制器的特点，注解中都存在XXXMapping来进行路径映射。怎么获取到注解呢？通过AST语法树分析来获取。

![image-20221016222416879](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/3c1e48368b784b15c3e70de01cd59d02.png )

![image-20221016222725557](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/57b425e03b358eb8fd47ea2e22e73107.png )

CodeQL代码实现

```
import java
  
from Method method, string c
where method.getAnAnnotation().toString() = c + "Mapping"
select method.getName() 
```

![image-20221016222809988](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/d0f26fe84e400d9fc072583ae1d8cb61.png )

这些方法的参数都可作为source，通过method.getParameter(n)获取Parameter

![image-20221016223250052](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/f42896ce434459d68d29ca63d9f31129.png )

所以我们设置Source的代码为：

```
override predicate isSource(DataFlow::Node src) { 
        exists(Method method, string c ,int n | 
            src.asParameter() = method.getParameter(n)
            and method.getAnAnnotation().toString() = c + "Mapping"
        )
    }
```

当然还有一种很更方便的获取方式，一行代码解决。那作者为啥要反其道而行呢？

①可以更好的理解分析AST语法树

②每种框架获取http请求参数不一，以下方法可能涵盖不到。当遇到很小众、很新的框架等原因，利用以下方式获取不到我们想要的参数该怎么办？就是利用以上最原始的方式分析语法树，代码自己的风格来获取。

③以下的方式不适合新手入门，可能理解不来。

```
override predicate isSource(DataFlow::Node src) { src instanceof RemoteFlowSource }
```

####  设置sink


本案例中，SQL语句最终执行点为query方法调用(MethodAccess)，且传入的参数是第一个。

![image-20221016224541235](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/0f0cdb39907654d48488db1deb1372ec.png )

所以我们设置Sink的代码为：

```
   override predicate isSink(DataFlow::Node sink) {
        exists(Method method, MethodAccess call |
            method.hasName("query")
            and call.getMethod() = method 
            and sink.asExpr() = call.getArgument(0)
        )
    }
```

意思为：查找一个query()方法的调用点，并把它的第一个参数设置为sink。

####  数据流


我们写好了source入口点和sink执行点，怎么实现一个污染参数通过头毫无拦截流到尾，成为可能真实存在的漏洞？这个连通工作就是CodeQL引擎本身来完成的。

代码样例：

```
from VulConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```

我们传递给`config.hasFlowPath(source, sink)`我们定义好的source和sink，系统就会自动帮我们判断是否存在漏洞了。

###  第一代成果


```
/**
 * @id java/examples/sqldemo
 * @name Sql-Injection
 * @description Sql-Injection
 * @kind path-problem
 * @problem.severity warning
 */
  
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.security.QueryInjection
import DataFlow::PathGraph
  
class SqlInjectionConfig extends TaintTracking::Configuration {
    SqlInjectionConfig() { 
        this = "SqlInjectionConfig" 
    }
  
    override predicate isSource(DataFlow::Node src) {  
        exists(Method method, string c ,int n | 
            src.asParameter() = method.getParameter(n)
            and method.getAnAnnotation().toString() = c + "Mapping"
        )
    }
  
    override predicate isSink(DataFlow::Node sink) {
        exists(Method method, MethodAccess call |
            method.hasName("query")
            and call.getMethod() = method 
            and sink.asExpr() = call.getArgument(0)
        )
    }
}
  
from SqlInjectionConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```

成功挖掘出SQL注入漏洞

![image-20221016230242905](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/ce3b0ee52bb23e26ffd33e59bd88253a.png )

注：上面的注释不能够删除，它是程序的一部分，因为在我们生成测试报告的时候，上面注释当中的name，description等信息会写入到审计报告中[Metadata for CodeQL queries — CodeQL (github.com)](https://codeql.github.com/docs/writing-codeql-queries/metadata-for-codeql-queries/ )

![image-20221016230706996](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/34ff34bb7ed69ba3a841f6f9492aa9e4.png )

###  漏报排除


![image-20221017084946974](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/4b6b558de349ee91da2bd2885c35427f.png )

![image-20221017085012253](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/d3e22ede19955c41a2e4635627f0890b.png )

人工审核很容易发现这是MyBatis注解式写法因为<img src="https://latex.codecogs.com/gif.latex?造成的SQL注入，但是CodeQL如何发现它呢？误报往往可以通过人工进行排查，而漏报一但项目上线就有可能造成重大的经济损失等。所以我们要尽可能全方面涵盖进行挖掘。

老方法，依旧通过AST语法树来进行分析。

![image-20221017085521315](images&#x2F;image-20221017085521315.png%20)

之前我们是通过定位query方法来发现漏洞，但是不够全面。现在大部分项目都是采用Mybatis注解方式，所以需要一起包含进去。

①通过定位接口名称以及接口注解来定位Mapper

②获得到相对应的Mapper后，通过注解字段Select，Update等来定位注解式语法

③通过参数名，获取参数别名

④将`"/>{`+参数别名+`}`在注解式子中进行匹配，发现漏洞

CodeQL代码实现：

```
import java
  
from Interface i, string c, Method method,string name, int n
  
where i.getName() = c + "Mapper" and method.getAnAnnotation().toString() = "Select" and method.getAnAnnotation().getValue(name).toString().indexOf("${"+method.getParameter(n).getAnAnnotation().getValue(name).toString().replaceAll("\"", "")+"}") > 0
  
select method.getParameter(n).getAnAnnotation().getValue(name).toString()
```

###  第二代成果


```
/**
 * @id java/examples/sqldemo
 * @name Sql-Injection
 * @description Sql-Injection
 * @kind path-problem
 * @problem.severity warning
 */
  
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.security.QueryInjection
import DataFlow::PathGraph
  
class SqlInjectionConfig extends TaintTracking::Configuration {
    SqlInjectionConfig() { 
        this = "SqlInjectionConfig" 
    }
  
    override predicate isSource(DataFlow::Node src) { 
        exists(Method method, string c ,int n | 
            src.asParameter() = method.getParameter(n)
            and method.getAnAnnotation().toString() = c + "Mapping"
        )
    }
  
    override predicate isSink(DataFlow::Node sink) {
        exists(|myBatisSink(sink) or querySink(sink))
    }
}
  
predicate myBatisSink(DataFlow::Node sink) {
    exists(Method method, MethodAccess call, Interface interface, string name, int n |
        interface.getAnAnnotation().toString() = "Mapper" 
        and method.getAnAnnotation().toString() in ["Select", "Update", "Insert", "Delete"]
        and call.getMethod() = method
        and method.getAnAnnotation().getValue(name).toString().indexOf("${"+method.getParameter(n).getAnAnnotation().getValue(name).toString().replaceAll("\"", "")+"}") > 0
        and sink.asExpr() = call.getArgument(0)
    )
}
  
predicate querySink(DataFlow::Node sink) {
    exists(Method method, MethodAccess call |
        method.hasName("query") 
        and call.getMethod() = method 
        and sink.asExpr() = call.getArgument(0)
    )
}
  
from SqlInjectionConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```

![image-20221017090534705](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/9f1f1468038badd287f44c65940bf1cf.png )

成功挖掘出MyBatis注解类型SQL注入漏洞。当然靶场中还存在其他各种各样的漏报，比如**Lombok**问题。

Lombok编写的项目，CodeQL在对项目解析时，会对CodeQL分析器造成干扰，导致所构建的数据库中少了很多源码。导致CodeQL分析不到Lombok带来的SQL注入问题。

![image-20221023160045939](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/f7f7ce91763fc797ed2cebbbe312be1b.png )

**解决方法**：

①使用maven-delombok，在pom.xml中添加以下代码，重新编译即可。（推荐）

```
<build>
	<sourceDirectory>${project.basedir}/src/main/lombok</sourceDirectory>
	<plugins>
		<plugin>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok-maven-plugin</artifactId>
			<version>1.18.20.0</version>
			<executions>
				<execution>
					<phase>generate-sources</phase>
					<goals>
						<goal>delombok</goal>
					</goals>
					<configuration>
						<encoding>UTF-8</encoding>
						<addOutputDirectory>false</addOutputDirectory>
						<sourceDirectory>src/main/java</sourceDirectory>
						<outputDirectory>${project.basedir}/src/main/lombok</outputDirectory>
					</configuration>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```

成功还原，并挖掘出Lombok插件带来的SQL注入风险。

![image-20221023160400883](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/6f32bcfd72b50ab453210f2845990d0c.png )

![image-20221023160713450](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/7d035457d0aa7179a002f962a310df16.png )

②官方也给出了解决方法https://github.com/github/codeql/issues/4984，这种还原方式有可能会出现未定义Object的场景。

```
# get a copy of lombok.jar
wget https://projectlombok.org/downloads/lombok.jar -O "lombok.jar"
# run "delombok" on the source files and write the generated files to a folder named "delombok"
java -jar "lombok.jar" delombok -n --onlyChanged . -d "delombok"
# remove "generated by" comments
find "delombok" -name '*.java' -exec sed '/Generated by delombok/d' -i '{}' ';'
# remove any left-over import statements
find "delombok" -name '*.java' -exec sed '/import lombok/d' -i '{}' ';'
# copy delombok'd files over the original ones
cp -r "delombok/." "./"
# remove the "delombok" folder
rm -rf "delombok"
```

###  误报排除


![image-20221016231250874](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/d9cc7e9cfe5f622c5e2fa710f402c889.png )

该方法的参数类型是List<Long>，不存在注入漏洞。所以我们需要用到上面所说的净化函数来进行阻断排除。

检测思路：如果当前Node节点的类型为基础类型，数字类型和泛型数字类型(比如List)时，就切断数据流。

![image-20221017091459850](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/3047223b42784f753ee5ff096fa81cf7.png )

![image-20221017091524060](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/724000043079b8c816c43c61761cac4a.png )

![image-20221017091542320](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/f4e136f09b98bb6050bce8bb401bc6be.png )

ParameterizedType：泛型类型的参数化

CodeQL代码实现：

```
override predicate isSanitizer(DataFlow::Node node) {
    node.getType() instanceof PrimitiveType or
    node.getType() instanceof BoxedType or
    node.getType() instanceof NumberType or
    exists(ParameterizedType pt| node.getType() = pt and pt.getTypeArgument(0) instanceof NumberType )
}
```

###  第三代成果


```
/**
 * @id java/examples/sqldemo
 * @name Sql-Injection
 * @description Sql-Injection
 * @kind path-problem
 * @problem.severity warning
 */
  
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.security.QueryInjection
import DataFlow::PathGraph
  
class SqlInjectionConfig extends TaintTracking::Configuration {
    SqlInjectionConfig() { 
        this = "SqlInjectionConfig" 
    }
  
    override predicate isSource(DataFlow::Node src) { 
        exists(Method method, string c ,int n | 
            src.asParameter() = method.getParameter(n)
            and method.getAnAnnotation().toString() = c + "Mapping"
        )
    }
  
    override predicate isSink(DataFlow::Node sink) {
        exists(|myBatisSink(sink) or querySink(sink))
    }
  
    override predicate isSanitizer(DataFlow::Node node) {
        node.getType() instanceof PrimitiveType or
        node.getType() instanceof BoxedType or
        node.getType() instanceof NumberType or
        exists(ParameterizedType pt| node.getType() = pt and pt.getTypeArgument(0) instanceof NumberType )
      }
}
  
predicate myBatisSink(DataFlow::Node sink) {
    exists(Method method, MethodAccess call, Interface interface, string name, int n |
        interface.getAnAnnotation().toString() = "Mapper" 
        and method.getAnAnnotation().toString() in ["Select", "Update", "Insert", "Delete"]
        and call.getMethod() = method
        and method.getAnAnnotation().getValue(name).toString().indexOf("${"+method.getParameter(n).getAnAnnotation().getValue(name).toString().replaceAll("\"", "")+"}") > 0
        and sink.asExpr() = call.getArgument(n)
    )
}
  
predicate querySink(DataFlow::Node sink) {
    exists(Method method, MethodAccess call |
        method.hasName("query") 
        and call.getMethod() = method 
        and sink.asExpr() = call.getArgument(0)
    )
}
  
from SqlInjectionConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```

![image-20221017091753741](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/8f64002b2289c7282c5ff7e6706594e5.png )

成功排除，当然有时候还有其他因素，比如开发写的过滤函数，白名单检测等排除。需要大家自己开动脑筋！

###  MyBatis-XML问题


这个问题CodeQL官网其实有提供相应特殊规则[MyBatisMapperXmlSqlInjection.ql](https://github.com/github/codeql/blob/main/java/ql/src/experimental/Security/CWE/CWE-089/MyBatisMapperXmlSqlInjection.ql )，该ql文件所做的事情是扫描Mapper配置Mybatis XML的${}的SQL注入。当然了包括上面所说的MyBatis注解式写法，官方也有提供相应的规则方便我们使用。

###  第四代成果


```
/**
 * @id java/examples/SqlInject
 * @name Sql-Injection
 * @description Sql-Injection
 * @kind path-problem
 * @problem.severity error
 */
  
import java
import semmle.code.java.dataflow.FlowSources
import MyBatisMapperXmlSqlInjectionLib
import MyBatisCommonLib
import MyBatisAnnotationSqlInjectionLib
import DataFlow::PathGraph
  
class SqlInjectionConfiguration extends TaintTracking::Configuration {
    SqlInjectionConfiguration() { 
        this = "SqlInjectionConfiguration" 
    }
  
    override predicate isSource(DataFlow::Node src) {  
        src instanceof RemoteFlowSource
    }
  
    override predicate isSink(DataFlow::Node sink) {
        exists(|
            sink instanceof MyBatisAnnotatedMethodCallArgument or
            sink instanceof MyBatisMapperMethodCallAnArgument or 
            querySink(sink)
        )
    }
  
    override predicate isSanitizer(DataFlow::Node node) {
        node.getType() instanceof PrimitiveType or
        node.getType() instanceof BoxedType or
        node.getType() instanceof NumberType or
        exists(ParameterizedType pt| node.getType() = pt and pt.getTypeArgument(0) instanceof NumberType )
      }
  
    override predicate isAdditionalTaintStep(DataFlow::Node node1, DataFlow::Node node2) {
        exists(MethodAccess ma |
            ma.getMethod().getDeclaringType() instanceof TypeObject and
            ma.getMethod().getName() = "toString" and
            ma.getQualifier() = node1.asExpr() and
            ma = node2.asExpr()
        )
    }
}
  
predicate querySink(DataFlow::Node sink) {
    exists(Method method, MethodAccess call |
        method.hasName("query") 
        and call.getMethod() = method 
        and sink.asExpr() = call.getAnArgument()
    )
}
  
from SqlInjectionConfiguration config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```

成功扫出MyBatis注解式和MyBatis-XML存在的SQL注入问题

![image-20221023192235185](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/37616165c204b2785aeb9996470c0f02.png )

###  总结


相信通过上面的学习，大家也都发现了所有的分析都是离不开AST语法树的。因为CodeQL自身只能识别AST语法树，所以要进行任何的分析都需要围绕AST语法树来进行分析，也可以更好的理解QL语法原理，对自己的分析大有益处。当作者自己第一次看别人的分析文章看着CodeQL代码半天一筹莫展，丝毫没理解懂。当知道QL的核心就是AST语法树，再结合着语法树进行分析就很好理解了。也会同时有更多自己不一样的分析思路。

当然了，通过CodeQL不可能挖掘出所有的漏洞。而且不同的源码需要不同的规则进行匹配。所以需要大家自己探索不断完善自己的规则，它的好处就是可以大大减少人工成本，在较通用的漏洞上帮助大家更好的发现。当然网上也有很多非常隐蔽的漏洞也是通过CodeQL进行挖掘的，这就需要大家自己独特的挖掘规则思路。

###  参考文章


https://www.freebuf.com/articles/web/283795.html