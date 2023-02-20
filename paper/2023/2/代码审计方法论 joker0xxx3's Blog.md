# 代码审计方法论 | joker0xxx3's Blog
发表于 2021-07-05 | 分类于 [代码审计](https://joker-vip.github.io/categories/%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1/)

[](#代码审计方法论 "代码审计方法论")代码审计方法论
-----------------------------

[](#1、介绍 "1、介绍")**1、介绍**
------------------------

代码审计(CodeAudit) 就是根据了解到的程序功能和关键业务，针对程序源代码逐条进行检查和分析，找出源代码中可能引发的常见安全漏洞和业务逻辑问题的缺陷，并提供代码修改建议或解决方案。

针对于源代码审计工作环境的部署，其实如果是以服务的方式为客户的代码进行审计，审计的环境一般都会相对复杂，会面临几种情况：

1.  最理想的环境，客户提供代码同时有对应的测试环境，这样的话就可以黑盒+白盒进行审计，可以提高审计的效率和覆盖度。
2.  只提供项目源代码，需要自己梳理整个项目源码的架构，通读所有关键代码，相对时间成本比较大，但是可以彻底了解整个项目源代码并且挖掘出高质量漏洞。
3.  只提供源代码片段，这种情况首先需要跟客户沟通是否可以提供完整项目源代码，否则是在一定程度影响源代码审计的完整性，因为部分功能代码片段可能会找不到调用的接口函数，无法追踪业务逻辑代码。

所以客户的代码审计服务基本上没有搭建环境的情况，因为他们不会提供数据库，想跑也跑不起来。

[](#2、源代码审计前期准备 "2、源代码审计前期准备")2、源代码审计前期准备
-----------------------------------------

1.  需要和客户确认最新源代码版本
2.  需要客户准备入场后代码审计环境，是我们自备电脑审计还是在客户专用的电脑上做审计
3.  如果是专用的电脑，需要在电脑上安装office或者wps、代码编辑器及IDE工具，同时部署代码审计工具
4.  需要提供被审计系统相关需求文档及设计文档，帮助审计人员了解业务，可以深入业务进行审计
5.  需要跟开发团队提前打好招呼，为审计人员讲解代码结构，可以方便审计人员迅速进入审计工作中

**常用源代码审计相关工具如下**

### [](#2-1-自动化工具 "2.1 自动化工具")2.1 自动化工具

Fortify、CheckMarx、Sonarqube

### [](#2-2-辅助工具 "2.2 辅助工具")2.2 辅助工具

Sublime text、Notepad++、EclipseIDE、JetBrain(CLion、IDEA、PyCharm、GoLang、PhpStorm…)

[](#3、审计流程 "3、审计流程")**3、审计流程**
------------------------------

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/12289087-5aec-44b3-8e38-8796a4b7a6d2.png?raw=true)
](https://image.3001.net/images/20210708/1625755871425.png)

[](#4、人工审计思路 "4、人工审计思路")**4、人工审计思路**
------------------------------------

### [](#4-1-正向数据流分析法-根据业务推代码 "4.1 正向数据流分析法-根据业务推代码")**4.1 正向数据流分析法-根据业务推代码**

**描述**

寻找敏感功能点，通读功能点代码；  
正向审计即从功能入口点进行跟踪，一直到数据流处理结束；

**优点**

精准定向挖掘，利用程度高；对程序有整体把握，能较全面的覆盖各种不同类的功能点，能挖掘到有价值的逻辑漏洞，有助于高质量的交付。

**缺点**

命名不规范的代码容易被忽略，导致失去先机；  
慢，易迷失在海量代码之间；  
难，绕不过去的框架问题。

**针对性漏洞**

登录，找回密码，文件上传，任意文件下载，验证码漏洞，流程绕过，越权等无明显特征等漏洞。

**数据流分析流程图**

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/d8af51be-fefc-4109-a8c6-ac1643a41399.png?raw=true)
](https://image.3001.net/images/20210708/16257558746870.png)

#### [](#常见的不可信数据入口方法 "常见的不可信数据入口方法")**常见的不可信数据入口方法**

_JAVA_

| 方法 | 说明 |
| --- | --- |
| getParameter | request类获取参数方法 |
| getParameterNames | 获取参数名 |
| getParameterValues | 获取参数值 |
| getParameterMap | 获取参数map类型 |
| getQueryString | 获取URL的value值 |
| getHeader | 获取http请求头 |
| getHeaderNames | 获取请求头名 |
| getRequestURI | 获取请求URL |
| getCookies | 获取cookie |
| getRequestedSessionId | 获取sessionid |
| getInputStream | 获取输入数据 |
| getReader | 获取请求内容 |
| getMethod | 获取请求方法 |
| getProtocol | 获取请求协议 |
| getServerName | 获取服务名 |
| getRemoteUser | 获取当前缓存的用户 |
| getUserPrincipal | 获取用户指纹 |

_PHP_

| 方法 | 说明 |
| --- | --- |
| $_GET | GET方法请求参数获取 |
| $_POST | POST方法请求参数获取 |
| $_COOKIE | 获取请求中cookie |
| $_REQUEST | $_REQUEST可以获取以POST、GET方法提交的数据 |
| $_FILES | POST方法上传文件 |
| $\_SERVER\[‘REQUEST\_METHOD’\] | 访问页面时的请求方法 |
| $\_SERVER\[‘QUERY\_STRING’\] | 查询(query)的字符串 |
| $\_SERVER\[‘HTTP\_ACCEPT’\] | 当前请求的Accept:头部的内容 |
| $\_SERVER\[‘HTTP\_HOST’\] | 当前请求的HOST:头部的内容 |
| $\_SERVER\[‘HTTP\_REFERER’\] | 当前请求的HOST:头部的内容 |
| $\_SERVER\[‘HTTP\_USER_AGENT’\] | 当前请求的HOST:头部的内容 |

#### [](#常见的不可信文件访问方法 "常见的不可信文件访问方法")**常见的不可信文件访问方法**

_JAVA_

| 方法 | 说明 |
| --- | --- |
| java.io.FileInputStream | 文件输入 |
| java.io.FileOutputStream | 文件输出 |
| java.io.FileReader | 文件读取 |
| java.io.FileWriter | 文件写入 |

_PHP_

| 方法 | 说明 |
| --- | --- |
| file\_get\_contents | 文件输入 |
| file\_put\_contents | 文件输出 |
| fread | 文件读取 |
| fwrite | 文件输出 |

### [](#4-2-逆向数据流分析法-根据缺陷推业务 "4.2 逆向数据流分析法-根据缺陷推业务")**4.2 逆向数据流分析法-根据缺陷推业务**

**描述**

根据敏感关键字回溯参数传递过程；  
逆向审计即先根据一些关键词搜索，定位到可能存在风险的关键词/函数，再反推到功能入口，看整个处理过程是否存在漏洞。

**优点**

针对性强，通过搜索敏感关键字可快速定位可能存在的漏洞，可快速、高效、定向的挖掘高质量的漏洞。

**缺点**

对程序整体了解不够深入，在漏洞定位时比较耗时，覆盖面不够全面，对逻辑漏洞往往覆盖不到。

**针对漏洞**

SQL注入，命令注入，反序列化等特征明显的漏洞。

针对常规漏洞（指纹识别度高）：如SQLi、CMDi、线程安全等  
Ibatis、Mybatis：使用xml做SQL映射，搜索$、${、+可以定位  
Hibernate：代码中的HQL，搜索字符串拼接的关键字Concat、append、+等

**数据流分析流程图**

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/691824ec-334b-448e-a8c7-ea2b24847f02.png?raw=true)
](https://image.3001.net/images/20210708/16257558812248.png)

**危险函数/关键字**

| 漏洞 | 方法 |
| --- | --- |
| SQL | Statement, PreparedStatement,sql,$,hql |
| XXE | SAXReader,DocumentBuilder,XMLStreamReader |
| FILE | MultipartFile,createNewFile,FileInputStream |
| SSRF | HttpCilent,URL,ImageIO,HttpURLConnection,OkHttpCilent |
| EXEC | getRuntime.exec,processBuilder.start |

### [](#4-3-总结 "4.3 总结")**4.3 总结**

**正向数据流分析法**

优点：快速开展，快速定位，有效挖掘严重漏洞。  
缺陷：可能遗漏某些隐藏接口/url

**逆向数据流分析法**

优点：较为全面的对所有接口/url进行跟踪审计  
缺点：比较耗时，审计到可能无法有效利用的漏洞

**建议优先关注的漏洞类型**

命令执行 > 代码执行 > 文件操作(上传/包含/下载) \> SQL注入 > 逻辑漏洞> SSRF > XSS>CSRF>XXE>反序列化

* * *

参考链接：

[https://www.yuque.com/haibei-a8jfo/vsoi0z/efynxz](https://www.yuque.com/haibei-a8jfo/vsoi0z/efynxz)

[https://www.cnblogs.com/afanti/p/13156152.html](https://www.cnblogs.com/afanti/p/13156152.html)

[https://mp.weixin.qq.com/s/DfgAdzpyZCRIZTsfzE8A4A](https://mp.weixin.qq.com/s/DfgAdzpyZCRIZTsfzE8A4A)