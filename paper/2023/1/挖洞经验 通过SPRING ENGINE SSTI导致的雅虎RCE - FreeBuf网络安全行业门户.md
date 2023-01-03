# 挖洞经验 | 通过SPRING ENGINE SSTI导致的雅虎RCE - FreeBuf网络安全行业门户
***本文中涉及到的相关漏洞已报送厂商并得到修复，本文仅限技术研究与讨论，严禁用于非法用途，否则产生的一切后果自行承担。** 

**最近在漏洞挖掘过程中发现了一个漏洞，之后提交给了雅虎的bug赏金计划，这篇文章我就来分享下是如何挖掘到此漏洞的。** 

在测试Web应用程序的安全性时，信息收集是查找可能存在漏洞的Web资产的重要组成部分，因为可以发现可能会增加攻击面的子域，目录和其他资产。我首先使用在线工具，如shodan.io，censys.io，crt.sh，[dnsdumpster.com](http://dnsdumpster.com/)，以及github上的脚本，如dirsearch，aquatone，massdns等这些工具进行子域名查找，在使用这些工具后，我发现了子域名 - datax.yahoo.com。这个子域名引起了我的兴趣，因为访问子域名根目录将重定向到[https://datax.yahoo.com/swagger-ui.html](https://datax.yahoo.com/swagger-ui.html) \- 并在页面随后显示403错误。

![](https://image.3001.net/images/20190113/1547367420_5c3af3fccbff7.png!small)
然后我运行dirsearch脚本快速地扫描目录，以此来尝试发现网站隐藏的目录。我不确定具体结果是什么，但我注意到在向网址添加空格字符时，状态代码已从403更改为200，[https://datax.yahoo.com/%20/](https://datax.yahoo.com/%20/)，然后我看到了以下页面。  

![](https://image.3001.net/images/20190113/1547367433_5c3af409a1081.png!small)

发现是swigger，经过查找发现swagger ui的主页上有api文档，并详细列出了所有可用的接口和请求参数。示例：[https://dev.fitbit.com/build/reference/web-api/explore/](https://dev.fitbit.com/build/reference/web-api/explore/)

从这里开始，我作为未经身份验证的用户，想测试一些API接口，以此发现潜在的漏洞。经过几个接口测试后，我遇到了以下接口，它显示了这个white label error页面，其中我输入的参数值却出现在错误响应中：

![](https://image.3001.net/images/20190113/1547367444_5c3af41445b8e.png!small)
注意到输入内容显示在页面中，随后我尝试了一些XSS payload，却无法在客户端成功执行javascript。最后当输入payload为${7*7}时，我惊讶地发现算术表达式已在响应中成功执行计算了。  

![](https://image.3001.net/images/20190113/1547367455_5c3af41f2ffac.png!small)

令我惊讶的原因是因为乘法表达式已执行，以及语法中“$”字符竟然成功执行了表达式。这通常表明在处理表达式时服务器端使用了某种模板引擎。经过一番研究后，我对这台主机可能使用的模板引擎进行了一些猜测。在呈现Yahoo!到目前为止我发现的所有情况之后 ，我收到了雅虎安全团队的确认，使用的模板是Spring Engine Template，他们要求我试着看看我还能做些什么来进一步利用它。由于我只能用这个做基础数学，他们想看看我是否可以从服务器中提取一些数据，以便他们充分评估漏洞影响。最后发现${T(java.lang.System).getenv()} 可用于检索系统的环境变量。  

![](https://image.3001.net/images/20190113/1547367464_5c3af428e99a1.png!small)
然后，我尝试使用.exec方法执行命令“cat etc/passwd”，以使用.exec方法演示能够执行命令从服务器提取数据。payload： ${T(java.lang.Runtime).getRuntime().exec('cat etc/passwd')}，但是该命令原样输出在响应中，并且未执行。我不确定为什么它不起作用，但后来遇到了另一位研究员的博客文章 - [http://deadpool.sh/2017/RCE-Springs/](http://deadpool.sh/2017/RCE-Springs/)，他在那里使用Java的concat()方法来组合与命令中使用的字符相关联的ASCII值。下面的最终payload用于成功执行命令cat etc/passwd：  

```
${T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(99).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(32)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(101)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(99)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(112)).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(119)).concat(T(java.lang.Character).toString(100))).getInputStream())}
```

我不想包含能透露此文件内容的截图。但为了演示，我将使用相同的技术显示执行“id”命令的输出：

![](https://image.3001.net/images/20190113/1547367498_5c3af44a9d347.png!small)

到目前为止，对于雅虎的bug赏金流程我有了很好的经验，并会根据他们处理，解决和评估提交的报告的方式向其他研究人员推荐这个bug提交流程。这个问题的总金额在验证问题时为500美元，在解决后几周为7,500美元，总金额为8,000美元。

谢谢你的阅读。

参考来源
----

[https://nvd.nist.gov/vuln/detail/CVE-2016-4977](https://nvd.nist.gov/vuln/detail/CVE-2016-4977)

[http://secalert.net/#cve-2016-4977](http://secalert.net/#cve-2016-4977)

[http://deadpool.sh/2017/RCE-Springs/](http://deadpool.sh/2017/RCE-Springs/)

自定义white label error页面
----------------------

[https://docs.spring.io/spring-boot/docs/current/reference/html/howto-actuator.html](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-actuator.html)

翻译来源
----

[https://hawkinsecurity.com/2017/12/13/rce-via-spring-engine-ssti/](https://hawkinsecurity.com/2017/12/13/rce-via-spring-engine-ssti/)

***本文作者：生如夏花，转载请注明来自FreeBuf.COM**