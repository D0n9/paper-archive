# 从SpringInspector源码视角深入浅出静态代码分析技术
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/06596228-0c1a-4ae5-8ca3-059385f66f63.gif?raw=true)
戳上面的**蓝字**关注我吧！

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/b94bdc08-9468-48b8-b275-fc32b6356c7e.gif?raw=true)

* * *

本篇文章10549个词，预计阅读时长30分钟。

假设本文的读者已经有相关SpringBoot、字节码开发经验。

* * *

**0****1**

**分析原理**

市面上目前的主流白盒检测引擎估计就分为两者（之前是由正则和AST语法分析占据的），一种是基于字节码的堆栈模拟检测，另一种就是通过CPG分析技术来检测产出漏洞。而我们介绍的SpringInspector就是基于字节码的检测，所以本篇主要还是围绕的字节码检测原理来说，不过笔者会在末尾加上CPG的分析原理介绍。

SpringInspector工具项目地址：https://github.com/4ra1n/SpringInspector

工具是通过读取SpringBoot的Jar包来分析其中的字节码

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/e212378c-e23a-43e9-b864-fb7f2f93d47d.png?raw=true)

大致的流程作者也在分析文章中讲过Reference\[1\]

1.  解压用户提供的Jar包
    
2.  解压后BOOT-INF/classes为项目代码，创建InputStream
    
3.  解压后BOOT-INF/lib为项目依赖库，包含各种普通Jar包
    
4.  将依赖库全部普通Jar包解压，为其中所有class文件创建InputStream
    
5.  根据String类找到rj.jar利用Guava的库得到JDK所有class文件，创建InputStream
    

先来看看程序的start方法中都有哪些启动和初始化相关的

```java
private static void start(Command command) {
    getClassFileList(command);
    discovery();
    inherit();
    methodCall();
    sort(command);
    buildCallGraphs(command);
    parseSpring(command);
    if (command.module == null || command.module.equals("")) {
        logger.warn("no module selected");
    } else {
        String module = command.module.toUpperCase(Locale.ROOT);
        if (module.contains("ALL")) {
            module = "SSRF|SQLI|XXE|RCE|DOS|REDIRECT|LOG";
        }
        if (module.contains("SSRF")) {
            SSRFService.start(classFileByName, controllers, graphCallMap);
            resultInfos.addAll(SSRFService.getResults());
        }
        if (module.contains("SQLI")) {
            SqlInjectService.start(classFileByName, controllers, graphCallMap);
            resultInfos.addAll(SqlInjectService.getResults());
        }
        if (module.contains("XXE")) {
            XXEService.start(classFileByName, controllers, graphCallMap);
            resultInfos.addAll(XXEService.getResults());
        }
        if (module.contains("RCE")) {
            RCEService.start(classFileByName, controllers, graphCallMap);
            resultInfos.addAll(RCEService.getResults());
        }
        if (module.contains("DOS")) {
            DOSService.start(classFileByName, controllers, graphCallMap);
            resultInfos.addAll(DOSService.getResults());
        }
        if (module.contains("LOG")) {
            LogService.start(classFileByName, controllers, graphCallMap);
            resultInfos.addAll(LogService.getResults());
        }
        if (module.contains("REDIRECT")) {
            RedirectService.start(classFileByName, controllers);
            resultInfos.addAll(RedirectService.getResults());
        }
    }
    logger.info("total results: " + resultInfos.size());
    ResultOutput.write(command.path,resultInfos);
    logger.info("delete temp dirs...");
}
```

**0****2**

**污点分析介绍**

这里插入我之前发布的污点分析介绍的一张截图

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/e8532376-5abf-49eb-9f32-f139b930178b.png?raw=true)

总体来说，在Source点传入的参数被打上污点标签，再通过一系列的传播后，所进入到的Sink点方法还具有污点标签，则判断漏洞存在。

详细可以查看Reference\[2\]的污点分析介绍或查看Reference\[3\]的论文《污点分析技术的原理和实践应用》

而该程序主要是通过数据流分析（其污点分析本身就是一种数据流分析技术的变种，只不过关注污点传播的流向）结合CG(Call Graph)调用图来判断污点参数的走向。

这里扩展一下数据流分析的相关介绍，其数据流分析技术可以分为以下两大类：

*   根据程序路径的分析精度分类：
    

*   流不敏感分析(flow insensitive)：不考虑语句的先后顺序，按照程序的物理位置从上往下顺序分析每一语句，忽略程序中存在的分支。
    
*   流敏感分析(flow sensitive)：考虑程序语句可能执行的顺序，通常需要结合程序的控制流程图(CFG，Control-Flow Graph)。
    
*   路径敏感分析(path sensitive)：不仅考虑语句的先后顺序，还对程序执行的路径条件加以判断，以确定分析使用的语句序列是否对应着一条可实际运行的程序执行路径。
    

*   根据程序路径的深度分类：
    

*   上下文不敏感分析(context-insensitive)：将每个调用或返回看作一个“goto”，忽略调用位置和函数参数取值等函数调用的相关信息。
    
*   上下文敏感分析(context-sensitive)：对不同调用位置调用的同一函数加以区分。
    
*   过程内分析(intra-procedure analysis)：只针对程序中函数内的代码。
    
*   过程间分析(inter-procedure analysis)：考虑函数之间的数据流，即需要跟踪分析目标数据在函数之间的传递过程
    

再来看看数据流分析是如何分析漏洞的：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/6789565f-e754-4850-8022-510a9540ff8b.png?raw=true)

首先会对程序的源代码进行代码建模，这部分通常是有词法分析、语法分析等分析技术，并生成对应的AST抽象语法树、三地址码和类似控制流程图、调用图这种数据结构。之后会读取漏洞分析规则，并对数据流进行分析，分析其变量进行跟踪，并分析其性质、取值、状态等信息。最终产出分析结果。

**03**

**初始化过程**

先看到getClassFileList()方法将我们的命令参数传入了进去，跟进去之后可以发现调用了resolveSpringBootJarFile方法。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/97afb1fe-0bfc-429c-b813-9d094603ae0c.png?raw=true)

方法中读取了jar包中相关的类文件和依赖，输出到临时目录中后，会返回一个结果集。

再接着看start方法的discovery()方法

