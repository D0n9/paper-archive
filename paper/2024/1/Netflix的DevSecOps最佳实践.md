# Netflix的DevSecOps最佳实践
*   应用安全
    

*   早期的安全工作
    
*   DevSecOps
    
*   沟通和协作
    
*   虚拟安全团队
    

*   云上安全
    

*   安全隔离原则
    
*   移除静态密钥
    
*   凭证管理
    
*   适当的权限划分
    
*   混沌工程在安全的使用
    

*   入侵感知
    

*   异常模型
    
*   防ssrf获取凭据
    

*   办公网安全
    

*   员工入职
    
*   BeyondCorp
    

Netflix这家公司作为科技股新贵FAANGS里的当红辣子鸡，市值是5.33个百度，国内竟然鲜有文章谈论这家公司的安全机制，只有少量的报道说到了混沌工程，谈到了14年Netflix推出了Security Monkey，使之可以记录并标记一个帐号的修改，同时对配置进行审核，确保符合AWS云上的安全标准。![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/df68a1fe-7553-4bcf-9ade-7a1bd58cd5ab.jpeg?raw=true)

下面跟随笔者，一起看下国外的互联网公司如何筚路蓝缕，随着时间的推移不断迭代改进安全状况，系统地实现了安全性。

应用安全
----

NetFlix应用安全团队总结出来一套如何通过默认安全，自动化工具以及建立业务团队的长期合作关系来扩展其安全工作的方法。

### 早期的安全工作

最早Netflix安全团队也和我们大多数一个人的安全部一样，创业伊始，安全问题颇多，只能随机找项目去挨个审计评估，找到问题提工单要求修复。这样的带来的问题显而易见：漏洞通常不能闭环解决，基于事件的运作方式导致安全和开发团队经常有冲突，只能奔走于修复单个错误，不能系统地提升安全性改进。开发团队也为此疲劳奔命，收到的不同来源的每个工单都是紧急高优先级的，打乱了开发计划。

### DevSecOps

最好的起步阶段是同业务团队建立合作关系。Netflix的安全团队与开发团队密切合作协助评估安全状况，协商划定和记录未来的重点安全投入领域，制定可能需要花费几个季度才能实现的的战略性安全改进计划，而不仅仅是为开发团队提供一个漏洞列表：）

#### 安全组件支持

安全的类库、组件，相关工具和安全服务化帮助应用程序开发起来既更高效，又安全。例如使用标准的身份验证库不仅得到了安全加固，而且易于调试和使用，也有出色的日志记录功能，并为开发人员（和安全团队）提供了有关登录角色的标注数据，甚至为使用者可以得到安全技术背后开发团队的双份技术支持。

#### 工具和自动化

同大家的共识一样，应用安全团队在漏洞扫描、管理以及程序清单和风险分类环节中大量使用了工具和自动化。

这些安全信息的目的是为了提供有价值的数据和背景知识，帮助安全团队了解应用程序的风险现状、加固的目的等，从而能够提出更好的安全建议。

##### 应用安全风险大盘

以部门维度列出出哪些提供外网访问服务的实例 显示应用已采用的安全控制措施，如SSO、使用了mTLS进行传输层加密和加密存储机制，以及是否填写了安全问卷(Zoltar)

##### 应用风险评分机制

安全团队可以通过多维度的计算。应用是否对外提供服务、有没有运行在旧版操作系统或镜像上、使用的安全框架组件里的哪一部分、有多少运行实例、是否运行在与合规性相关的AWS帐户(如PCI)中。当然读者们可以看出这个自动化风险评分指标有些武断，可以改进。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/02d52cf6-c621-4402-a55c-3a0b97d94fe9.jpeg?raw=true)

打分列表详情

##### 漏洞扫描

漏扫工具是Netflix在AppSec USA 2016上开源的Scumblr，github地址在 https://github.com/Netflix-Skunkworks/Scumblr 。此后为了内部使用进行了大量更改，通常用于对代码库运行小型、轻量级的安全检查，或对线上实例运行简单的检查。

##### 安全指导调查表

类似于表格版本的威胁建模工具，跟踪不同应用程序的预期需求和其他难以工具自动检查的方面。当开发者填写问卷时，他们会得到针对他们所使用的语言和框架的更定制的安全建议，这使得开发团队能够专注于能够真正降低风险的事情，一切自助化。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/67ae3bb7-522f-4286-9b6b-d4b0170890a3.jpeg?raw=true)

指导表的详情

##### 应用程序清单

不同于传统的就有软件、主机的资产大盘，该列表不仅包括代码文件，还包括AWS账户，IAM，负载均衡信息以及与应用程序相关的所有其他信息，例如所有权和组织架构信息。目的是业务可以更方便的接入安全服务，和更快验证是否使用安全控件，以及发现未知角落的资产。当然最大的用处是在出现安全事件时，更快地定位到责任人。

