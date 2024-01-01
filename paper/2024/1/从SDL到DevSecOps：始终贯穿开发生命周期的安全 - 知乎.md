# 从SDL到DevSecOps：始终贯穿开发生命周期的安全 - 知乎
> 最近参与了《研发运营一体化(DevOps)能力成熟度模型》等标准安全部分制定，以及在内部做了一次分享，趁着分享之后聊聊自己对对研发安全以及DevSecOps的理解和实践尝试。

随着云计算被普遍运用，微服务等基础架构的成熟，同时企业业务高速发展带来的对开发运维更高效的要求，企业开发运维模型也从传统的瀑布模型演变到敏捷模型再到DevOps，而安全模型也随之改变，但**其核心一直都是贯穿始终以及更前置的安全**。其中“DevSecOps”是Gartner在2012年也提出的DevOps模式下的安全概念，**人人为安全负责，让业务、技术和安全协同工作以生产更安全的产品**。

### 从漏洞与威胁防御说起

假如问大家“如何收敛产品中的安全漏洞”，可能得到的答案是安全测试；而如果问题改为“如何减少产品中漏洞产生”，那么答案可能是“减少漏洞代码”。

其实两个问题得到的答案对应的是**两种防御理论，一个是漏洞防御，一个是威胁防御**。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/b810b517-39fb-46c2-8029-557e34d1d981.jpeg?raw=true)

漏洞防御体系从防御角度是类似导弹防御系统，针对性的防御，面对的对象是可以识别的也就是已知、明确的安全漏洞进行修复和防御，这种方式好处就是模式简单，能够快速、高效的起到效果，但问题也很明显，就是这事一种单点的方式，其实是适合攻击，所以会不够全面，存在未知的风险。这事一种增加攻击成本的方式，我把漏洞都补上了或者加上防御措施，那么一般的黑客想攻破我的系统就得突破导弹防御系统，也就是找到防御方没有修复或者增加防护策略的安全问题，攻击成本就变得更高；

而威胁防御体系，其实讲究的是基于软件开发全生命周期甚至包含运维阶段发掘潜在的安全威胁，通过威胁建模、安全设计、安全测试等多个角度消减威胁和建立防御手段。相对比漏洞防御体系，这里的威胁不需要是明确的已经形成的安全问题，而是潜在的威胁都应该建立对应的手段进行识别和消减。优势自然显而易见，更系统化，更全面，但更系统更全面自然需要花费更多的时间，整体执行周期长，同时要求在各个阶段都要有安全动作，操作复杂性高，要求安全覆盖度更高。

过去我们在梳理腾讯云相关安全落地的具体体系的时候，其实就是结合了两种理论，参考SDL、IPDRR等模型制定：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/03ef7a71-7cbc-4c3a-b70f-5802f0f9cace.jpeg?raw=true)

### 更前置的安全

回到软件安全开发，**不管是SDL还是DevSecOps，其中主要强调的一个就是安全前置或者安全左移，就是更早的在软件开发生命周期嵌入安全动作，就能更容易的收敛安全漏洞问题**。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/bb1112f0-bcba-4477-b3b4-2df495757a92.jpeg?raw=true)

统计来自Forrester Analytics Global Business Technographics Security Survey, 2019，图来自《SDL已死，应用安全路在何方？》

为什么要关注应用安全？关注软件开发安全？这是来自Forrester的一个调研统计，从图中可以看出，**企业的攻击风险点依旧是以应用漏洞为首**，攻击者依旧是紧盯目标在软件安全领域的安全漏洞持续渗透。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/a8ca2fab-ffb1-49d8-84e5-26b54b0df63e.jpeg?raw=true)

而这张图描述了在开发运维不同阶段的漏洞修复成本，可以发现越早成本越低，漏洞也更容器修复，所以这就是为什么软件安全需要更左移，更前置。

**越早的收敛漏洞成本越低，而软件安全开发模型就是用于解决的方案。** 

### 安全开发生命周期(SDL)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/81d3dc24-69c2-4a92-8af9-afed6ef680b3.jpeg?raw=true)

SDL由微软提出并应用一个**帮助开发人员构建更安全的软件和解决安全合规要求的同时降低开发成本的软件开发过程**，侧重于软件开发的安全保证过程，旨在开发出安全的软件应用。