```cs
private static void discovery() {
    DiscoveryService.start(classFileList, discoveredClasses, discoveredMethods,
                           classMap, methodMap, classFileByName);
    logger.info("total classes: " + discoveredClasses.size());
    logger.info("total methods: " + discoveredMethods.size());
}
```

这里通过DiscoverService的start方法初始化了一系列的变量，其中最重要的是new了一个DiscoveryClassVisitor对象，该对象继承于ClassVisitor类。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/3c9be1a7-f4a8-4a6d-a34a-f1a2ea353cc1.png?raw=true)

看了下逻辑，基本上是读取临时文件中的类和方法等信息。

继续往下看，发现进入到了inherit方法中，该方法体中首先进入到了InheritanceUtil.derive()中

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/b5a7fe0b-6288-4a35-95ba-0688d2edbc93.png?raw=true)

derive对象会将所有的类继承关系遍历，并封装到InheritanceMap的对象中，该对象有两个map，分别用来通过父类找子类，和通过子类找父类的关系。  

之后又调用了InheritanceUtil.getAllMethodImplementations方法

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/f1de1699-3e97-4482-8a15-dec4b707d9c7.png?raw=true)

其中通过子类和父类的方法名和签名是否相同来判断子类是否重写了父类的方法。并将其添加到集合中。

接着继续往下看，程序来到methodCall()方法中，跟进查看

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/28541631-d155-4c7b-b052-cd83c000ac58.png?raw=true)

其中使用了自建的MethodCallMethodAdapter选择器，会把方法的属性存入到this.methodCalls的Map变量中。再通过JSRInlinerAdapter提高对低版本的兼容性。

之后程序会进入到sort方法中，这个方法会把之前的methodCalls的方法进行遍历逆拓扑排序。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/847e3f02-00b1-473e-b8b9-c348ff1b0860.png?raw=true)

**为何要用逆拓扑排序**

既然讲到了逆拓扑排序，就得仔细讲解一下为何要如此做，这么做的目的是什么？

至于为何要用逆拓扑排序，可以看如下一个XSS漏洞的例子

```kotlin
@RequestMapping("/reflection")
@ResponseBody
public String reflection(@RequestParam("data") String message) {
    return xssService.iftest(message);
}
```

iftest方法的具体实现如下：

```typescript
@Override
public String iftest(String message) {
    if (!message.equals("test")) {
        return message;
    }
    return "error";
}
```

由此可以知道方法调用过程是reflection(String) -> iftest(String)。

因此我们做静态代码分析的时候，肯定要先判断iftest的污点是否有返回或者被清洗掉的可能，因此代码分析的过程就是iftest() -> reflection()的一个反向的过程。但是方法调用过程是一组有向图，因此就需要用到逆拓扑排序算法来进行分析。

**逆拓扑排序具体实现逻辑**

首先了解一下什么是逆拓扑排序

我有如下一张图

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/9182bb08-279e-49f1-b1dc-dd7d3e6e1c00.png?raw=true)

而该图的逆拓扑排序步骤如下：

开辟一块栈空间，用来存放节点

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/0e25627b-6614-4573-b832-5005ba7d7d49.png?raw=true)

并将指针P指向第一个节点位置，也就是元素0。

元素0有两个向量，分别是元素1和2，我们从1开始遍历

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/c50d6aae-2bd7-4f86-bf03-40ecd87b2224.png?raw=true)

遍历元素之后指针P指向元素1，同时将之前的元素0压到栈中。此刻又以元素1为节点，继续遍历，发现向量指向元素3。  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/73f1f382-aa02-425a-b80a-3767094d6290.png?raw=true)

如此往复，将元素1也压入栈中。  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/3149c975-9536-4a75-86f3-9716911a57f1.png?raw=true)

但由于3并无向量，因此将3压栈后又出栈，并将指针P返回上一个节点。

同样的，元素1也并无向量了（向量3已经遍历过了），因此也将元素1出栈并返回上一个节点。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/32703dff-0616-4551-b4bf-b6560851ee4a.png?raw=true)

此时发现元素0还有向量2没有遍历，因此继续上述的步骤

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/3ea39f37-f85c-4f7b-ab31-57119844f9c2.png?raw=true)

最终出栈顺序为3 1 2 0，仔细查看就可以发现，出栈的顺序正好是DFS的遍历路径。

有了简单的介绍逆拓扑排序原理，再来看看Reference\[4\]的Longof师傅的文章的讲解，仔细分析下逆拓扑排序是如何分析程序中的方法调用过程的。

有如下方法调用过程图：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/9e0a1735-cda7-440e-ad5e-4ac3da3e933e.png?raw=true)

但是图中出现了med1->med2->med6->med1出现了环，是无法进行逆拓扑排序。

但作者用了一个栈(stack)，一个访问记录表(visited)、和一个出战列表(sortedmethods)数据结构来记录整个遍历过程。

为了方便查看，Longof师傅简化了一下图形结构，用树形的方式来表达：  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/f5338cbb-972a-4f9f-a8ea-d59d210578d9.png?raw=true)

从根节点med1开始遍历，先把med1加入到stack中

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/1edcc225-c8d8-4f7b-9690-c9ac1573bf30.png?raw=true)

之后再判断，med1有哪些子方法，发现了med2，因此将med2也加入到堆栈中

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/e758dc21-f4e7-4f78-8d87-fc96d7cbb26d.png?raw=true)

再从med2为节点进行遍历，发现还有med3，med3后面还有med7。因此将med3和med7都压入栈中

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/67ca163e-33a5-49c4-aa7f-4072d0012525.png?raw=true)

此时med7之后就没有子方法了，便将其从栈中弹出，并标记访问过(存入visited)和出栈顺序(存入sortedmethods)。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/e50aada5-4fa6-47b2-8770-98dcf6c0edf2.png?raw=true)

之后节点回到上一层，发现还有med8没有遍历，且med8没有子方法，因此之后的数据结构存储内容如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/b2ecab1b-1d60-46ba-80e7-575fcbe6c58f.png?raw=true)

继续往上返回，发现med3没有子方法了，弹出，同时发现med2还有子方法med6

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/28a69336-f2ff-4e97-9c9f-68c417174332.png?raw=true)

之后发现med6还有子方法med1，但是med1已经在stack中了，因此出现了环。就丢弃此次的med1，并返回。因此med6并无子方法，弹出med6。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/d5e86611-5ac3-44a5-b5ea-00102f10d64f.png?raw=true)

