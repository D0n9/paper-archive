# CoraxJava 社区版使用及编写自定义规则
这里引用官方的介绍：

> CoraxJava(Corax社区版)是一款针对Java项目的静态代码安全分析工具，其核心分析引擎来自于Corax商业版，具备与Corax商业版一致的底层代码分析能力，并在此基础上配套了专用的开源规则检查器与规则。
> 
> CoraxJava由两部分组成，分别是`CoraxJava核心分析引擎`和`CoraxJava规则检查器`。其中规则检查器模块支持实现多种规则的检查。目前CoraxJava包含了抽象解释和IFDS（Sparse）等程序分析技术。未来，我们将持续优化引擎分析性能和提高分析精度，并研究引入更强大的程序分析算法，以持续增强核心分析能力，推动代码的安全、质量和性能不断提升。
> 
> CoraxJava具有以下特点：
> 
> 1.  完全开放的规则检查器模块，开源多个规则检查器代码示例。
>     
> 2.  支持使用Kotlin/Java语言开发自定义规则检查器。
>     
> 3.  支持通过**配置文件**或**编写代码**的方式修改、生成规则检查器。
>     
> 4.  分析对象为Java字节码，但需要源代码作为结果显示的参考和依据
>     
> 5.  分析结果以sarif格式输出。
>     

社区版和商业版的一些区别：

可以看到社区版也支持Java污点分析，下面简单介绍一下安装使用CoraxJava和自定义规则：

### 一、环境要求

首先建议安装Java17版本，我这里已经安装好了：

查看Java版本是否符合要求

`java -version`

### ![](https://iotaa.cn/upload/image.png)
二、编译构建

1、下载核心引擎：

