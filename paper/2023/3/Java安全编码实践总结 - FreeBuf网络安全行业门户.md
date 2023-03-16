# Java安全编码实践总结 - FreeBuf网络安全行业门户
**Java作为企业主流开发语言已流行多年，各种java安全编码规范也层出不穷，本文将从实践角度出发，整合工作中遇到过的多种常见安全漏洞，给出不同场景下的安全编码方式。** 

本文漏洞复现的基础环境信息：jdk版本：1.8 ，框架：springboot1.5，数据库：mysql5.6和mongodb3.6，个别漏洞使用到不同的开发框架会特别标注。

安全编码实践
------

### Sql注入防范

常见安全编码方法：预编译+输入验证

预编译适用于大多数对数据库进行操作的场景，但预编译并不是万能的，涉及到查询参数里需要使用表名、字段名的场景时(如order by、limit、group by等)，不能使用预编译，因为会产生语法错误。常见的预编译写法如下

jdbc：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/13cbbf33-b5f0-4a4c-b15b-764448ce7e95.jpeg?raw=true)

Hibernate

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/a85cb8a6-2a41-4a88-8320-041b37c42a4c.jpeg?raw=true)

Ibatis

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/b8d50bbc-e578-42c4-a9e8-131e5bb9bb10.jpeg?raw=true)

Mybatis

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/ce7ac4c1-824d-4237-a4fd-68191f77d581.jpeg?raw=true)

在无法使用预编译的场景，可以使用数据校验的方式来拦截非法参数，数据校验推荐使用白名单方式。

错误写法：不能使用预编译的场景（直接拼接用户的查询条件）

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/f9981f53-e063-4936-a867-c75baa8c7687.jpeg?raw=true)

漏洞利用验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/0d970477-09df-4ebd-9153-8695d5d25a12.jpeg?raw=true)

不能使用预编译的正确写法（通过白名单验证用户输入）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/0242415e-9e8e-4f84-9ac9-f882aea84388.jpeg?raw=true)

漏洞修复验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/54b6d8b3-9115-45bc-a156-1bb6ee60b643.jpeg?raw=true)

### Nosql注入防范

涉及到非关系型数据库mongdb在查询时不能使用拼接sql的方式，需要绑定参数进行查询，跟关系型数据库的预编译类似

错误写法(拼接用户的查询条件)：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/db00915a-1222-49e6-9c5d-fdb0ddd5b9bf.jpeg?raw=true)

漏洞利用验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/e7cbe32d-79bd-47dd-b17a-470f844ae3ed.jpeg?raw=true)

正确写法(参数绑定)：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/eefe8c01-1d92-46b8-9ce7-04b688a64e31.jpeg?raw=true)

漏洞修复验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/92ef038f-6ea7-4874-ad2e-a00bbf91fa1d.jpeg?raw=true)

### Xss防范

> 白名单校验
> 
> 适用于纯数字、纯文本等地方，如用户名
> 
> Esapi
> 
> 适用于常规的输入输出，如用户评论

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/59d0766c-b39b-4262-a528-087f1a1e66a7.jpeg?raw=true)

错误写法（对用户输入内容不做处理）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3da62cfb-32a1-4f95-8275-57d82950da1b.jpeg?raw=true)

正确写法（通过esapi的黑白名单配置来实现富文本xss的过滤）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/98b72981-2326-46ec-914a-5f2ec1a96294.jpeg?raw=true)

漏洞利用及修复验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/ea84df1c-077f-4996-87f9-374658190da4.jpeg?raw=true)

### XXE注入防范

为了避免 XXE injections，应为 XML 代理、解析器或读取器设置下面的属性：

> factory.setFeature("[http://xml.org/sax/features/external-general-entities](http://xml.org/sax/features/external-general-entities)",false);
> 
> factory.setFeature("[http://xml.org/sax/features/external-parameter-entities](http://xml.org/sax/features/external-parameter-entities)",false);

如果不需要 inline DOCTYPE 声明，可使用以下属性将其完全禁用：

> factory.setFeature("[http://apache.org/xml/features/disallow-doctype-decl](http://apache.org/xml/features/disallow-doctype-decl)",true);

错误写法(以xmlReader为例，允许解析外部实体)：

> XMLReaderxmlReader = XMLReaderFactory.createXMLReader();
> 
> xmlReader.parse(newInputSource(newStringReader(body)));

漏洞利用验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/5b0ca150-a3f0-4690-9be8-010e9e3e567e.jpeg?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3cd79378-5b72-424b-8cdb-bb252c6caa55.jpeg?raw=true)

正确写法（禁止解析部实体）：

`XMLReaderxmlReader = XMLReaderFactory.createXMLReader();`