之后发现med2也无子方法，便弹出med2。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/9e0b1ad2-60ef-4cc5-8c57-f02edb6a99a6.png?raw=true)

此时发现med1还有子方法med3，但是med3已经被visited标记访问过了，因此丢弃med3（去重）。

之后就只剩med4没有访问过，根据之前的遍历方式，加入med4并弹出，之后med1也无未访问过的子方法，再弹出med1。最终变量存储情况如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/064e8bd5-c10d-47b3-bb33-19106175b72a.png?raw=true)

因此按照顺序查看sortedmethods就可以知道逆拓扑排序结果：med7->med8->med3->med6->med2->med4->med1。

----------------分割线-----------题外科普结束-----------------

继续看我们的程序，将所有类的方法调用过程进行DFS遍历，得到方法的调用关系图之后。程序进入到了buildCallGraphs的start()方法中。

```cs
private static void buildCallGraphs(Command command) {
    CallGraphService.start(discoveredCalls, sortedMethods, classFileByName,
                           classMap, graphCallMap, methodMap, methodImpls);
    if (command.isDebug) {
        Output.writeTargetCallGraphs(graphCallMap, command.packageName);
    }
}
```

其中的sortedMethods变量就是我们刚才得到的逆拓扑排序结果

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/dcd5a3a2-4d09-4f88-bf75-8f2e476d86b0.png?raw=true)

进来之后发现创建了CallGraphClassVisitor来访问对应的类文件，而其中又使用了CallGraphMethodAdapter来解析。

同时CallGraphMethodAdapter继承于CoreMethodAdapter，程序在CoreMethodAdapter类中有两个重要的变量，分别是operandStack和localVariables，对应操作数栈和本地变量。而这里就是程序的核心重要部分，模拟代码的调用时的栈帧情况。

**JVM栈帧(Stack Frame)介绍**

每个线程都会拥有自己的多个栈帧

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/62459c22-9c10-48d3-8657-5ac0cb9f156f.png?raw=true)

而一个栈帧中又包括如下几个：

*   局部变量表（Local Variable Table）：在方法执行时，虚拟机使用局部变量表完成参数值到参数变量列表的传递过程的，如果执行的是实例方法，那局部变量表中第 0 位索引的 Slot 默认是用于传递方法所属对象实例的引用。
    
*   操作数栈（Operand Stack）：方法执行的过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈和入栈操作。
    
*   动态链接（Dynamic Linking）：在类加载阶段中的解析阶段会将符号引用转为直接引用，这种转化也称为静态解析。另外的一部分将在运行时转化为直接引用，这部分称为动态链接。
    
*   返回地址（Return Address）：有return、ireturn、areturn等返回方式。
    
*   帧数据区（Stack Data）：Java栈帧还需要一些数据来支持常量池解析、正常方法返回以及异常派发机制。
    

可能读者光看这些概念还不是很理解，这里我会详细讲解一下，假设有如下代码：

```cs
package com.jvm;

/**
 * 编译：javac StackFrame.java
 * 反编译：javap -p -v StackFrame.class
 */
public class StackFrame {
    public static void main(String[] args) {
        add(1, 2);
    }

    private static int add(int a, int b) {
        int c = 0;
        c = a + b;
        return c;
    }
}
```

反编译之后可以看到如下代码：

```properties
Classfile /StackFrame.class
  Last modified 2022年4月6日; size 361 bytes
  MD5 checksum ee66ef285318160bfb9869f53068ac8b
  Compiled from "StackFrame.java"
public class com.jvm.StackFrame
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #3                          // com/jvm/StackFrame
  super_class: #4                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 3, attributes: 1
Constant pool:
   #1 = Methodref          #4.#15         // java/lang/Object."<init>":()V
   #2 = Methodref          #3.#16         // com/jvm/StackFrame.add:(II)I
   #3 = Class              #17            // com/jvm/StackFrame
   #4 = Class              #18            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               main
  #10 = Utf8               ([Ljava/lang/String;)V
  #11 = Utf8               add
  #12 = Utf8               (II)I
  #13 = Utf8               SourceFile
  #14 = Utf8               StackFrame.java
  #15 = NameAndType        #5:#6          // "<init>":()V
  #16 = NameAndType        #11:#12        // add:(II)I
  #17 = Utf8               com/jvm/StackFrame
  #18 = Utf8               java/lang/Object
{
  public com.jvm.StackFrame();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: iconst_1
         1: iconst_2
         2: invokestatic  #2                  // Method add:(II)I
         5: pop
         6: return
      LineNumberTable:
        line 9: 0
        line 10: 6

  private static int add(int, int);
    descriptor: (II)I
    flags: (0x000a) ACC_PRIVATE, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=2
         0: iconst_0
         1: istore_2
         2: iload_0
         3: iload_1
         4: iadd
         5: istore_2
         6: iload_2
         7: ireturn
      LineNumberTable:
        line 13: 0
        line 14: 2
        line 15: 6
}
SourceFile: "StackFrame.java"
```

先来看看main方法的内容：

```properties
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: iconst_1
         1: iconst_2
         2: invokestatic  #2                  // Method add:(II)I
         5: pop
         6: return
      LineNumberTable:
        line 9: 0
        line 10: 6
```

可以看到方法的descriptor是(\[Ljava/lang/String;)V，表明这是一个数组类型的java.lang.String类型的参数，且返回值为V（void）

flags又说明了该方法是一个public static修饰的属性

再来看看code中的stack和locals，说明这个地方只需要2个栈空间和一个本地变量

iconst\_1和iconst\_2是将int类型数值1和2压入栈中。这里当int取值-1~5采用iconst指令，取值-128~127采用bipush指令，取值-32768~32767采用sipush指令，取值-2147483648~2147483647采用 ldc 指令。

继续往后面看invokestatic指令调用了一个静态的类方法，而这个类是#2，也就是在常量池中定义的StackFrame.add方法

```shell
#2 = Methodref          #3.#16         // com/jvm/StackFrame.add:(II)I
```

再来看看add方法的调用

```properties
private static int add(int, int);
    descriptor: (II)I
    flags: (0x000a) ACC_PRIVATE, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=2
         0: iconst_0
         1: istore_2
         2: iload_0
         3: iload_1
         4: iadd
         5: istore_2
         6: iload_2
         7: ireturn
      LineNumberTable:
        line 13: 0
        line 14: 2
        line 15: 6
```

方法初始化时stack=2，locals=3，且有两个参数传递进来，因此初始化栈帧情况如下图所示：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/07fa98b3-edc2-43f6-8da0-c5b6baf836f1.png?raw=true)

