# 蓝军视角：阿里云 RCE 战火余烬下的启示
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/7238d424-d5a3-481e-a6e4-4f4511b09937.png?raw=true)

2023年4月19日，Wiz Research 在文章 Accidental ‘write’ permissions to private registry allowed potential RCE to Alibaba Cloud Database Services 中披露了被命名为BrokenSesame的一系列阿里云数据库服务漏洞，向我们展示了如何从一个容器逃逸漏洞，与私有仓库写权限的组合，最终实现RCE的攻击链路。该漏洞最终可导致未授权访问阿里云客户的PostgreSQL数据库，并且可以通过在阿里巴巴的数据库服务执行供应链攻击。

时隔一个月，在经过研究与复现的过程中，不由得感叹攻击者的构思巧妙；而反过来作为防守人员，我们能否在这次战役中，获得一些经验和启示？让我们站在蓝军的视角再看一遍完整的攻击路程。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/25f75c2c-70f4-439a-ae51-bc6a7cb4576d.png?raw=true)

容器提权

在原文中，作者分享了两个案例：ApsaraDB RDS for PostgreSQL 和 AnalyticDB for PostgreSQL。两个案例的第一步均为容器提权：从普通账户提权至更高的权限。在这一步中，两个案例分别用到了不同的攻击链路：

*   cron定时任务'/usr/bin/tsar' -> 高权限执行的二进制文件 -> 可修改的动态链接库 -> 覆盖链接库 -> 定时任务出发执行获取root权限。
    
*   容器共享目录 -> 业务特性导致任意文件读取（符号链接）-\> 获取到另一个容器的读取下权限。
    

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/5db105db-a6aa-45d5-8011-a11ee02789f8.png?raw=true)

在这两个链路中，链路1实际上是在渗透过程中最常使用的一种攻击方式，通过注入恶意动态链接库实现账户提权。这条链路涉及到了两个关键的问题点：cron定时任务的启动权限和动态链接库权限。cron的高权限导致所有定时任务都通过高权限账户root来执行，而可修改的动态链接库权限导致覆盖动态链接库；

针对这两个风险点，传统的HIDS文件监控即可覆盖到该层面。除此之外，对于添加进入定时任务的二进制程序，应该严格限制其权限，包括动态链接库文件。

在第二条链路中，我们发现实际上业务在设计架构模式上时，使用了共享容器的目录来实现通信；但业务代码并没有考虑符号链接的场景，导致了第二个文件下的任意文件读取。

整体来看，链路1和链路2分别属于的HIDS主机监控/黑白盒业务安全扫描的范畴，说明即使在云场景下，传统的业务安全依旧处于重要地位。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/8db2b09c-8137-4831-8cdb-19a6fb3a9411.png?raw=true)

容器逃逸

获取到容器root权限后，下一步便是朝着宿主机进行攻击，同样是两条链路：

*   共享pid namespcae -> 监听发现共享的挂载目录'/home/adbpgadmin' -> 植入ssh/config获取容器B权限 -> 通过容器B的docker.sock 逃逸至宿主机。
    
*   由提权过程中的任意文件读取获取到业务代码 -> 代码审计发现命令注入 -\> 通过命令注入获取到特权容器的shell -> 通过core_pattern实现容器逃逸。
    

在这两条链路中，攻击者使用了两种不同的逃逸方式：共享namespace导致攻击者获取到更多信息、挂载docker.sock导致逃逸、特权容器复写core_pattern导致逃逸；这几种方式是云环境中比较经典的逃逸场景，使得容器绕过各种隔离限制，对容器外的宿主机或其他容器的资源进行操作。

如何对容器逃逸进行防御？传统的业务安全和主机安全并没有针对主机cgroups和namespace的防御设计；结合云原生的架构模式，可以在两个环节建设容器逃逸检测。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/ee593319-846d-422c-af15-480d7cfa5f53.png?raw=true)

**DevSecOps安全左移，对代码仓库构建产物的IaC(基础设施即代码)进行审查，在开发阶段对可能存在逃逸风险的配置项问题进行阻断，防止出现各类逃逸问题。** 

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/22863d26-1fd3-4c36-970f-76a43979ee73.png?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/f3a685b1-c743-4e08-b6b4-e1a510cbb218.png?raw=true)

**运行时监测, 对于已经上线并实际运行的业务容器，实时监控逃逸特征。通过订阅内核事件，抓取可能为逃逸的行为特征，结合当前运行中的容器配置信息进行分析，综合给出是否存在逃逸风险以及是否发生逃逸事件的告警。** 

  

结合这两点，可以做到对逃逸风险的有效控制，防止攻击者进一步的攻击行为。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/9b2515be-f426-4eeb-bdfa-d7f9bc31b375.png?raw=true)

横向扩散

在实现逃逸后，获取到的数据就越来越宽泛。