`xmlReader.setFeature("[http://xml.org/sax/features/external-general-entities](http://xml.org/sax/features/external-general-entities)",false);`

`xmlReader.setFeature("[http://xml.org/sax/features/external-parameter-entities](http://xml.org/sax/features/external-parameter-entities)",false);`

`xmlReader.parse(newInputSource(newStringReader(body)));`

### 文件上传漏洞

文件名随机，防止被猜解上传路径

限制上传文件大小，防止磁盘空间被恶意占用

限制上传文件类型，防止上传可执行文件

正确写法（限制文件类型大小，通过uuid生成随机文件名保存）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/7f8d411b-224b-499a-8807-27e8370de9b7.jpeg?raw=true)

漏洞利用验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/9b06a238-7c66-407d-9650-07483fe55c11.jpeg?raw=true)

漏洞修复验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/7646a9e6-e58e-4473-9240-80365a2b8dd0.jpeg?raw=true)

### 文件包含

限制文件在指定目录，逻辑名称绑定文件路径，跟文件上传的处理类似，通过文件id读取对应资源文件

错误写法（直接请求用户设置的资源）：

`String returnURL = request.getParameter("returnURL");`

`returnnew ModelAndView(returnURL);`

漏洞利用验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3c295577-6902-4e8c-86d0-60220ce48813.jpeg?raw=true)

正确写法（通过文件id和真实路径的映射设置白名单）：

`if(SecurityUtil.checkid(file_id) ==null) {`

`return"资源文件不存在！";`

`}`

`returnget_file(SecurityUtil.find_path(file_id));`

`}`

文件上传后对应的路径会存储在数据库里，表结构如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/0451a6b2-98dc-45b3-ba34-3f56a55235ff.jpeg?raw=true)

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/770268e8-c169-424a-ae1b-913d8b9a025a.jpeg?raw=true)

Csrf

常见的框架已经自带了防范csrf的功能，只需要正确的配置启用即可

### struts2

JSP使用<s:token/>标签，在struts配置文件中增加token拦截器

页面代码：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/bd9eed4d-f242-4d6a-9345-3fea3d0c596f.jpeg?raw=true)

漏洞修复验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/c6e56e8a-0cc0-49ea-8e3b-c28b99ce515a.jpeg?raw=true)

### Spring

正确写法：使用spring-security

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/43ae0d94-d616-4d92-92b3-e05d5e8f3e6a.jpeg?raw=true)

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/be7e2e1f-b4ac-4107-b14d-79bea1ef2619.jpeg?raw=true)

### 越权

Java通用权限框架(shiro)

进行增删改查操作时采用无法遍历的序号

对于敏感信息，应该进行掩码设置屏蔽关键信息。

垂直越权

角色权限矩阵

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/c14cdabc-8151-4d73-8f3f-438ea31243c2.jpeg?raw=true)

限制匿名用户和低权限用户，执行操作前检查用户登录状态和权限清单

正确写法（判断用户权限清单是否包含请求的权限）：

![](https://image.3001.net/images/20200614/1592133424_5ee60730055c8.png!small)

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/bef98a86-f50e-4338-8188-13974bec80c0.jpeg?raw=true)

水平越权：

操作前判断下当前用户是否有对应数据权限，修复后修复前两次验证，通过返回长度不同可看到水平越权问题已解决。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/05177aeb-3ba6-40b7-8c38-8a9ba91f733c.jpeg?raw=true)

### url重定向&ssrf

url重定向

对于白名单内的地址，用户可无感知跳转，不在白名单内的地址给用户风险提示，用户选择是否跳转

正确写法：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/f8cbe382-325a-4af5-a679-6862a71c07b2.jpeg?raw=true)

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/debe86f5-bf22-482a-a989-5dd1a2831b7e.jpeg?raw=true)

Ssrf

漏洞利用验证：

![](https://image.3001.net/images/20200614/1592133787_5ee6089b3afc1.png!small)

正确写法（限制请求协议，设置白名单域名，避免内网地址探测）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/b2971c80-f88f-405f-baa4-cbccf077dba7.jpeg?raw=true)

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/e44693a9-9a05-46b6-b9d4-610e6eb0108f.jpeg?raw=true)

### 拒绝服务

正则表达式拒绝服务，这种漏洞需要通过白盒审计发现，黑盒测试比较难发现。

错误写法（正则匹配时未考虑极端情况的资源消耗）

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/eca1c6ed-8b2f-4f15-8ce1-f9f153cb62a6.jpeg?raw=true)

漏洞利用验证，随着字符长度增加，响应时间会越来越长，cpu满负荷运转