前两行定义了一个变量0，并通过istore_2指令存入到本地变量的第2个下标上，栈帧变成如下图：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/c9be3661-8eae-48cb-acec-e5e4d12855ad.png?raw=true)

之后再经过iload的两个指令，取本地变量的0和1下标的值压入栈中  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/6eb006d3-cff2-48c4-a8ef-8d37e441d04a.png?raw=true)

后面就是通过iadd将arg2和arg1相加，再把结果result存入操作数栈中。并用istore\_2更新本地变量，和iload\_2重写加载到操作数栈中，方便后续return返回。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/8fe18e93-6801-4c28-9969-f5c18e7cef0d.png?raw=true)

而SpringInspector程序，就是通过分析字节码，来模拟栈帧(Stack Frame)的方式来分析代码。知道其原理了，接下来就继续通过代码来查看其具体的实现方式。

先来看看CoreMethodAdapter重写的visitCode方法

```cs
@Override
public void visitCode() {
    super.visitCode();
    localVariables.clear();     //初始化数据
    operandStack.clear();       //初始化数据

    if ((this.access & Opcodes.ACC_STATIC) == 0) {      //如果this.access不是静态方法，则结果会等于0
        localVariables.add(new HashSet<>());        //加上this指针
    }
    for (Type argType : Type.getArgumentTypes(desc)) {
        for (int i = 0; i < argType.getSize(); i++) {
            localVariables.add(new HashSet<>());
        }
    }
}
```

在子类实现的CallGraphMethodAdapter类中visitCode方法设置参数索引

```cs
@Override
public void visitCode() {
    super.visitCode();
    int localIndex = 0;
    int argIndex = 0;
    if ((this.access & Opcodes.ACC_STATIC) == 0) {      //不是static方法时，local variable的0下标是this指针
        localVariables.set(localIndex, "arg" + argIndex);
        localIndex += 1;
        argIndex += 1;
    }
    for (Type argType : Type.getArgumentTypes(desc)) {
        localVariables.set(localIndex, "arg" + argIndex);
        localIndex += argType.getSize();
        argIndex += 1;
    }
}
```

在往后跟，程序在遇到方法调用的时候，会触发重写的visitMethodInsn方法

```typescript
@Override
@SuppressWarnings("all")
public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {
    Type[] argTypes = Type.getArgumentTypes(desc);      //解析descriptor，判断方法的参数类型
    if (opcode != Opcodes.INVOKESTATIC) {       //不是静态方法时
        Type[] extendedArgTypes = new Type[argTypes.length + 1];
        System.arraycopy(argTypes, 0, extendedArgTypes, 1, argTypes.length);        //将参数类型复制到extendedArgTypes的1下标位置
        extendedArgTypes[0] = Type.getObjectType(owner);        //0下标位置存放this对象
        argTypes = extendedArgTypes;
    }
    switch (opcode) {       //检查方法调用的类型
        case Opcodes.INVOKESTATIC:
        case Opcodes.INVOKEVIRTUAL:
        case Opcodes.INVOKESPECIAL:
        case Opcodes.INVOKEINTERFACE:
            int stackIndex = 0;
            for (int i = 0; i < argTypes.length; i++) {     //遍历参数类型
                int argIndex = argTypes.length - 1 - i;     // 这个argIndex是目标方法的参数索引
                Type type = argTypes[argIndex];
                Set<String> taint = operandStack.get(stackIndex);       // 从Operand Stack中取出当前参数对应的值
                if (taint.size() > 0) {
                    for (String argSrc : taint) {
                        if (!argSrc.startsWith("arg")) {
                            throw new IllegalStateException("invalid taint arg: " + argSrc);
                        }
                        int dotIndex = argSrc.indexOf('.');
                        int srcArgIndex;
                        if (dotIndex == -1) {
                            srcArgIndex = Integer.parseInt(argSrc.substring(3));
                        } else {
                            srcArgIndex = Integer.parseInt(argSrc.substring(3, dotIndex));
                        }
                        //CallGraph类中有调用方法和目标方法，调用参数和目标参数的索引，用来标记传播规则
                        discoveredCalls.add(new CallGraph(
                            new MethodReference.Handle(
                                new ClassReference.Handle(this.owner), this.name, this.desc),       //从什么方法调用
                            new MethodReference.Handle(
                                new ClassReference.Handle(owner), name, desc),      //调用了什么方法
                            srcArgIndex,
                            argIndex));
                    }
                }
                stackIndex += type.getSize();
            }
            break;
        default:
            throw new IllegalStateException("unsupported opcode: " + opcode);
    }
    super.visitMethodInsn(opcode, owner, name, desc, itf);
}
```

这个方法前面添加this参数类型的代码这块我就不细说了，重点讲switch-case语句的内容

在其中，从operandStack中获取当前的操作数栈的内容

```sql
Set<String> taint = operandStack.get(stackIndex);
```

之后会遍历taint集合，定位调用函数的参数，传递到了下一个方法的哪一个索引  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/aff42bdd-64e3-4aaf-8476-f77dceab05bd.png?raw=true)

这里的CG方法调用图，是侧重于方法/函数之间的调用流程图，而CFG是控制流程图，侧重于if-else、switch-case这种带有流程的语句分析，读者千万不要搞混了。

再来看看靶场的SSRF漏洞CG图是如何生成的

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/ac87b2ee-7460-4ccb-9c80-c70f789f1af2.png?raw=true)

最后就会把SSRFController.ssrf1() -> SSRFService.ssrf1()这个调用流程加入到discoveredCalls集合中。  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/3d49a1f6-f283-4001-afdb-3c6196dd6ff2.png?raw=true)

但是读者就发现了，这里的ssrfService的ssrf1方法是接口，并不是ssrf1的实现类。这又应该怎么找到呢。  

其实在CallGraphService类的CallGraphClassVisitor访问结束后，会对当前返回的disc。对discoveredCalls进行优化。