##### 安全大脑

这个项目直观向开发团队展示了名下自动分配给每个应用程序的风险、当前发现的漏洞以及应该实现的最有效的安全控制/最佳实践。

security Brain专门指导开发团队现在应该关注的最大的“需要理解解决”任务项。基本上是威胁建模处理的结果和漏洞清单。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/55ae1b9c-9ec1-4ee9-84cc-47ada7f39dc5.jpeg?raw=true)

安全大脑的界面

### 沟通和协作

DevSecOps不是安全甩锅责任给业务，而是向业务翻译清楚安全协助要达成的目标是什么，业务要使用哪些专业手段，来多快好省地达成愿景。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/0bd15fbf-51cc-4456-b8b3-faf539a0c96b.jpeg?raw=true)

合作制定的安全计划

### 虚拟安全团队

内部组件的云安全卓越小组有什么特征和职责呢？首先人数控制到10个人以下，包含不同的部门成员，有产品经理，架构师，基础架构工程师，安全工程师，运维和开发工程师等；其次，该虚拟团队以降低和消除DevSecOps转型的风险为己任，全职投入；最后，该团队要负责制定安全合规相关的原则，流程，可动手实现安全相关自动化工具，培训和影响其他团队采用最佳的安全实践，制定和指导安全基线。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/f6ea88db-ec1b-4af3-a1a9-ce7631fcf072.jpeg?raw=true)
懂Netflix人，会知道团队背后也是有苦衷的，在自由与责任的企业文化中，他们认为处在一个创新市场，所有高效能员工很少犯错，犯了错，快速修正就好了......

云上安全
----

上面虽然提到内部研发人员可以使用一系列的开发工具和基础架构实现安全性，但是云上总会出现意外情况，云安全很重要，Netflix如何在不投入更多的云原生安全开发资源的限制下，做到云上安全性呢？

### 安全隔离原则

职责分离:安全团队将把高级用户限制在自己的AWS子帐户中，这样他们的（凭据风险）就不会影响生态系统的其他部分。

区分敏感应用程序和数据：不是所有的环境、应用或数据都是同等重要，理解哪些是重要的并定义恰当的优先顺序，理解不同业务组织的风险偏好；只有有限的用户可以访问这些应用程序和数据。

通过开发针对AWS帐户的系列工具来减少风险：如果要做帐户级别的自动化划分，Netflix云安全团队在这些领域投入了大量资金自研。

### 移除静态密钥

静态密钥如ak、sk、token不会过期，并且会导致很多问题，比如当git repos中的AWS密钥泄露在GitHub时.....想想一下，开发一个github自动监控系统真的有用吗？NetFlix的做法是通过为每个应用程序提供一个角色来实现这一点，然后EC2元数据服务为该角色提供短期凭据,类似于STS机制。

### 凭证管理

移除还不够，之前是开发人员ssh到机器上访问凭证，或者使用亚马逊的api来获取，这样没有办法进行监控。具体的凭证管理是构建了一项服务称为ConsoleMe，用户可以使用SSO或CLI通过Web界面请求凭据处理创建，修改和删除AWS凭证，集中进行**审核**和**记录**对云账户的所有访问。

**该策略将IP限制凭据限制**到请求者所连接的VPN，因此即使凭据意外泄漏，它们也不会起作用。

另一个有限的策略是设置凭据的有效期是一个小时，从根本上减少暴露时间。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/9b154e77-ef52-4796-b315-466cb20cdc24.jpeg?raw=true)

架构设计

### 适当的权限划分

而且如果开发人员离开公司或跳槽到其他公司，则很容易导致其维护开发的服务被遗忘，云上环境经常会有这样的风险，然而这些应用程序可能已获得敏感的数据操作权限。安全团队提供了一个AWSIAM顾问工具Aardvark https://github.com/Netflix-Skunkworks/aardvark 协助用户自助设定恰当的权限，定期访问AWSIAM的Access API更新并保存用户的访问行为 Netflix通过RepoKid降低了这种风险。github代码见https://github.com/Netflix/repokid Netflix的新上线应用程序被授予基本的AWS权限集。使用aws提供的的Access Advisor和CloudTrail作为数据源，收集有关应用程序行为的数据，并自动删除AWS权限，如果检测到故障，会自动回滚。

### 混沌工程在安全的使用

Netflix是一家严重依赖于AWS的线上服务公司，2011年开发了监控AWS账户的安全策略测试。可以认为是云上配置的黑盒扫描器，类似于aws的cloudwatch或阿里云的操作审计+配置审计，提供一个统一云环境安全合规监控和分析的框架，Web 应用的漏洞扫描，自动化监控SSL证书和秘钥有效不过期，检查防火墙、用户、用户组和权限策略等安全配置发现违规和漏洞，并终止有问题实例。主要有以下功能：

*   针对aws云上资源的api，尝试进行恶意调用，和注入测试；
    