*   通过k8s节点存储凭证可获取各种敏感资源如secrets、configmaps。
    
*   通过imagePullSecrets获取到了私有镜像仓库权限。
    
*   环境变量中存储了access_keys等敏感信息。
    
*   其他用户的pod信息等。
    

可以看到，在实现了容器逃逸后，攻击者轻松从各种凭证信息实现横向扩散，包括各类敏感资源数据、镜像仓库权限等等；**这好比在传统的渗透中，从DMZ区进入到了内网环境后，发现大多主机MS-17010通杀的场景**；说明对于云环境下的节点管理，仍需要加强警惕，尤其是在容器逃逸防护较弱的场景下。

针对这部分横向移动与扩散的攻击行为，可以和容器逃逸一样分为两个方面进行防护：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/56b7bc01-31a7-41b9-9349-fe24e9179d2e.png?raw=true)

**事前防护：对镜像资产/容器资产进行扫描，保证镜像/容器内没有敏感信息以及过高的凭证信息。** 

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/927c148d-3aeb-44da-9f85-28500a4e6b73.png?raw=true)

****事中防护：通过对集群日志审计来发现横向移动行为，快速响应并隔离问题容器。****

  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/7dd11aed-9c00-4c27-a0ed-747f3fc0edd9.png?raw=true)

综合反思

回顾整条攻击链，我们可以总结以下几点：

**传统的业务安全在云原生环境下依旧处于重要地位**

回看整个攻击过程，所有的切入点仍然是一个容器服务。虽然容器提供了相对隔离的运行环境，但传统的业务安全如：web安全所导致的问题依旧作为了云安全事件中的切入点，其地位不亚于弱口令。

**云环境的场景下，传统安全的危害效应将指数级放大**

云环境中的配置复杂，一旦存在了配置不当的情况，攻击者便可以轻易的将攻击从一个容器扩散到一台主机、一个集群、甚至于多个集群、整个k8s环境。此时，作为入侵的入口导致的影响指数级上升。

**云环境的场景下，容器逃逸是整个攻击链路的核心**

在整个攻击的步骤中，我们可以总结出：攻击者获取到一个容器的最高权限后，必须千方百计的实现逃逸来绕过各种资源限制，才能够产生更为严重的影响；因此，云环境场景下，逃逸问题是连接传统安全与云安全的关键核心。

**云环境对基础设施的监控、审计需求更加复杂和迫切**

云计算的特性使得基础设施的边界变得模糊，资源的动态变化增加了管理和保护的难度，同时多租户环境下的安全性和合规性风险也需要得到充分的关注和解决，带有缺陷的隔离限制将轻易的导致用户数据泄露。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/50b0d96a-d008-42a8-ac5a-8d3a87fdfedb.png?raw=true)

同时，我们也能看到，云环境的场景下，对于蓝军防守方，提供了一个新的思路：

**在面对传统安全覆盖率永远无法达到100%以及层出不穷的0day漏洞，加强对云安全防护以及逃逸检测能够有效的中断攻击链路，降低攻击危害，实现低成本/高回报率的防护。** 

针对对以上的问题，牧云·云原生安全平台提供了完整的监控与检测方案，深入监控了每一个容器的生命周期，在传统的 webshell、反弹 shell 等安全入侵能力检测上，结合云环境特点，实现了容器逃逸风险监测、集群日志审计等功能，帮助清晰明了的掌控集群的实时安全；当发生安全事件时，黑客的恶意行为和特征将会被检测与捕捉，实时反馈到平台中。

除此之外，为保证供应链安全，平台提供了定时任务机制和多种多样的资源集成，定时检测镜像仓库等远端资源，第一时间发现风险镜像，包括敏感信息泄漏、恶意文件、软件漏洞等问题。并通过配置可以阻断来自于该镜像的容器创建请求以及镜像构建请求，防止攻击扩散。

与此同时，平台还考虑了云场景下资源迭代的快速以及漏洞响应的及时性，设计了插件系统，结合开源社区安全检测能力的沉淀，能够快速赋能最新漏洞检测需求，并热更新于平台中，提供与时俱进的插拔式安全检测能力；如前一阵子出现的Minio漏洞（CVE-2023-28432），可以通过快速插入专项检测插件来实现 0 day 速查。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/cbc3a9cc-767c-415c-9381-9c1f08b0a161.png?raw=true)
 

扫码加入

云原生  

交流群

往期推荐

[2023年了，云原生安全怎么做？](http://mp.weixin.qq.com/s?__biz=MzIzOTE1ODczMg==&mid=2247496527&idx=1&sn=c62fd08fdbdfde4afe24c4c609eef16e&chksm=e92ce7ecde5b6efa9be6c13c430704a86d117b7406ec8998ed066620a6050577b60c2ab43f8d&scene=21#wechat_redirect)  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/5/ace18a25-ec3d-4e6e-a74c-6abee52bcfd7.png?raw=true)