```cs
// resolve interface problem: interface -> impls
for (int i = 0; i < discoveredCalls.size(); i++) {
    if (i >= tempList.size()) {
        break;
    }
    MethodReference.Handle targetMethod = tempList.get(i).getTargetMethod();        //调用的目标方法
    ClassReference.Handle handle = targetMethod.getClassReference();
    ClassReference targetClass = classMap.get(handle);      //从目标方法中检索到获取目标类
    if (targetClass == null) {
        continue;
    }
    if (targetClass.isInterface()) {        //判断目标类是否为接口
        Set<MethodReference.Handle> implSet = methodImplMap.get(targetMethod);      //从之前的方法实现类Map中检索该接口对应的实现类
        if (implSet == null || implSet.size() == 0) {
            continue;
        }
        for (MethodReference.Handle implMethod : implSet) {
            String callerDesc = methodMap.get(implMethod).getDesc();    //获取该实现类的Desc
            if (targetMethod.getDesc().equals(callerDesc)) {        //判断是否为接口的实现类
                tempList.add(new CallGraph(         //将接口方法到实现类的方法实现也创建一个调用流程图
                    targetMethod,
                    implMethod,
                    tempList.get(i).getTargetArgIndex(),
                    tempList.get(i).getTargetArgIndex()
                ));
            }
        }
    }
}
// unique
discoveredCalls.clear();
discoveredCalls.addAll(tempList);       //更新discoveredCalls
```

之前在inherit()方法中遍历的关系和接口实现类，在这里就可以通过接口名称找到对应的实现类。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/fb9304dc-2020-46a9-aa8d-b5592cd8cf87.png?raw=true)

最后build到graphCallMap数据结构中。

之后就来到初始化过程的最后一步，解析Springboot的方法

```cs
/**
* 通过Spring的Controller注解来解析有Controller注解的类
* @param command
*/
private static void parseSpring(Command command) {
    if (command.springBoot) {
        SpringService.start(classFileList, command.packageName, controllers, classMap, methodMap);
        if (command.isDebug) {
            Output.writeControllers(controllers);
        }
    }
}
```

跟进SpringService的start方法，看看程序是如何解析SpringBoot的。  

跟进来还是常规的ASM操作，用SpringClassVisitor来访问所有的类文件，先来看看他的Visitor方法

```typescript
@Override
public void visit(int version, int access, String name, String signature,
                  String superName, String[] interfaces) {
    this.name = name;
    if (name.startsWith(this.packageName)) {
        Set<String> annotations = classMap.get(new ClassReference.Handle(name)).getAnnotations();       //获取类的注解
        if (annotations.contains(SpringConstant.ControllerAnno) ||
            annotations.contains(SpringConstant.RestControllerAnno)) {      //判断该类是不是Controller的请求处理类
            this.isSpring = true;       //设置该类是Spring的请求处理类
            currentController = new SpringController();     //创建一个SpringController类承载该请求类
            currentController.setClassReference(classMap.get(new ClassReference.Handle(name)));
            currentController.setClassName(new ClassReference.Handle(name));
            currentController.setRest(!annotations.contains(SpringConstant.ControllerAnno));        //相关属性设置完成
        }
    } else {
        this.isSpring = false;
    }
    super.visit(version, access, name, signature, superName, interfaces);
}
```

上述方法结束后，会对类文件的方法进行访问，因此程序会进入到visitMethod方法中

```kotlin
@Override
public MethodVisitor visitMethod(int access, String name, String descriptor,
                                 String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (this.isSpring) {        //判断是不是Spring请求处理类
        if (name.equals("<init>") || name.equals("<clinit>")) {       //如果是初始化方法，或者是静态代码块，则跳过该方法
            return mv;
        }
        return new SpringMethodAdapter(name, descriptor, this.name, Opcodes.ASM6, mv,
                                       currentController, this.methodMap);     //方法进入到SpringMethodAdapter选择器中
    } else {
        return mv;
    }
}
```

在SpringMethodAdapter中，会遍历类中的方法注解和方法的参数，判断Controller是用何种方式访问，并使用SpringMapping类来存放这些Controller属性信息。

至此程序初始化的过程就结束了，下面接着来看看SpringInspector是如何检测产出漏洞的。

**04**

**漏洞分析判定过程**

回到之前的org.sec.app.Application的start(Command command)方法中  

方法中的前几个初始化过程已经结束了，之后就是检测输入的命令需要检测的漏洞类型

```java
if (command.module == null || command.module.equals("")) {
    logger.warn("no module selected");
} else {
    String module = command.module.toUpperCase(Locale.ROOT);
    if (module.contains("ALL")) {
        module = "SSRF|SQLI|XXE|RCE|DOS|REDIRECT|LOG";
    }
    if (module.contains("SSRF")) {
        SSRFService.start(classFileByName, controllers, graphCallMap);
        resultInfos.addAll(SSRFService.getResults());
    }
    if (module.contains("SQLI")) {
        SqlInjectService.start(classFileByName, controllers, graphCallMap);
        resultInfos.addAll(SqlInjectService.getResults());
    }
    if (module.contains("XXE")) {
        XXEService.start(classFileByName, controllers, graphCallMap);
        resultInfos.addAll(XXEService.getResults());
    }
    if (module.contains("RCE")) {
        RCEService.start(classFileByName, controllers, graphCallMap);
        resultInfos.addAll(RCEService.getResults());
    }
    if (module.contains("DOS")) {
        DOSService.start(classFileByName, controllers, graphCallMap);
        resultInfos.addAll(DOSService.getResults());
    }
    if (module.contains("LOG")) {
        LogService.start(classFileByName, controllers, graphCallMap);
        resultInfos.addAll(LogService.getResults());
    }
    if (module.contains("REDIRECT")) {
        RedirectService.start(classFileByName, controllers);
        resultInfos.addAll(RedirectService.getResults());
    }
}
```

这里就还是拿之前介绍的SSRF漏洞来跟踪吧

```css
SSRFService.start(classFileByName, controllers, graphCallMap);
```

传递的三个参数均是之前初始化过程中返回的结果集，如果还没看懂建议再回过头多看几遍。

最终进入到了org.sec.core.service.base.BaseService的start0方法中，我这里截取了关键代码

