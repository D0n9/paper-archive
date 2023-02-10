# Infrastructure from Code，一种新理念
我们经常说到Infrastructure as Code（基础设施即代码，IaC），相信几乎所有人都明白它是什么。

但是你可能没听说过一个新兴的概念：Infrastructure from Code（IfC），它是一种思考云基础设施的新方式，一种创建应用程序的新方法。

在部署时，你的云提供商会检查你的应用程序代码，然后自动负责配置你的应用程序代码所需的任何底层基础设施。这与基础设施即代码形成对比，你（开发人员）需要在单独的文件中明确定义基础架构资源，比如在CloudFormation/Serverless Framework YAML或CDK堆栈中。

IfC工具将通过在你的应用程序代码文件中查找约定和提示来完成此操作。让我们看一下无服务器云文档中的示例：

```


import { api, data } from '@serverless/cloud'

api.post('/users', async (req,res) => {  
 await data.set(`users:${req.body.email}`, req.body)  
 res.send({ message: 'User saved' })  
})




```

每当你使用Serverless Cloud CLI部署上述代码时，其托管服务会检测到API和数据结构，并理解它需要在后台提供API和数据资源。我相信他们在AWS的后台分别使用了API Gateway和DynamoDB，但开发人员不需要知道这一点，也不需要为使用这些资源时可能出现的复杂配置（IAM等）而烦恼。

这样做的主要好处是，编写的代码要少得多，而且减少或完全不需要学习CloudFormation等IaC语言，许多刚接触无服务器的开发者认为这是一个重要的学习曲线。Shawn Swyx Wang在他的The Self-Provisioning Runtime一文中指出：

如果开发者体验的柏拉图式的理想是一个 “只写商业逻辑” 的世界，那么逻辑上的结局就是语言+基础设施的组合，它可以解决其他一切问题。

我同意这种方法将是应用开发的未来。但这个结局可能还需要几年时间，我在短期内的一个担心是这种隐含的传统方法缺乏灵活性。

比如说我需要使用一个还不支持的云服务，或者对一个还没有被公开的底层原语的设置进行修改。或者，如果我需要完全弹出，我的应用程序完全是用专有的结构编写的。我确实认为，与AWS/Azure/GCP这些巨头相比，供应商锁定的论点对小型云计算企业来说更有分量。

但随着无服务器云等Infrastructure-from-Code工具的特性和能力的增长，其中一些担忧会减少，谁知道呢，甚至可能会出现一些标准。

_—__**1**_ _—_  

**基础设施即代码的演变**

近年来，基础设施格局已经发生了变化，基础设施即代码（IaC）成为定义基础设施的首选解决方案。IaC是使基础设施易于配置和拆分的最新的主要主流模式，这一趋势可以说是从2000年代初期到中期的虚拟化商品化开始的。

第一波IaC工具引入了一个新的DSL，旨在以可重复的方式创建、配置和管理云资源。Chef、Ansible、Puppet和Terraform是这一浪潮中最受欢迎的工具中的一部分。

之后，第二波IaC工具用TypeScript、Python和Go等现有的编程语言取代DSL，以表达同样的想法。Pulumi和CDK是这一浪潮中的一些流行例子。

虽然这些工具在不断改进和增加更高层次的功能，但从根本上说，它们都需要人来声明他们所需要的特定基础设施组件，通常需要相当详细的信息。这意味着开发人员不仅需要了解他们的应用程序需要哪些资源，而且需要了解权限模型、依赖资源和这些资源之间的通信联系。

在IaC的世界里，想要将API暴露在互联网上的开发者或运营商需要建立一个API网关，将其连接到Web服务器或框架，转换流量，配置私有网络和安全规则，并确保角色权限（如IAM）都被正确配置——这还没有考虑其他资源，如数据库、机密管理器或消息传递系统。

在代码中这样做更容易重复和审计，但这也意味着你的开发人员或DevOps人员基本上需要编写两个独立的应用程序（服务级应用程序和基础设施）来协同工作。