![](https://image.3001.net/images/20200614/1592134355_5ee60ad364a2b.png!small)

正确写法（运行超过2秒就中止匹配）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/db3546f7-b02f-49df-9fcf-57d7f4dcbcdd.jpeg?raw=true)

漏洞修复验证：

![](https://image.3001.net/images/20200614/1592134454_5ee60b36ac622.png!small)

### 不安全的加密模式

需要通过白盒审计发现漏洞，直接黑盒测试比较难。

错误写法：使用ECB模式，相同明文生成相同密文

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/246b0aca-f043-4800-afb6-00f78c64283e.jpeg?raw=true)

漏洞利用验证（使用选定明文攻击从后向前按位猜解）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/709c8629-3d9c-4f1b-93de-29fdaa551e97.jpeg?raw=true)

正确写法：使用CBC模式，相同明文生成不同密文

```
Cipher cipher =Cipher.getInstance("AES/CBC/PKCS5Padding");
```

### 不安全的随机数

需要通过白盒审计发现漏洞，直接黑盒测试比较难。

错误写法：使用伪随机，相同种子生成相同随机数序列

漏洞利用验证：

需要通过java生成前后2000毫秒内的随机数，然后使用python调用这些随机数尝试暴破

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/6ac57a57-bc00-4917-a3ce-85f75438fe5e.jpeg?raw=true)

正确写法：使用Securerandom

漏洞修复验证（Securerandom不能指定seed，避免伪随机）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3f255b53-0a3b-437d-82d8-f1e6fdd6200d.jpeg?raw=true)

### 条件竞争

Servlet的单例模式容易导致条件竞争，也是推荐白盒方式审计漏洞。

错误写法：初始积分100个，每天限制签到1次领取1积分

![](https://image.3001.net/images/20200614/1592134753_5ee60c6168931.png!small)

漏洞利用验证（10个并发可实现多次签到，这里多并发跟业务功能复杂度和服务器性能有关，如果想必现漏洞，可以在读取签到次数和增加签到次数之间增加2秒延时，可以保证漏洞复现。）

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/79ec7adc-fd91-4be9-b8cb-1314885fd96d.jpeg?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/603319a6-396d-42b0-abd5-198b9ab9f23d.jpeg?raw=true)

正确写法：使用线程同步

![](https://image.3001.net/images/20200614/1592134792_5ee60c88edc36.png!small)

漏洞修复验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/bef8977b-b3e1-4976-97ae-c5124947a8ef.jpeg?raw=true)

修复后返回数据包速度明显变慢，不能再重复签到领积分

![](https://image.3001.net/images/20200614/1592134839_5ee60cb717c0c.png!small)

### 日志伪造防范/http响应拆分防范

日志伪造黑盒测试无法发现，需要通过白盒审计发现漏洞。

错误写法（直接将用户的输入打印到日志文件）：

`public voidlog_forging(HttpServletRequest request,HttpServletResponse response)`

`throwsException {`

`logger.info("user: "+request.getParameter("name")+" received 100$;");`

`}`

漏洞利用验证（通过%0d%0a插入换行控制符，伪造日志记录）

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/01a9850e-2368-4a0f-ac82-0e95677ad973.jpeg?raw=true)

正确写法（过滤换行空格）：

`public voidlog_forging_sec(HttpServletRequestrequest, HttpServletResponse response)`

`throwsException {`

`Pattern p = Pattern.compile("\\s*|\t|\r|\n");`

`Matcher m =p.matcher(request.getParameter("name"));`

`String name= m.replaceAll("");`

`logger.info("user: "+name+" received 100$;");`

`}`

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/4e97cd58-0de8-4eae-a344-f858b3fabdcc.jpeg?raw=true)

http响应拆分，只在低版本web服务器上出现，使用tomcat9未复现这个问题

错误写法

`@RequestMapping("/http_splitting")`

`@ResponseBody`

`public voidhttp_splitting(HttpServletRequest request,HttpServletResponse response)`

`throwsException {`

`Stringaddheader=request.getParameter("addheader");`

`response.addHeader("addheader", addheader);`

`}`

漏洞修复验证(新版本的web服务器可以自动处理http响应拆分)：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/86f1dda9-db58-4ef3-89de-e913b4544b35.jpeg?raw=true)

动态代码执行
------

Runtime.exec

错误写法(直接执行用户输入的命令)：

Process p = run.exec(cmd);

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/2904e9c2-95c2-4a7f-86dd-c64a204b2d8a.jpeg?raw=true)

正确写法：

1.输入净化

2.Switch-case命令映射

`if(!cmd.equals("xxx")){`

`return"command "+cmd+" not allowed!";`

`}`

3.使用语言标准api获取系统信息