```cs
for (SpringController controller : controllers) {       //遍历循环controller
    for (SpringMapping mapping : controller.getMappings()) {
        MethodReference methodReference = mapping.getMethodReference();
        if (methodReference == null) {
            continue;
        }
        Type[] argTypes = Type.getArgumentTypes(methodReference.getDesc());     //获取Mapping的方法参数类型
        Type[] extendedArgTypes = new Type[argTypes.length + 1];
        System.arraycopy(argTypes, 0, extendedArgTypes, 1, argTypes.length);      //腾出argTypes[0]的位置，Mapping的方法肯定是非Static，因此是有this指针的
        argTypes = extendedArgTypes;
        boolean[] vulnerableIndex = new boolean[argTypes.length];
        for (int i = 1; i < argTypes.length; i++) {     //将Mapping的所有参数标记为污点
            vulnerableIndex[i] = true;
        }
        Set<CallGraph> calls = allCalls.get(methodReference.getHandle());       //获取该方法的调用图集合
        if (calls == null || calls.size() == 0) {
            continue;       //方法中没有调用，则跳过此Mapping分析
        }
        tempChain.add(methodReference.getClassReference().getName() + "." + methodReference.getName());     //将方法的全限定名添加进tempChain
        for (CallGraph callGraph : calls) {     //遍历调用图
            int callerIndex = callGraph.getCallerArgIndex();        //获取调用参数的索引
            if (callerIndex == -1) {
                continue;
            }
            if (vulnerableIndex[callerIndex]) {     //如果调用参数的索引是有污点标记的，则进入到传播分析中
                tempChain.add(callGraph.getTargetMethod().getClassReference().getName() + "." +
                              callGraph.getTargetMethod().getName());     //将目标方法的完全限定名也加入到tempChain中
                List<MethodReference.Handle> visited = new ArrayList<>();
                doTask(callGraph.getTargetMethod(), callGraph.getTargetArgIndex(), visited);
                if (isSqli) {
                    if (hasSqlInject) {
                        SqliCollector.collect(tempChain, results);
                    }
                    hasSqlInject = false;
                }
            }
        }
        tempChain.clear();
    }
}
```

方法获取了Mapping的CG调用图，并判断调用图中传递的参数是否有污点标记，继续往下跟踪到doTask方法中。

```kotlin
private static void doTask(MethodReference.Handle targetMethod, int targetIndex,
                           List<MethodReference.Handle> visited) {
    if (visited.contains(targetMethod)) {       //判断目标方法是否已经访问过
        return;
    } else {
        visited.add(targetMethod);
    }
    ClassFile file = classFileMap.get(targetMethod.getClassReference().getName());      //获取目标方法的类文件
    try {
        if (file == null) {
            return;
        }
        ClassReader cr = new ClassReader(file.getFile());
        Constructor<?> c = classVisitor.getConstructor(MethodReference.Handle.class, int.class);        //用相关定义的漏洞Visitor来消费传播
        Object obj = c.newInstance(targetMethod, targetIndex);
        BaseClassVisitor cv = (BaseClassVisitor) obj;
        cr.accept(cv, ClassReader.EXPAND_FRAMES);
        if (isSqli) {
            SqlInjectClassVisitor sqlCv = (SqlInjectClassVisitor) cv;
            if (sqlCv.getSave().contains(true)) {
                hasSqlInject = true;
                return;
            }
        } else {
            Method method = collector.getMethod("collect", BaseClassVisitor.class, List.class, List.class);
            method.invoke(null, cv, tempChain, results);        //调用collect方法，并将结果输出到results变量中
        }
        tempChain.clear();
    } catch (Exception e) {
        e.printStackTrace();
        return;
    }
    Set<CallGraph> calls = allCalls.get(targetMethod);
    if (calls == null || calls.size() == 0) {
        return;
    }
    for (CallGraph callGraph : calls) {         //通过递归的方式，查看调用图中的目标方法是否还有传播路径
        if (callGraph.getCallerArgIndex() == targetIndex && targetIndex != -1) {
            if (visited.contains(callGraph.getTargetMethod())) {
                return;
            }
            tempChain.add(callGraph.getTargetMethod().getClassReference().getName() + "." +
                          callGraph.getTargetMethod().getName());
            doTask(callGraph.getTargetMethod(), callGraph.getTargetArgIndex(), visited);
        }
    }
}
```

doTask中是使用递归的方法来分析调用图中的目标方法是否还有调用，同时还在其中创建了SSRFClassVisitor类来消费targetMethod的目标类文件。

跟进后可以看到SSRFClassVisitor类是继承了BaseClassVisitor，再来看看BaseClassVisitor中的关键方法visitMethod是如何消费目标类的方法。

```kotlin
@Override
public MethodVisitor visitMethod(int access, String name, String descriptor,
                                 String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    try {
        if (this.methodHandle!=null && name.equals(this.methodHandle.getName())) {
            MethodVisitor methodVisitor = (MethodVisitor) targetMethodAdapter
                .getConstructors()[0].newInstance(this.methodArgIndex, this.pass,
                                                  Opcodes.ASM6, mv, this.name, access, name, descriptor);
            return new JSRInlinerAdapter(methodVisitor,
                                         access, name, descriptor, signature, exceptions);
        }
    }catch (Exception e){
        e.printStackTrace();
    }
    return mv;
}
```

观看代码可以看到是调用了targetMethodAdapter的构造方法，而这里的targetMethodAdapter是之前传入的org.sec.core.asm.SSRFMethodAdapter类，该类的父类是由MethodVisitor继承过来的

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/cd53d39d-9a36-4f06-ba92-0351a75e65f1.png?raw=true)

再来看看SSRFMethodAdapter中的visitMethodInsn方法，这里我简化了一下规则

```cs
@Override
public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {
  boolean urlCondition = owner.equals("java/net/URL") && name.equals("<init>") &&
      desc.equals("(Ljava/lang/String;)V");
  boolean urlOpenCondition = owner.equals("java/net/URL") && name.equals("openConnection") &&
      desc.equals("()Ljava/net/URLConnection;");
  boolean urlInputCondition = owner.equals("java/net/HttpURLConnection") &&
                name.equals("getInputStream") && desc.equals("()Ljava/io/InputStream;");
  if (urlCondition || urlOpenCondition) {
    if (operandStack.get(0).contains(true)) {
      super.visitMethodInsn(opcode, owner, name, desc, itf);
      operandStack.set(0, true);
      return;
    }
  }
  if (urlInputCondition) {
    if (operandStack.get(0).contains(true)) {
      pass.put("JDK", true);
      super.visitMethodInsn(opcode, owner, name, desc, itf);
      return;
    }
  }
  super.visitMethodInsn(opcode, owner, name, desc, itf);
}
```