现在是下一个模式的时候了，我们认为这个模式是将你的基础设施从应用程序代码中衍生出来，而不是将其定义为代码。

_—__**2**_ _—_  

**什么是Infrastructure from Code？**

Infrastructure from Code（IfC）是一个分析你的应用代码以推断你所需要的云资源的过程，然后创建和维护它们，而无需你手动定义它们。

IfC不依赖手动配置，而是仅凭借其在代码中的存在，来推断Web服务器并将其暴露给互联网。组件的设置成为该工具处理的实现细节。与任何新的抽象层次一样，放开控制权一开始可能会让人感到害怕。但我们认为它将开启生产力的代际飞跃，因为公司能够专注于他们的核心服务，而不是无差别的工作。

IfC有几种不同的方法，不同的提供商对它有不同的呈现。所有这些都有一个共同的高层次的愿景，即让服务代码完成大部分对话，并让IfC工具将其需求转化为基础设施。这些方法在与服务级代码结合的紧密程度上有很大的差异。

**编程语言**

在这种方法中，像Wing和DarkLang这样的初创公司正在引入以云为中心的新编程语言。这些基于语言的方法可能会引入新的结构，这些结构无法在像Python、Go或Java等现有语言中简单地建模。

相比之下，DarkLang提供了不同的构建模块，如云数据存储和将API公开到Internet的方法：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/702d272c-752c-454f-8f75-bbd78fb6be4f.jpeg?raw=true)

另一方面，Wing更像是基础设施和代码，而不是FROM代码，它将云结构和经典的编程结构结合在同一个结构中。例如，在Wing中，开发人员可以用cloud.Function结构定义一个计算元素，并使用几行代码定义一个 Bucket 来存储 blob 数据：  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/e24fe2d0-f8eb-410b-afca-7eaecc2420e8.jpeg?raw=true)

基于语言的方法有可能提供卓越的用户体验，因为它允许引入现有编程语言难以或不可能实现的新概念。诸如交互性和分布式计算等功能可以更容易实现，使软件开发者的过程更简单、更直观。

这种方法有四个主要的考量：

*   软件开发人员必须致力于学习一门全新的语言，这意味着告别多年的实践和他们已经知道的编程语言的专业知识。
    
*   开始使用一种新语言意味着从头开始一个新的生态系统或拥有出色的互操作性故事。
    
*   将一种新语言集成到现有工具和服务中需要大量的工作。与Typescript一样，Wing已经提前完成了这项工作，并且已经支持Typescript和JavaScript模块。
    
*   从技术上和组织上来说，寻找和雇用具有较新语言专业知识的开发人员将是困难的。（比如说Haskell）
    

**基于SDK**

在这种方法中，像Ampt和Nitric这样的工具引入了他们自己的SDK，开发人员在他们的代码中使用。在部署时，这些工具会分析服务代码如何使用SDK，并从中生成基础设施。

引入他们自己的SDK使得从代码中推断出的使用情况更可预测，并且可以针对其设计的场景进行定制，但这也意味着SDK在利用新的底层云功能方面总是落后一步。 

例如，为了用Nitric将一个端点暴露到Internet，我们从@nitric/sdk包中导入api包，并根据其特定语法定义路由：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/367c56c4-07d8-4660-a6e8-b9c37e66f2ce.jpeg?raw=true)

另一个例子是一个集合包，它作为数据持久化的文档存储：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/fa76ddd6-a263-4207-9546-ecf1aa88127c.jpeg?raw=true)

使用新的特定于平台的SDK（如Ampt和Nitric提供的SDK）需要开发人员学习和使用这些库，但是它们有可能释放出只有它们才能提供的独特功能。然而，这是以牺牲流行的社区驱动的库可能提供的功能的广度和深度为代价的。

虽然这些库仍然可以使用，但它们不会受益于仅在提供者SDK中可用的独特功能。最终，开发人员将不得不在流行的社区驱动库的功能深度和广度与特定于平台的SDK提供的功能之间做出权衡。