[https://github.com/Feysh-Group/corax-community/releases/tag/v2.0](https://github.com/Feysh-Group/corax-community/releases/tag/v2.0)

2、克隆项目到本地并且执行build：

```null
git clone https://github.com/Feysh-Group/corax-community.git
cd corax-community
./gradlew build
```

![](https://iotaa.cn/upload/image-cwef.png)
提示需要修改gradle-local.properties

那么直接修改coraxEnginePath为刚才release下载的jar包，这里注意**要使用绝对路径**：

![](https://iotaa.cn/upload/image-ciye.png)
再次执行./gradlew build，可以看到已经build成功了：

### ![](https://iotaa.cn/upload/image-whfs.png)
三、开始分析

根据官方文档，运行以下命令：

```null
java -jar ~/corax-cli-community-2.0.jar --verbosity info --output build/output --enable-data-flow true --target java --result-type sarif --auto-app-classes ./corax-config-tests --config default-config.yml@./build/analysis-config
```

~/corax-cli-community-2.0.jar是下载的核心引擎路径

--output 是输出的结果路径

--auto-app-classes 对应的是要分析的代码路径

--config是规则路径，也就是上一步编译后的规则路径

然后执行可以看到正在分析，等待分析结束：

### ![](https://iotaa.cn/upload/image-pcdi.png)
四、查看结果：

这里使用Vscode安装Sarif Viewer插件：

![](https://iotaa.cn/upload/image-ucyh.png)
打开刚才生成的output目录：

![](https://iotaa.cn/upload/image-kafp.png)
可以看到漏洞的详细信息：

![](https://iotaa.cn/upload/image-hgwe.png)
查看漏洞流：

![](https://iotaa.cn/upload/image-qfqg.png)
可以看到结果检测出了命令执行漏洞。

### 五、增加规则：

官方项目自带的检查规则：

![](https://iotaa.cn/upload/image-dvij.png)
这里我们来自己定义一个SSRF漏洞的检测规则：

1、首先根据官方说明选择合适的规则检查器：

![](https://iotaa.cn/upload/image-gcvv.png)
SSRF属于注入类的问题，也就是可以直接使用已有的污点分析，只需要增加相关规则就可以实现。

首先修改com/feysh/corax/config/community/CheckerDeclarations.kt

增加对SSRF检查器的定义：

```null
object SSRFChecker : IChecker {
    override val report: IRule = CWERules.CWE918_SSRF
    override val category: IBugCategory = BugCategory.Ssrf
    override val standards: Set<IRule> = setOf(
        CWERules.CWE918_SSRF,
    )
    object ServersideRequestForgery : CheckType() {
        override val bugMessage: Map<Language, BugMessage> = mapOf(
            Language.ZH to msgGenerator { "使用  ${args["type"]} 中的 `$callee` 可能容易受到服务端请求伪造漏洞的攻击" },
            Language.EN to msgGenerator { "This use of `$callee` can be vulnerable to Server-Side Request Forgery in the ${args["type"]} " }
        )
        override val checker: IChecker = SSRFChecker
    }
}
```

修改com/feysh/corax/config/community/standard/CWERules.kt

增加CWE918：

```null
CWE918_SSRF("cwe-918","The web server receives a URL or similar request from an upstream component and retrieves the contents of this URL, but it does not sufficiently ensure that the request is being sent to the expected destination.")
```

![](https://iotaa.cn/upload/image-ttqa.png)
修改com/feysh/corax/config/community/category/BugCategory.kt，增加漏洞分类：

```null
Ssrf(setOf(BuiltinBugCategory.SECURITY),"","")
```

然后修改com/feysh/corax/config/community/checkers/taint-checker.kt，增加检查类型：

```null
"server-side-request-forgery" to CustomSinkDataForCheck(control + GeneralTaintTypes.CONTAINS_COMMAND_INJECT, reportType = SSRFChecker.ServersideRequestForgery),
```

![](https://iotaa.cn/upload/image-cjvs.png)
然后就可以使用污点分析来分析SSRF漏洞了，最后需要添加sink规则：

修改corax-config-community/rules/community.sinks.json

增加一条规则，比如这里需要检测sink点：

org.apache.http.client.fluent.Request#Get(java.lang.String)

那么规则定义如下：

```null
{"kind":"server-side-request-forgery","signature":"org.apache.http.client.fluent.Request: * Get(java.lang.String)","subtypes":false,"arg":"Argument[0]","provenance":"manual","ext":""},
```

`kind`就是刚才在taint-checker定义的检查类型

`signature` 可以为 `matchSoot` 格式 或者 为 `matchSimpleSig` 格式。

subtypes 表示是否 handle 子类的重写方法（static method 必定为 false, interface 必定为 true）

`provenance`，和 `ext` 无关紧要，保留字段。

这里我们只需要修改sink规则就可以实现污点分析

修改完成后先编译规则：

```null
./gradlew build
```

### ![](https://iotaa.cn/upload/image-hgci.png)
六、验证漏洞

这里使用的是micro\_service\_seclab靶场：

[micro\_service\_seclab/src/main/java/com/l4yn3/microserviceseclab/controller/](https://github.com/l4yn3/micro_service_seclab/blob/main/src/main/java/com/l4yn3/microserviceseclab/controller/SSRFController.java)[SSRFController.java](http://ssrfcontroller.java/) [at main · l4yn3/micro\_service\_seclab (](https://github.com/l4yn3/micro_service_seclab/blob/main/src/main/java/com/l4yn3/microserviceseclab/controller/SSRFController.java)[github.com](http://github.com/)[)](https://github.com/l4yn3/micro_service_seclab/blob/main/src/main/java/com/l4yn3/microserviceseclab/controller/SSRFController.java)

下载靶场项目，之后编译靶场项目：

```null
git clone https://github.com/l4yn3/micro_service_seclab.git
cd micro_service_seclab
mvn compile -Dmaven.test.skip=true
```

![](https://iotaa.cn/upload/image-dhsz.png)
编译完成后就可以进行扫描了:

```null
java -jar ~/corax-cli-community-2.0.jar --verbosity info --output build/output --enable-data-flow true --target java --result-type sarif --auto-app-classes ~/Downloads/CodeDownload/micro_service_seclab/ --config default-config.yml@./build/analysis-config
```

![](https://iotaa.cn/upload/image-tctu.png)
然后使用Vscode查看扫描结果：

![](https://iotaa.cn/upload/image-qghc.png)
这里能看到整个漏洞的调用链：

### ![](https://iotaa.cn/upload/image-admy.png)
七、知识点补充

#### summaries：

以这段demo为例

```null
String taint = getSecret(); 
StringBuilder sb = new StringBuilder();
sb.append("abc");
sb.append(taint); 
sb.append("xyz");
String s = sb.toString(); 
leak(s); 
```

人工可以轻易看出来存在从source到sink存在污点传播路径，但是需要告诉程序哪些**方法**会引发污点传播，比如这个例子里的append以及toString方法，以及它们是如何传播污点的，

引用南大软件分析教程：

> 当一个引发污点传播的方法 `foo` 被调用时，有以下三种污点传播的模式：
> 
> ![](https://tai-e.pascal-lab.net/pa8/transfer-pattern.png)

1.  **Base-to-result**：如果 receiver object（由 `base` 指向）被污染了，那么该方法调用的返回值也会被污染。`StringBuilder.toString()` 是这样一类方法。
    
2.  **Arg-to-base**：如果某个特定的参数被污染了，那么 receiver object（由 `base` 指向）也会被污染。`StringBuilder.append(String)` 是这样一类方法。
    
3.  **Arg-to-result**：如果某个特定的参数被污染了，那么该方法调用的返回值也会被污染。`String.concat(String)` 是这样一类方法。
    

也就是说这里需要修改以 `summaries.json` 为后缀的文件来告诉程序那些方法会引发污点传播：

*   `"propagate": "taint"` 意为 taint kinds 传递，在方法执行后，to 和 from 可能指向了不同的对象，如果不同，那么 to 被 taint 了或副作用了，from 仍然不受影响，除非在程序中存在别名关系。
    
*   `"propagate": "value"` 意为值传递，to 和 from 指向了一个对象。
    

当缺失传播路径的时候，我们需要手工补充污点传播路径，官方已经定义了一些污点传播路径，如果遇到污点传播中断的地方，可以修改规则手工补全。

以官方的general.summaries[.](https://github.com/Feysh-Group/corax-community/blob/main/corax-config-general/rules/general.summaries.json)json为例

```null
{"signature":"java.lang.AbstractStringBuilder: * <init>(String)","subtypes":true,"argTo":"Argument[this]","propagate":"taint","argFrom":"Argument[0]","provenance":"manual","ext":""},
```

StringBuilder是继承父类的构造函数也就是java.lang.AbstractStringBuilder的init方法:

![](https://iotaa.cn/upload/image-voyy.png)
subtypes设置为true代表包含子类

argTo

propagate taint 代表

也就是说这个传播规则代表的意思是StringBuilder sb = new StringBuilder("xxx")，从构造函数入参的第一个参数到参数本身本身存在污点传播路径；

第二个例子：

```null
{"signature":"java.lang.AbstractStringBuilder: * append(**)","subtypes":true,"argTo":"ReturnValue","propagate":"value","argFrom":"Argument[this]","provenance":"manual","ext":""}
  ...
]
```

这里就是对应的Stringbuilder 的append方法，可以看到Stringbuilder类的append方法是继承AbstractStringBuilder，重写父类的append方法：

![](https://iotaa.cn/upload/image-ewuv.png)

所以我们只需要在规则中定义父类的方法签名，并且subtypes设置为true，就可以覆盖所有子类了，传播方法是Arg-to-result,所以定义argFrom是Argument\[this\]，argTo是ReturnValue，`"propagate": "value"` 意为值传递

待补充：

*   source定义
    
*   sanitizer定义
    
*   定制checker