*   展示、通知、记录发现的风险项给内部响应团队
    
*   维护历史的各项配置
    
*   支持创建各项新规则
    
*   支持NetFlix多种账户体系
    
*   使用 Safestack AVA、Metasploit、AttackIQ和 SafeBreach等工具，尝试主动攻击系统
    

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/f712f042-0a75-4a75-92bf-ac02820bd53f.jpeg?raw=true)

目标是保证安全措施的有效性，抗攻击的韧性。

入侵感知
----

在云上攻防中，经常有一个ssrf或者rce可以访问元数据接口获取凭据，利用这个凭据来访问s3 bucket，操作iam，AWS提供的GuardDuty服务仅仅可以检测何时在AWS外部使用实例凭证，而不是从攻击者在AWS内的操作中检测。NetFlix使用了10万+云主机实例，如何感知凭证泄露呢？有两个最佳实践：

### 异常模型

攻击者一般会使用自动化的枚举脚本爆破，尝试调用aws提供的各个特权api，借助于后端的审计，一旦访问一个未使用的服务，安全团队就会得到警报。

### 防ssrf获取凭据

最简单粗暴的办法是waf拦截防止aws的http://169.254.169.254  这个请求的访问，该高危接口可以获得到了的云主机信息。有没有更优雅的办法呢？答案是通过感知元数据请求的User Agent来实现。由于攻击者构造ssrf的UA一般是不可控的，而正常基于AWS SDK的请求UA是固定的，比如ruby是“aws-sdk-ruby3/3.54.2 ruby/2.5.5 x86_64-darwin18 aws-sdk-batch/1.20.0”。而ssrf漏洞，ua一般会是curl或者java client，这是感知的关键。

办公网安全
-----

Netflix在对待人的因素上十分人性化，因为他们认为员工应该像在学校不但学习一样工作着。所以安全措施要尽量避免强制性漏洞更新、全员邮件、消息提醒轰炸，反而尽量重视培训、自助服务、重视反馈这些细节。

### 员工入职

新领到电脑时，就有提示指导如何进行安全设置。辅助严格的新员工培训。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/e31f3765-827a-4d81-ba2d-ed6c449c9471.jpeg?raw=true)

大大的标签进行引导

### BeyondCorp

使用新设备连入网络时，数据来源是windows：landesk，mac ：jamf，Linux：OSquery，移动设备：Google MDM进行系统自检，详细检查项包括：

1.  磁盘是否启动加密配置
    
2.  防火墙是否开启
    
3.  系统自动更新是否开启
    
4.  补丁是否更新
    
5.  自动锁屏是否开启
    
6.  没有root、越狱
    
7.  EDR产品carbon black 是否安装
    

后台记录访问的安全性。登入每个新的appkey应用时，porta自动判断风险因素l会弹出提示，是否是本人登入。![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/005bc37e-fc7a-4c93-9402-acb57bc380a8.jpeg?raw=true)

参考资料：

1.  https://tech.qq.com/a/20171224/001288.htm
    
2.  https://www.slideshare.net/RyanHodgin/netflix-security-monkey-overview
    
3.  https://github.com/aws/aws-sdk-ruby/issues/1363
    
4.  https://github.com/Netflix-Skunkworks/aws-credential-compromise-detection
    

[

DevSecOps在携程的最佳实践





](http://mp.weixin.qq.com/s?__biz=MzA5Mzg3NTUwNQ==&mid=2447804648&idx=1&sn=22b627a68e75806515f44d35bfeaafba&chksm=844514b6b3329da015db9d442127ee475b9376b3a98e46d8c43a1b71c766c58a192505d082a2&scene=21#wechat_redirect)

[](http://mp.weixin.qq.com/s?__biz=MzA5Mzg3NTUwNQ==&mid=2447804624&idx=1&sn=8fe5ecd6aff6078428b7c4dbd6abf0f1&chksm=8445148eb3329d98d438422ea60f2f0ef4ecf24f0135531569db5148a3ee947a011ab37fe8cc&scene=21#wechat_redirect)

[](http://mp.weixin.qq.com/s?__biz=MzA5Mzg3NTUwNQ==&mid=2447804624&idx=1&sn=8fe5ecd6aff6078428b7c4dbd6abf0f1&chksm=8445148eb3329d98d438422ea60f2f0ef4ecf24f0135531569db5148a3ee947a011ab37fe8cc&scene=21#wechat_redirect)

[从SDL到DevSecOps：腾讯云是如何更早地收敛安全漏洞的？](http://mp.weixin.qq.com/s?__biz=MzA5Mzg3NTUwNQ==&mid=2447804624&idx=1&sn=8fe5ecd6aff6078428b7c4dbd6abf0f1&chksm=8445148eb3329d98d438422ea60f2f0ef4ecf24f0135531569db5148a3ee947a011ab37fe8cc&scene=21#wechat_redirect)