**注释+框架**

通过这种方法，像Encore和Shuttle这样的工具可以让开发者注释其代码的一部分，然后这些工具将它们合并到工具的框架中。根据工具和你的部署目标，这个框架可能被托管在IfC供应商的云基础设施上，或者它可能更直接地与AWS、GCP或Azure等第三方云提供商集成。这些通常都有自己的部署工具。

例如，在Encore中，开发人员无需导入服务发现SDK或服务包装器，而是用预先定义的函数签名编写普通函数，然后注释这些函数，以告诉Encore如何将这些函数翻译成其托管的对应函数。

使用的输入和输出类型成为API请求/响应模式，而注释则指定了URL路径。然后，Encore自动在AWS/GCP/Azure的本地、预览和云环境中提供相关的基础设施。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/3894f8bf-c58f-4766-98d1-48ae91be748e.jpeg?raw=true)

对于其他基础设施资源，Encore采用了更类似于SDK的方法。声明关系数据库、发布/订阅、缓存、cron job、secrets和配置等基础设施资源是通过Encore提供的SDK完成的：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/c5309183-2aad-4ba6-83dc-e35a732ac0e1.jpeg?raw=true)

Shuttle实现了一种带有注解的依赖注入模式，以向一个函数添加路由上下文，并将请求范围内的对象（如头文件、参数和正文）注入该函数。Shuttle的方法对流行的开源库的使用进行了包装。例如，为了使用Rust的Rocket框架，你定义并注释了一个创建ShuttleRocket实例的方法。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/d0f76b46-6fa7-40b2-aead-08b1d304d141.jpeg?raw=true)

Shuttle再次利用依赖注入和带注释的#\[shared::Postgres\]来启用数据持久性，这允许它与另一个名为sqlx的流行开源库结合使用：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/b8b0c102-b0ef-4f76-bda3-548f28936230.jpeg?raw=true)

然而，Shuttle和Encore同时依赖于SDK方法和注释方法，使概念数量翻倍，也使开发者需要考虑的弊端增加。

SDK方法要求开发者学习和使用自定义库，牺牲了流行的社区驱动的库所提供的功能的深度和广度，而注释方法可以成为自己的语言/DSL，要求开发者理解注释处理器，并经常手动编写代码来解决任何问题。多种方法使得创建连贯和可预测的语言设计更加困难，特别是当两者相互作用时。

**纯注解**

这种方法只基于代码内的注释，并依靠现有的、开源的库，如网络框架和持久性。这建立了一个更严格的关注点分离：在这种方法中，IfC工具不负责托管，甚至不负责选择框架，而是专注于了解开发者对框架和工具的使用。

这个领域的领先工具是Klotho，它超越了Infrastructure-from-Code，进入了Architecture-as/from-Code，强调其工作是理解应用程序的架构，而不是定义它。

Klotho有目的地引入了几个关键注释，称为功能，使现有的编程语言成为云原生语言：

*   将网络API暴露在互联网上
    
*   将多模态数据持久化到不同类型的数据库中
    
*   static_unit将静态资产打包并上传到CDN进行发布
    

例如，在一个Python应用程序中，注释流行的FastAPI模块会将其路由器上定义的所有路由暴露给互联网。Klotho能够追溯并了解开发者如何使用FastAPI的原生路由器定义路由。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/7a715465-26a1-4a35-9a76-f2c7d7d6cce8.jpeg?raw=true)

数据持久化是使用持久化注解启用的。例如，在JavaScript中，开发人员可以用持久化功能来注解标准的Redis客户端或TypeORM实例：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/1990e347-014e-49d4-87c6-41e637fd47e4.png?raw=true)

当你部署时，Klotho不仅会推断出它需要部署的基础设施，而且还会重写服务代码，将该基础设施的连接字符串与你所注释的变量、常量或函数连接起来。

Klotho还有两个专业的高级功能：

*   exec_unit划定了服务边界，便于跨执行单元的调用
    
*   pubsub实现跨执行单元的事件驱动消息传递
    

