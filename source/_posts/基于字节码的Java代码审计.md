---
title: 基于字节码的Java代码审计
tags: [Java,代码审计]
categories: 渗透基础
date: 2022-02-21 14:26:16
---

之前看了基于字节码的`Java`代码审计工具的实现，最近终于有空可以好好看一下其是如何实现的了。本文并不会从代码出发，而是试图从字节码角度分析其可行性。
<!-- more -->

## JVM简介

要了解字节码首先需要对JVM有所了解，Java虚拟机并不关心Java语言，它只和字节码相关联，这一方面使得Java程序可以`Run AnyWhere`，另一方面也为其运行其他语言提供了支持 -- 只要编译成为符合字节码规范的内容，均可以在Java虚拟机中运行。

与Java类似，Java虚拟机可以操纵原始类型、引用类型两种操作类型，与之对应的是原始值以及引用值。

原始类型包括：

- 数值类型
- Boolean类型
- returnAddress类型 指向某个操作码（opcode）的指针

数值的一些取值范围

![image-20220210142623710](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/1581643921a8fe22a08be5c34de0895a.png)

在运行时会有如下运行时数据区：

![image-20220126154212655](https://firewore.oss-cn-beijing.aliyuncs.com/imgs/image-20220126154212655.png)

作用分别如下：

### 程序计数器

因为JVM的多线程是通过线程轮流切换实现的，在任何时候，Java虚拟机的一个内核只会处理一个线程，因此为了切换线程后可以记录当前执行位置需要把这个地址记录下来。如果执行的是Java代码，则这里记录的是字节码指令地址，如果是native方法的话则为null

### 虚拟机栈

每个方法被创建时，其都会创建一个栈帧，里面保存着局部变量表、操作数栈、动态链接等信息。在一个时间点上，只有当前执行方法的栈帧处于活动状态。及当前栈帧，该方法称为当前方法，方法所在类为当前类。

局部变量表中存放虚拟机支持的数据类型，除去`long`,`double`占两个位置外，其余类型均占一个位置。它可以根据索引进行获取，在非static方法中，0位往往表示类本身。

操作数栈中数据往往从局部变量表中获取，在进行方法调用前会进行出栈，作为被调用函数的局部变量。如果存在返回值，则返回值会入栈至调用函数的操作数栈中。

### 本地方法栈

其作用类似虚拟机栈、只不过其作用对象为Native服务。

### Java堆

大部分Java对象实例以及数组在Java堆当中。

### 方法区

存储已被虚拟机加载 的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。

### 运行时常量池

存放编译期生成的字面量与符号引用，栈帧中存在有一个指向当前方法所在类型的运行时常量池的引用，一个方法若是想调用其他方法，或者访问成员变量时要用符号引用表示，此时会使用动态链接将符号引用转换为直接引用。

我们重点关注方法执行时的情况，也就是虚拟机栈，首先创建下面的类

```java
public class NewWorld{
	public int say(int a){
		return a;
	}

	public void world(){
		int b = say(1);
		say(b);
	}
}
```

编译后使用`javap`查看其栈帧操作

```bash
javap -c -verbose NewWorld
......
{
  public NewWorld();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public int say(int);
    descriptor: (I)I
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=2, args_size=2
         0: iload_1
         1: ireturn
      LineNumberTable:
        line 3: 0

  public void world();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=1
         0: aload_0
         1: iconst_1
         2: invokevirtual #2                  // Method say:(I)I
         5: istore_1
         6: aload_0
         7: iload_1
         8: invokevirtual #2                  // Method say:(I)I
        11: pop
        12: return
      LineNumberTable:
        line 7: 0
        line 8: 6
        line 9: 12
}
......
```

先来看`world`方法

```bash
0: aload_0
1: iconst_1
```

`aload`表示从局部变量表中获取一个变量并压入操作数栈中，`aload_0`表示获取第`0`个变量，在非`static`方法中，这就意味着`this`或者说，当前类。

`iconst`表示压栈一个数值类型的常量

```bash
2: invokevirtual #2                  // Method say:(I)I
```

`#2`表示链接到了运行时常量池中序号为2的位置，`javap`其实也有输出这部分内容：

```bash
Constant pool:
   #1 = Methodref          #4.#14         // java/lang/Object."<init>":()V
   #2 = Methodref          #3.#15         // NewWorld.say:(I)I
   #3 = Class              #16            // NewWorld
   #4 = Class              #17            // java/lang/Object
```

`invokevirtual`表示调用一个虚函数，而符号引用表明这里调用的是`NewWorld.say`，虚函数是什么这里暂时不深入解释。进入`say`函数前，此时的操作数栈中有两个变量：

```bash
0: this
1: 1
```

进入`say`函数后，它的局部变量表就为：

```bash
0: this
1: 1
```

再看`say`函数的操作数

```bash
0: iload_1
1: ireturn
```

首先将局部变量表中的第一个参数压入操作数栈，这里就是`1`，之后返回栈顶的数值至调用函数`world`中，回到`world`:

```bash
5: istore_1
6: aload_0
7: iload_1
8: invokevirtual #2                  // Method say:(I)I
11: pop
12: return
```

此时栈中只有刚才的返回值`1`，将它赋值给局部变量表参数1后再次将局部变量表中前两个参数入栈并执行函数`say`，函数返回前调用`pop`清空操作栈。

至此字节码执行一个函数的过程就结束了，这里就可以进入下一个问题了

## 使用字节码进行审计

首先看这一段代码：

```java
public class RceYes{
    public void eval(String cmd){
        try {
            Runtime.getRuntime().exec(cmd);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
  
  public void act(String str){
    eval(str);
  }
}
```

如何判断这里存在漏洞呢？很显然，这里存在有危险函数`Runtime.getRuntime().exec`，而其参数`cmd`也是函数`eval`的入参，再往回看，发现`act`函数调用了`eval`，且为其提供的参数就是`act`的入参，只要`act`的参数可控，就可以达成`RCE`。据此我们可以简单得出一个规则：

**可控参数可以传播到危险函数即可认为其存在漏洞**。我们先来单独看下`eval`函数

```bash
  public void eval(java.lang.String);
    descriptor: (Ljava/lang/String;)V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=2
         0: invokestatic  #2                  // Method java/lang/Runtime.getRuntime:()Ljava/lang/Runtime;
         3: aload_1
         4: invokevirtual #3                  // Method java/lang/Runtime.exec:(Ljava/lang/String;)Ljava/lang/Process;
		......
```

`aload_1`从局部变量表中获取1号位的参数，也就是传入`eval`的参数，并供给`exec`调用，在上文的分析中其实我们已经知道被调用函数的局部变量表的值来自与调用函数的操作数栈，如下是`act`函数内容：

```bash
  public void act(java.lang.String);
    descriptor: (Ljava/lang/String;)V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: invokevirtual #6                  // Method eval:(Ljava/lang/String;)V
         5: return

```

在调用`eval`函数时会先把`this`跟局部变量表1号位参数入栈，至此不难看出，参数在不同函数之间的传递是有迹可循的 ，根据栈帧操作其实很容易就能判断出一个函数的参数是否会影响到函数体内的另一个函数

![image-20220211004935748](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220211004935748.png)

![image-20220211005037973](https://wiki-1258274765.cos.ap-nanjing.myqcloud.com/wiki/imgimage-20220211005037973.png)

这也就是说我们可以模拟参数在栈帧中传递，从而判断其是否可以到达危险函数位置，及可以通过模拟运行函数时过程进行代码审计。

## 实现

上文已经找到了字节码代码审计的关键，那接下来如何通过代码去实现呢？首先第一个问题，如何将`Class`文件解析成字节码指令的形式。这里就不得不提一下`ASM`了，作为字节码增强技术，它可以动态修改字节码或者是遍历类的结构，`ASM`此处就不进行深入展开了，我们单单看一下遍历类结构这一点，它会按照一定顺序逐语句对字节码进行解析，当解析到函数时我们就可以自定义一个局部变量表以及操作数栈来进行模拟操作，这一思路早在18年就已经被实现：[GadgetInspector](https://github.com/JackOfMostTrades/gadgetinspector) ，我们简单看下其这一部分的实现：

```java
    public void visitVarInsn(int opcode, int var) {
        // Extend local variable state to make sure we include the variable index
        for (int i = savedVariableState.localVars.size(); i <= var; i++) {
            savedVariableState.localVars.add(new HashSet<T>());
        }

        Set<T> saved0;
        switch(opcode) {
            case Opcodes.ILOAD:
            case Opcodes.FLOAD:
                push();
                break;
            case Opcodes.LLOAD:
            case Opcodes.DLOAD:
                push();
                push();
                break;
            case Opcodes.ALOAD:
                push(savedVariableState.localVars.get(var));
                break;
            case Opcodes.ISTORE:
            case Opcodes.FSTORE:
                pop();
                savedVariableState.localVars.set(var, new HashSet<T>());
                break;
            case Opcodes.DSTORE:
            case Opcodes.LSTORE:
                pop();
                pop();
                savedVariableState.localVars.set(var, new HashSet<T>());
                break;
            case Opcodes.ASTORE:
                saved0 = pop();
                savedVariableState.localVars.set(var, saved0);
                break;
            case Opcodes.RET:
                // No effect on stack
                break;
            default:
                throw new IllegalStateException("Unsupported opcode: " + opcode);
        }

        super.visitVarInsn(opcode, var);

        sanityCheck();
    }
```

`visitVarInsn`表示观察到有从局部变量表中获取或写入数据的操作码时进行的操作，`opcode`是具体操作数，`var`表示要操作第几个变量。它首先判断自定义的本地变量表大小是否不够`var`的调用，不够则进行增加，注意，这里增加的是空值，其实在`GadgetInspector`中，自定义的`LocalVar,stackVars`中并不会存真实的值。这里其实就是按照JVM规范实现了局部变量表与操作数栈间的数据流动。有了数据流动，再分析调用函数的传参时其操作数栈也能明确了。于是，第二个问题也来了，`ASM`是对单个`Class`或者`Method`进行分析的，也就是说当我们发现一个函数体中的被调函数可以被污染时，我们是无法进入被调函数进行分析的，所以我们需要提前做好一件事，找出所有方法调用，并将其逆序排列。这样子首先被分析的函数肯定是调用链的底层，当它的参数可以污染到危险函数时再判断其调用函数的参数是否可以污染它，至此完整的一条链路就浮现而出了。

## 总结与想法

总的来说先了解原理在看工具会发现容易理解了不少。除了上文提到了工具，有许多师傅依据与此发布了不少实用的工具，例如 [SpringInspector](https://github.com/4ra1n/SpringInspector),[ACAF](https://github.com/Er1cccc/ACAF)等。其实其核心在于构建出一个”数据库“，保存着方法与方法、方法与类之间的关系，然后通过查询找到符合条件的触发点。这不禁让人想起另一款工具 -- `CodeQL`，它在分析源代码时也是很实用了，那有没有一种方式可以把这两种方法联动起来呢？笔者因此诞生了一个略微有些不切实际的想法：为`GadgeInspector`实现一个`QL`解析器，使部分`CodeQL`的语句可以被解析运行，这样一方面不需要每次污点查询不同漏洞都对源码进行修改，另一方面不需要为两个工具准备不同的检测语句。想法是简单的，但是实现起来却碰到了难题：语义分析。笔者对编译原理所知了了，只能暂时放弃这个想法了~

## 参考文章

- https://xz.aliyun.com/t/7058
- https://4ra1n.love/post/zA8rsm1ne/#%E6%80%BB%E7%BB%93
- https://xz.aliyun.com/t/10756
- 《Java虚拟机规范》