可以看到这里定义了Sink点，举例在SSRFService的实现类里面有HttpURLConnection

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/11b041af-75c7-4a95-af3f-28fdbcf243e7.png?raw=true)

再来看看SSRFMethodAdapter的父类ParamTaintMethodAdapter，其中重写了visitCode方法  

```java
@Override
public void visitCode() {
    super.visitCode();
    int localIndex = 0;
    int argIndex = 0;
    if ((this.access & Opcodes.ACC_STATIC) == 0) {
        localIndex += 1;
        argIndex += 1;
    }
    for (Type argType : Type.getArgumentTypes(desc)) {
        if (argIndex == this.methodArgIndex) {
            localVariables.set(localIndex, true);
        }
        localIndex += argType.getSize();
        argIndex += 1;
    }
}
```

在if分支argIndex和this.methodArgIndex比较是否相等的时候，会把该变量的localIndex索引打上“污点”值true。

再来看看靶场的ssrf1存在漏洞的IR代码

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/c2b71ba1-6272-484c-b7ef-1935e7550166.png?raw=true)

在规则的前面都有aload方法  

同时在CoreMethodAdapter类的解析字节码文件方法中，是通过将本地变量中存放的值，而之前已经在visitCode方法中已经将对应的localVariables的参数索引打上了“污点”。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/6bb6dbb2-8884-486a-922f-2047e3857de3.png?raw=true)

再来回看一下之前的方法的消费visitMethodInsn  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/db46c8e3-6fc6-48ec-9db8-81f20cfa8b70.png?raw=true)

这里还需要设置传播函数，是因为污点参数data是直接进入的URL中，而最后的Sink点是HttpURLConnection.getInputStream()方法，因此污点不应该在URL处丢失。所以需要检测URL和url.openConnection()地方的污点是否连续。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/25f81686-2ab1-414b-8505-59441c6d498f.png?raw=true)

当程序检测到Sink点getInputStream方法后，后会进入如下if分支中

```swift
if (urlInputCondition) {
    if (operandStack.get(0).contains(true)) {  //判断操作数栈中是否标记该参数为污点
        pass.put("JDK", true);    //标记JDK关键词
        super.visitMethodInsn(opcode, owner, name, desc, itf);
        return;
    }
}
```

给pass变量添加了JDK的关键词

再接着看doTask方法的后续内容

```kotlin
Method method = collector.getMethod("collect", BaseClassVisitor.class, List.class, List.class);
method.invoke(null, cv, tempChain, results);        //调用collect方法，并将结果输出到results变量中
```

这里调用的collect方法是SSRFCollector类的collect静态方法

这里的cv变量就是刚才ASM消费的结果

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/3a6a8329-08ed-4942-8800-1463a774716c.png?raw=true)

跟进collect方法中查看  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/20a9346e-fab7-4973-a4c2-5cf27b9dd55a.png?raw=true)

```cs
if (cv.getPass("JDK") != null && cv.getPass("JDK")) {
    ResultInfo resultInfo = new ResultInfo();
    resultInfo.setRisk(ResultInfo.MID_RISK);  //设置中危
    resultInfo.setVulName("JDK SSRF");    //设置漏洞名称
    resultInfo.getChains().addAll(tempChain);    //将有漏洞的完全限定名放进去
    results.add(resultInfo);    //放到结果集中
    logger.info("detect jdk ssrf");
    System.out.println(resultInfo);
}
```

至此就是SpringInspector分析字节码的原理，总体来说是通过模拟堆栈来获取调用图CG，并通过解析SpringBoot文件的Controller的Mapping方法，设置其Mapping传入的参数为污点源，再通过分析其对应Mapping的调用流程图看污点参数是如何传播的。

以上就是白盒引擎中，字节码的分析原理。之后我再介绍一下目前主流分析引擎，针对QL图形查找的CPG分析技术。

**05**

Code Property Graph（CPG）分析技术浅析

SpringInspector实际上是属于针对字节码解析和堆栈模拟上的污点传播分析，这种情况其实对内存开销还是非常大的，不过本篇既然讲到了白盒分析，自然就开个分支讲一下目前市面上主流的白盒分析技术CPG。

CPG又称代码属性图，用来描述其程序的AST抽象语法树、CFG控制流程图、PDG程序依赖图

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/fd08fd08-5a5e-49dd-8d69-3443e9538e1d.png?raw=true)

样例程序：  

```cs
void foo(){
  int x = source();
  if (x < MAX){
    int y = 2 * x;
    sink(y);
  }
}
```

静态代码扫描器会将程序代码逻辑生成出以下三种图

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/de1fcb3e-9c3c-4f4a-ae89-6c21e8fb72b3.png?raw=true)

而将其转换成CPG图的时候：  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/6c15434f-151b-44ee-a0c2-671353d50a06.png?raw=true)

而一张CPG图有如下几种特征：

*   节点及其类型。节点代表程序的组成部分，包含有底层语言的函数、变量、控制结构等，还有更抽象的 HTTP 终端等。每个节点都有一个类型，如类型 METHOD 代表是方法，而 LOCAL 代表是一个局部变量。
    
*   有向边及标签。节点之间的边代表它们之间的关系，而标签则是关系的说明。比如一个方法 CONTAINS 包含了一个局部变量。
    
*   键值对。节点带有键值对，即属性，键取决于节点的类型。比如一个方法至少会有一个方法名和一个签名，而一个局部变量至少有名字和类型。
    

CPG的开源工具

Plume：https://plume-oss.github.io/

Joern：https://joern.io/

通常会将JAR文件读取，再通过Soot将类文件转换为AST，并通过AST等信息来生成CFG调用图，最后生成CPG图。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/02071746-26b0-476a-a063-40db62c8c3a4.png?raw=true)

之后我会直接使用joern-cli项目里的工具来生成CPG和查询CPG  

而我这里使用的靶场就是自己用Servlet编写的XSS漏洞，具体代码如下：