例如，用pubsub功能注释一个普通的NodeJS EventEmitter，允许两个独立的执行单元，在这种情况下，两个独立的模块或文件，通过普通事件进行通信，但在云环境下，由一个适当的管道支持，如SQS/SNS/Redis流等。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/fe005dc6-869e-41e1-a87d-92f31403e5ec.jpeg?raw=true)

同样，在单独的执行单元中调用函数，会自动将它们转化为由API调用、gRPC、Linkerd或类似的适当管道支持的线上调用。

然而，权衡利弊，工具需要不断扩大，以了解现有的和不断增长的库、语言、设计模式、云和它们提供的底层服务的集合——问题的维度变得很大，可能太大了。

与注解+SDK的方法类似，但没有SDK作为补充，注解系统的诱惑力越来越大，直到它成为一种完全成熟的语言/DSL，要求开发人员了解注解处理器，并经常手动编写代码来解决任何问题。

_—__**3**_ _—_  

**总结**

*   Infrastructure from Code有4种主要方法：基于SDK（Ampt，Nitric）、基于代码内注释（Klotho）、两者的结合（Encore，Shuttle）和通过新的编程语言明确定义（Wing，DarkLang）。  
    
*   采用Infrastructure from Code的公司有望在交付云驱动软件的生产力和效率方面拥有显着优势。
    
*   Infrastructure from Code预计将在未来几年变得更加流行，并将自己确立为现有云开发方法的替代方案。
    
*   Infrastructure from Code（IfC）是一个过程，它可以通过理解软件应用程序的源代码来自动创建、配置和管理云资源，而无需明确描述。
    
*   第二波IaC工具，如Pulumi和CDK，使用现有的编程语言，如TypeScript、Python和Go，来表达与第一波工具相同的想法。
    
*   基础设施即代码（IaC）工具，例如 Chef、Ansible、Puppet和Terraform，是最早使用领域特定语言（DSL）创建和管理云基础设施的工具之一。
    

推荐阅读：

*   《[为Kubernetes集群部署一个ChatGPT机器人](http://mp.weixin.qq.com/s?__biz=Mzg5Mjc3MjIyMA==&mid=2247558898&idx=1&sn=4c512e659bca68fd976627640b7647db&chksm=c03aace1f74d25f7f35db40e411426c455bde7092095853fafb395647adee3d90216e69ab104&scene=21#wechat_redirect)》
    
*   《[FinOps，值得关注](http://mp.weixin.qq.com/s?__biz=Mzg5Mjc3MjIyMA==&mid=2247558700&idx=1&sn=ba7e8719c4fa7e85baf31c5c190508b6&chksm=c03aac3ff74d252959fa0840e8b1087536272baee6d7ae163fab816fbecaf3cd77d1bd4ad87b&scene=21#wechat_redirect)》
    

* * *

分布式实验室策划的《[**Kubernetes实战集训营**](http://mp.weixin.qq.com/s?__biz=Mzg5Mjc3MjIyMA==&mid=2247558165&idx=1&sn=e14ac020f2e40d7d96a09326f13a18b8&chksm=c03aae06f74d2710d6b520e9c9b761c6fec18d0cf409fa790097d0ac749a96846ebabef1872f&scene=21#wechat_redirect)》正式上线了。这门课程通过通过5天线上培训，40个小时直播，15个随堂练习，50天课后辅导，把Kubernetes的60多个重要知识点讲给你，并通过实战帮你掌握Kubernetes。培训重实战、重项目、更贴近工作，边学边练，2月25日正式开课。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/794578ea-da3b-4da7-9afc-487283612bef.jpeg?raw=true)
](http://mp.weixin.qq.com/s?__biz=Mzg5Mjc3MjIyMA==&mid=2247558165&idx=1&sn=e14ac020f2e40d7d96a09326f13a18b8&chksm=c03aae06f74d2710d6b520e9c9b761c6fec18d0cf409fa790097d0ac749a96846ebabef1872f&scene=21#wechat_redirect)