SDL的核心理念就是将**安全考虑集成在软件开发的每一个阶段**：需求分析、设计、编码、测试和维护。从需求、设计到发布产品的每一个阶段每都增加了相应的安全活动，以减少软件中漏洞的数量并将安全缺陷降低到最小程度。

其实从表中我们就可以清晰的看到每个阶段需要做的不同的安全活动，借用网上的一张图，大概的执行流程是这样的：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/54a9db79-4cb9-40d7-b4ee-94cf98c0cc83.jpeg?raw=true)

在SDL模型里，有个比较核心的点是安全设计核心原则：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/f4577bd5-4694-4539-8157-2cfdf1cf4831.jpeg?raw=true)

其他都好理解，重点说下攻击面最小化和默认安全。攻击面最小化其实主要是两块，一个维度是暴露的CGI等都可能是攻击点，非必要的接口等要减少，另一个是即时暴露也应该限制访问访问范围从而缩小攻击面；而默认安全则指产品在设计的时候一些配置的默认项应该考虑是安全状态，比如一些安全开关应该是默认开启。

在SDL模型里，还有个很重要的执行要点，那就是威胁建模，威胁建模是在需求设计阶段的一项识别和消减威胁的重要手段。关于威胁建模，微软提出的一个方法叫做“STRIDE”。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/3ecce5f4-01c5-48fa-937e-85a4d045323b.jpeg?raw=true)

STRIDE威胁建模其实就是基于数据流图去识别不同环节是否存在仿冒、篡改、抵赖、信息泄露、拒绝服务、权限提升几个维度的安全威胁，并制定对应的消减措施，落实并验证的一个过程。其中六个维度的威胁大家通过表格里的安全属性就可以看到它其实是基于信息安全基本要素制定的。额外提下，由于隐私安全的关注的爆发，也成为了第七个安全威胁。具体这里就不对威胁建模展开说明了，大家有兴趣可以通过扩展阅读内容进行学习。

经常有同学会对几个关键词混淆不清，SDL、S-SDLC、SDLC，这里也额外做下解释，好帮助大家理解。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/129b358f-ab63-4657-9989-a1cda0fe749c.jpeg?raw=true)

通过英文全称大家应该就可以看得出区别，前两者都是安全开发生命周期，第三个是软件开发生命周期。至于S-SDLC和SDL有什么区别呢？S-SDLC是由开源Web安全组织OWASP推出的一个项目，它跟SDL的区别是它更关注的是SDL的落地化。

### DevOps与DevSecOps

要讲DevSecOps就必须先介绍下DevOps，就涉及到软件开发模型的变更。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/cf732a18-c56f-4537-9c8d-dd987d7181df.jpeg?raw=true)

这是一个软件开发运维的流程，一开始可能是一个角色负责所有阶段，当系统变得复杂化后，于是几个角色就被区分出来，分别是开发、测试、运维。

针对这样一个流程，最经典的一个模型是瀑布模型：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/c03993c9-28b4-4772-a5e6-771c33dd0122.jpeg?raw=true)

就是一个阶段做完再到下个阶段，我们会发现这是一个很低效的过程，于是后来演变出了敏捷模型（其实还有其他的模型，不过相关性不大，就不做介绍了），我们用来自网上的两张图来体现敏捷模型与瀑布模型的区别：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/8bec1462-0e39-4456-9e32-387a9a77a9ee.jpeg?raw=true)

通过对比可以看到，在敏捷模型里开发和测试不是原来的完成全部开发再测试这样的情况，而是类似下图这种：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/d7f678e1-edad-46a8-87ca-57f77de64883.gif?raw=true)

在敏捷模型里，大家可以发现，运维阶段的工作还是在最后，所以再之后演变出DevOps：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/73e479af-8f4f-42e3-82b7-62cf158032c3.jpeg?raw=true)

从图中可以看到迭代过程中，开发测试部署是快速迭代同时进行，部署操作不再是等到最后。这就是一个简单的开发运维模型的一个变更过程。

然后我们回过头来看看DevOps:

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/1edc498d-872a-4a08-8964-5e0a2997e225.jpeg?raw=true)

第一段是来自AWS的解释，DevOps集文化理念、实践和工具于一身。可以提高组织交付应用程序和服务的能力。与使用传统软件和基础设施管理流程相比，能够帮助组织更快的发展和改进产品。这种速度使组织更好的服务其客户，并在市场上高效的参与竞争。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/aec10533-aab8-418d-a187-2337e6df274a.jpeg?raw=true)