```java
package org.vulnServlet.Controller;

import java.io.IOException;
import java.io.PrintWriter;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * Servlet implementation class XssServlet
 */
@WebServlet("/XssServlet")
public class XssServlet extends HttpServlet {
  private static final long serialVersionUID = 1L;

    /**
     * @see HttpServlet#HttpServlet()
     */
    public XssServlet() {
        super();
    }

  /**
   * @see HttpServlet#doGet(HttpServletRequest request, HttpServletResponse response)
   */
  protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    String text = request.getParameter("name");
    PrintWriter out = response.getWriter();
    out.println("Hello, "+text);
  }

  /**
   * @see HttpServlet#doPost(HttpServletRequest request, HttpServletResponse response)
   */
  protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    doGet(request, response);
  }

}
```

用joern-parse编译生成cpg

```sql
./joern-parse --language java /opt/joern-cli/javacode
```

之后会在本目录生成一个cpg.bin文件，随后用importCpg命令导入

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/22f37524-049e-4815-b0f4-509fce35516d.png?raw=true)

并使用workspace命令查看该项目的open状态是否为true，如果不为true则需要用open("cpg.bin")的方式打开工程。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/57b73966-f790-4765-88c3-8c3db507782e.png?raw=true)

之后就可以通过cpg.method.l的方式查看所有的方法，这里我用CPGQL Query语句查询println方法使用的地方

```javascript
({cpg.method.name("println").callIn}).l
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/43254d3b-1f43-47a7-8bc7-dfd16e4d482c.png?raw=true)

能够查到数据，就说明joern已经成功解析了cpg流程。这时候我们再定义source点到sink点的查询语句，查询我们的xss漏洞：

```ruby
({def source =
  cpg.call.methodFullNameExact(
    "javax.servlet.http.HttpServletRequest.getParameter:java.lang.String(java.lang.String)"
  )

def responseWriter =
  cpg.call.methodFullNameExact(
    "javax.servlet.http.HttpServletResponse.getWriter:java.io.PrintWriter()"
  )

def sinks =
  cpg.call.methodFullNameExact(
      "java.io.PrintWriter.println:void(java.lang.String)"
    )
    .where(_.argument(0).reachableBy(responseWriter))

sinks.where(_.argument(1).reachableBy(source))}).l
```

查询的语句的最后一行是判断sinks的第1个参数是否是有流通source点的流，其查询结果如下  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/81ff62b6-7887-4397-890f-4e46c61968b6.png?raw=true)

可以成功查询到结果，且可以看到是相关代码的第32行

更多的查询语句：https://queries.joern.io/

本篇文章暂不对CPG的漏洞定位做过多的探讨，主要以SpringInspector的字节码分析技术为主，CPG算是本篇文章的一部分题外探讨，扩展一下读者对市场上主流的白盒扫描有一个大概的认识。感兴趣的读者也可以多仔细研究研究CodeQL，原理就是通过图形查询的方式关注Source-Sink数据流。

**06**

**总结与思考**

前面分析了SpringInspector的源码，以及如何检测产出漏洞的。经过思考和以前学习的IAST相关知识，绝对还可以往如下几个方向深入调研和改进：

1.  污点传播过程中清洗函数怎么定义？能否通过自然语言处理得到其特征来自动化做？
    
2.  SpringInspector是通过定义漏洞类来检测调用目标方法的参数是否带有污点标记，可是这样成本很高，每次支持一个新的漏洞或者一条新的Sink点都需要重新写代码重新编译，是否有可以通过数据库读写来热更新漏洞规则。
    
3.  SpringInspector是通过Controller的Mapping来定义污点传播的Source点，能不能支持除开SpringBoot的框架？说个直白的例子，如果开发人员在Controller的Mapping中使用Request.getParameter获取参数，能否还能定位到污点源？或者用数据绑定的方式，如何知道把污点打到哪个参数上？
    
4.  这里是通过模拟堆栈的情况下传播的污点源，对漏洞传播的规则也需要不全，如果没有new URL()的传播规则，则污点进入URL构造方法后就会丢失，如何补全传播规则。
    
5.  白盒该如何集成到CI/CD中？集成到CI/CD中如何缩短检测时间，提高流水线效率？
    
6.  SpringInspector开发成B/S架构的可行性如何，对系统性能的要求怎样，分析万行代码的时间需要多少？
    

在写这篇文档的过程中，其实也思考了自己之前走过的路。觉得之前太操之过急，急于做出一些东西出来，导致很多知识都是浅尝辄止，从未过深的钻研背后的原理及思考其可扩展的方向。经过这篇文章，翻阅了很多老前辈的资料和文章，才知道自己都是一些肤浅末学，只知其然，而不知其所以然。所以这篇文章是笔者经过翻阅许多资料总结的一些浅见，为的就是能尽可能的全面展现静态代码技术的核心原理。如果文章中有出现文笔错误的，或是讲的不够细致的，欢迎联系笔者一起探讨。

**07**

**Reference**

\[1\].https://xz.aliyun.com/t/10433

\[2\].https://tttang.com/archive/1334/

\[3\].《污点分析技术的原理和实践应用》http://www.jos.org.cn/html/2017/4/5190.htm

\[4\].https://paper.seebug.org/1034

\[5\].《Modeling and Discovering Vulnerabilities with Code Property Graphs》【Fabian Yamaguchi】

\[6\].https://blog.csdn.net/water_likly/article/details/105609686

\[7\].https://blog.csdn.net/fcsfcsfcs/article/details/119296342

\[8\].https://www.freebuf.com/articles/es/259762.html

\[9\].https://www.cnblogs.com/jhxxb/p/11001238.html

\[10\].《面向Jimple语言的基于依赖的污点分析方法设计与实现》【周轩】https://www.docin.com/p-1547764509.html

\[11\].https://xz.aliyun.com/t/10216（建议读者读读，文章中介绍了不少的白盒工具的调研）

\[12\].https://github.com/zsdlove/hades

\[13\].《软件漏洞分析技术》著【吴世忠】

本次分享不易，如果喜欢本文，点个转发和在看吧~

* * *

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/b4b9a270-e8e6-48d5-a0f8-3d8d78ab58ee.png?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/4289b7bf-a517-4dc8-aee1-ab6ce7f24ff9.png?raw=true)