"当前用户:"+System.getProperty("user.name")

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/64e1c2b0-f023-4f79-ac3a-1db0d5a6f9a2.jpeg?raw=true)

### 反序列化

错误写法（对序列化的类未做限制）：

> Stringdata=request.getParameter("data");
> 
> byte\[\] decoded = java.util.Base64.getDecoder().decode(data);
> 
> ByteArrayInputStream bytes =newByteArrayInputStream(decoded);
> 
> ObjectInputStream in =newObjectInputStream(bytes);
> 
> in.readObject();
> 
> in.close();

漏洞利用验证

![](https://image.3001.net/images/20200614/1592134942_5ee60d1e4b97d.png!small)

正确写法(使用serialkiller，主要也是通过黑名单去过滤，可以防御大部分的攻击)

> String data =request.getParameter("data");
> 
> byte\[\] decoded = java.util.Base64.getDecoder().decode(data);
> 
> ByteArrayInputStream bytes =newByteArrayInputStream(decoded);
> 
> try{
> 
> ObjectInputStream in =newSerialKiller(bytes,"d:\\\serialkiller.conf");
> 
> in.readObject();
> 
> in.close();
> 
> }catch(InvalidClassException e){
> 
> response.getWriter().write("class not allowed!");
> 
> }

漏洞修复验证

![](https://image.3001.net/images/20200614/1592134953_5ee60d2910283.png!small)

### 表达式注入

Spel表达式注入

错误写法（直接解析表达式）：

> public voidspel_injection(HttpServletRequest request,HttpServletResponse response)
> 
> throwsException {
> 
> A a=newA("test");
> 
> ExpressionParser parser =newSpelExpressionParser();
> 
> StandardEvaluationContext context =newStandardEvaluationContext();
> 
> parser.parseExpression("Name = "+request.getParameter("name")).getValue(context, a);
> 
> response.getWriter().write("usrname:"+a.getName());
> 
> }

漏洞利用验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/cfb3e479-a57c-440a-a9b5-3c154205f164.jpeg?raw=true)

正确写法(使用SimpleEvluationContext，可解析白名单内的表达式)：

> public voidspel\_injection\_sec(HttpServletRequestrequest, HttpServletResponse response)
> 
> throwsException {
> 
> A a=newA("test");
> 
> ExpressionParser parser =newSpelExpressionParser();
> 
> EvaluationContext context =SimpleEvaluationContext.forReadWriteDataBinding().build();
> 
> parser.parseExpression("Name = "+request.getParameter("name")).getValue(context, a);
> 
> response.getWriter().write("usrname:"+a.getName());
> 
> }

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/8a6fd211-4b90-49be-af1e-28a269037830.jpeg?raw=true)

OGNL表达式注入涉及到struts框架的安全配置，主要是禁用动态方法调用，不再继续展开验证。

### Spring-boot安全配置

1.Spring Boot 应用程序配置为显式禁用Shutdown Actuator：endpoints.shutdown.enabled=false避免应用被非法停止

2.删除生产部署上的 spring-boot-devtoos依赖关系。

3.不要远程暴露mbean spring.application.admin.enabled=false

4.启用html自动转义

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/b7235855-74ea-40d4-b7f3-eee72a4df01e.jpeg?raw=true)

5.使用默认http防火墙StrictHttpFirewall

6.Spring Security身份认证配置,该配置默认为拒绝对之前不匹配请求的访问：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3222c05b-0bf3-4550-aebe-afc404aedff9.jpeg?raw=true)

7\. 禁用 SpringBoot ActuatorJMX

`endpoints.jmx.enabled=false`

`management.endpoints.jmx.exposure.exclude=*`

8\. Spring Boot Actuator 开启security功能

错误配置：

```
management.security.enabled=false
```

漏洞利用验证：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/732a50ee-aa71-4146-bfec-d2f22769fe28.jpeg?raw=true)

正确配置：

```
management.security.enabled=true
```

漏洞修复验证

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/aaec6452-6688-4c94-a569-2fd3926d42d8.jpeg?raw=true)

总结
--

作为安全人员经常会被开发问如何修复漏洞，开发需要具体到某行代码如何改动，通过对常见漏洞的复现利用以及安全编码实践，可以加深安全人员对相关漏洞原理的理解，根据业务需要更具体地帮助开发人员写出健壮的代码，预防或修复安全漏洞。

参考资料
----

> [https://github.com/JoyChou93/java-sec-code/](https://github.com/JoyChou93/java-sec-code/) 前面的常见注入类漏洞参考了这里的代码。
> 
> [https://vulncat.fortify.com/en](https://vulncat.fortify.com/en) 后面的不安全加密模式，不安全随机数等配置漏洞参考fortify官方的漏洞知识库。