对于DevOps来说，有个核心元素就是CI/CD，所以到DevSecOps，大家会发现在CI/CD嵌入安全也是一个重要环节。

DevOps的流水线大概是这样的：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/d9cbfe30-1522-46bd-ba04-ed5f95918840.jpeg?raw=true)

其实，DevOps早在九年前就有人提出来，为什么这两年才开始受到越来越多的企业重视和时间呢？因为DevOps的发展是独木不成林的，现在有越来越多的技术支撑。微服务架构理念、容器技术使得DevOps的实施变得更加容易，计算能力提升和云环境的发展使得快速开发的产品可以立刻获得更广泛的使用。

随着开发运维模型的变更，SDL就不再是完全适用于新的模型，DevOps带来的各种优势和技术趋势甚至成为了安全实施的难点：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/eb814d1a-ec5b-4bd9-a048-91ee6ed1ec0a.jpeg?raw=true)

所以早在2012年Gartner就提出了DevSecOps，并通过这么多年的发展，逐渐成熟。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/14bb5b96-5770-46a4-9136-b051730fd6e4.jpeg?raw=true)

相对于SDL，DevSecOps不再是一个单纯的安全开发模型，也不仅仅是关注开发阶段，它所强调的是**人人为安全负责，人人参与安全，安全嵌入到开发到运维的每个阶段**。

所以首先从角色角度，安全团队不再置身于业务之外，也不再是安全团队兜底安全，安全是一个开发、安全、运维、QA一起协作的过程。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/dcbb3e56-1750-4d63-90c4-3fa6fd7ccf19.jpeg?raw=true)

然后我们可以简单做下SDL和DevSecOps的对比,其中最明显就是安全责任、安全关注的流程以及效率的区别。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/1f652ecb-9c38-41e5-b5fa-18b003ea858f.jpeg?raw=true)

但是其实是不是原来SDL的东西就没法用到DevSecOps？需要推翻呢？不是的，需要做的是进一步的流程融入，更加自动化，更多前置，以及安全文化的塑造。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/d16c0d38-1315-4871-924b-d0d16312f3a1.jpeg?raw=true)

在DevSecOps里有三个关键点，分别就是人和文化、流程以及技术。传统安全里业务发展优先，安全是“以后”才会发生的事情，甚至认定安全会阻碍业务的发展，而DevSecOps强调的是人人参与安全，人人为安全负责,安全是大家的事,其实也确实应该是这样的；而流程方面更多要考虑整合流程，建立相关安全流程，加强不同团队间的协作,以及安全需要低入侵柔和的嵌入开发和运维流程；技术方面更多是构建安全工具链，实现更多自动化安全检测。

在具体落地和实践方面，RSAC2019大会安全专家Larry Maccherone提出的实践DevSecOps的九大关键因素文化融合的七个阶段,也可以作为参考:

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/44750ab2-ec5d-4413-8efa-b52cd0476899.jpeg?raw=true)

文化融合的七个阶段

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/dd3584a8-9431-49b3-9061-135fb7ca92b0.jpeg?raw=true)

包含有文化意识维度的安全意识、安全编码,在架构和设计维度的威胁建模，工具方面需要构建的第三方导入代码分析(第三方组件安全)、代码编写分析；然后是全漏洞管理要建立团队工作协议，也就是强调协作，建立漏洞管理共识和处理流程，对安全问题优先进行高危漏洞清理。最后其他监督方式如安全同行的审阅，一些安全评估手段。

七个阶段运用不同颜色直观展示DecSecOps在组织中的实践和接受程度。

这九大实验要素基本覆盖DevSecOps落地的一些关键点，包括说需要关注供应链安全，考虑第三方组件的安全，这也是我们现在在做的一些方向。

在落地DevSecOps过程中，其中很重要的一块我觉得是构建工具链，在不同的DevOps阶段需要进行不同的安全动作,都需要不同的工具支撑。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/340307bb-5a9e-41fc-9762-018bfc9b4152.jpeg?raw=true)

其中比较关键的工具是AST及SCA：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/2a2ae0cb-df1a-47be-ab0a-031061d9967d.jpeg?raw=true)

DAST就是动态应用测试，比如说我们常见的AWVS、APPSCAN等漏扫就是这个类型，SAST是静态应用测试，通常就是代码审计工具，而IAST则是介于两者之间的交互式应用测试工具，通常通过插桩的方式，既能像SAST定位具体问题代码位置也能像DAST能定位到具体CGI。这里需要注意的是，通常流量代理测试的方式也有人归类到IAST，但实际它不是真正意义上的IAST。

过去我们在做腾讯云研发安全的过程中，也在构建相关工具链：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/785794c2-50fe-4505-a42d-f69160266c27.jpeg?raw=true)

或者通过另外一个视图可以看到我们过去在云相关安全工作在DevSecOps中的情况：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/421ee229-0e9a-401d-b757-667cfa5b7a1e.jpeg?raw=true)

除了工具链，上文也提到，DevSecOps的落地中很重要的一个部分也是我们一直做的一个点就是如何在CI/CD嵌入相关安全动作。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/0ac60a73-e76b-4b79-bb63-629f14d60da6.jpeg?raw=true)

RSAC2018出现了一个新概念“Golden Pipeline”，叫做“黄金管道”，特指一套通过稳定的、可落地的、安全的方式自动化地进行应用CI/CD的软件流水线体系，所以基于这样图将我们的工具链对应上，大概就包含这几个部分的内容。

所以总结来看，DevSecOps的落地是有几个关键点的：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/70c289d8-09e2-426b-a055-d9a24ae5c149.jpeg?raw=true)

我用一棵树来表示，最基础的部分是工具链的建设，然后核心的枝干，也就是落地点就是在CI/CD中进行安全动作的嵌入，而枝叶部分，一些需要重点的安全动作就包含有自动化测试、软件成分识别与检查，以及对应一些在新的技术趋势下需要特别关注的关注点就包含有容器安全、API安全、第三方组件安全。

过去在几个关键点其实我们也在做一些落地，比如联合公司内代码扫描平台和其他团队共同构建的静态代码扫描：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/99a8f3e5-02fc-4c03-9186-7392385565a7.jpeg?raw=true)

基于公司统一源，以及与其他团队合作及自研的第三方组件安全管理方案：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/25b3a740-62ef-4012-8b75-c420618606a5.jpeg?raw=true)

与IEG安全团队共建包含现在开源协同作为公司统一容器基线安全解决方案的容器镜像扫描能力：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/2cf4c429-fedb-414b-b6b3-93f7381aa9cd.jpeg?raw=true)

还有与云API、官网等团队合作的一些API安全方面的尝试：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/69900e50-ff5a-4ab3-8d5d-f51c5b7be378.jpeg?raw=true)

以及未来更多展望，包含更完善的安全开发库及自动化安全规范检查，或者安全设计与需求阶段的自动化检查等。

注：文中部分图来自 [https://baijiahao.baidu.com/s?id=1649919009474612596](https://link.zhihu.com/?target=https%3A//baijiahao.baidu.com/s%3Fid%3D1649919009474612596)

博客原文：[https://www.fooying.com/from\_sdl\_to\_devsecops\_security\_in\_dev/](https://link.zhihu.com/?target=https%3A//www.fooying.com/from_sdl_to_devsecops_security_in_dev/)

### 扩展阅读

*   微软SDL [https://www.microsoft.com/en-us/securityengineering/sdl/practices](https://link.zhihu.com/?target=https%3A//www.microsoft.com/en-us/securityengineering/sdl/practices)
*   STRIDE威胁建模方法讨论 [https://www.freebuf.com/articles/es/205984.html](https://link.zhihu.com/?target=https%3A//www.freebuf.com/articles/es/205984.html)
*   威胁建模工具入门 [https://docs.microsoft.com/zh-cn/azure/security/develop/threat-modeling-tool-getting-started](https://link.zhihu.com/?target=https%3A//docs.microsoft.com/zh-cn/azure/security/develop/threat-modeling-tool-getting-started)
*   S-SDLC [http://www.owasp.org.cn/owasp-project/S-SDLC/](https://link.zhihu.com/?target=http%3A//www.owasp.org.cn/owasp-project/S-SDLC/)
*   应用安全测试技术DAST、SAST、IAST对比分析 [https://www.aqniu.com/learn/46910.html](https://link.zhihu.com/?target=https%3A//www.aqniu.com/learn/46910.html)
*   从RSAC看DevSecOps的进化与落地思考 [https://www.freebuf.com/articles/neopoints/228886.html](https://link.zhihu.com/?target=https%3A//www.freebuf.com/articles/neopoints/228886.html)