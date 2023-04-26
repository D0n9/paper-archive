# 更高水准的供应链安全实践：Alibaba Dragonwell 及其 SLSA 2 级认证-阿里云开发者社区
_01_ 前言
-------

计算机科学史上涌了 C/C++、Java、JavaScript、Ruby、Python、Perl 等多种编程语言。每一种语言都有其擅长的领域，其中Java语言凭借其面向对象、自动内存管理、多线程性能优越等优势持续处于浪潮之巅。目前市场上存在着大量质量不错 OpenJDK 的衍生版本可供用户选择。它们或者性能卓越，或者针对某些场景做出了优化。但 JDK 作为基础软件，归根结底软件的可信任和可用性是最我们最基础的追求。

_02_ Alibaba Dragonwell
-----------------------

Alibaba Dragonwell，一款免费的、生产就绪型的 OpenJDK 发行版。阿里巴巴提供长期支持，包括性能增强和安全修复。不同于以往的 OpenJDK。Alibaba Dragonwell 有其特有的五大优势：

**2.1 安全与稳定**

Alibaba Dragonwell 与 OpenJDK 社区保持紧密合作，始终保持对社区工作的跟踪，及时同步上游更新，以保证 Java 应用的安全和稳定。

**2.2 性能与效率**

Alibaba Dragonwell 的前身是阿里巴巴内部使用的 AJDK。AJDK 作为阿里巴巴 Java 应用的基石，支撑了阿里几乎所有的 Java 业务，积累了大量业务场景下验证过的新技术，这些新技术极大得提高了阿里巴巴Java业务的性能和故障排查效率。AJDK 创新技术，会逐步贡献到 Dragonwell 沉淀。

**2.3 Java SE 标准兼容**

Alibaba Dragonwell Standard Edition 完全遵循 Java SE 标准。

**2.4 特色功能**

Alibaba Dragonwell Extended Edition还具备诸多特色功能，例如 JWarmup、ElasticHeap 等等。这些特性在阿里巴巴内部得到了广泛应用，解决了很多生产实践中的痛点，为阿里巴巴 Java 业务的稳定运行立下了汗马功劳，可以说是 Alibaba Dragonwell 的独门武器。

**2.5 新技术的快速采用**

基于阿里工程实践，Alibaba Dragonwell 会选择移植高版本 Java 的重要功能，这些移植功能已经在阿里内部被大规模部署，用户都可以免费使用，而不用等 OpenJDK 下一个 LTS 版。

随着 Alibaba Dragonwell 的迭代，越来越多的新特性将会被开源。随着使用 Alibaba Dragonwell 的 Java 应用日益增多，在源码和构建工程均开源的情况下，**我们如何保证用户所使用的 Alibaba Dragonwell 确实是出自阿里云编译器团队？如何保证我们的 JDK 在发布构建过程中没有被篡改呢？**

_03_ 我们为什么要做供应链安全
-----------------

软件供应链是在整个软件开发生命周期 (SDLC) 中涉及应用程序，或是任何方式在其他开发中发挥作用的任何事和物。而 JDK 在 Java 软件的供应链中毫无疑问占据了核心地位。JDK 是 Java 软件开发的基础，它提供了软件运行环境、调试和编译的工具以及丰富的 API。如此核心的工件，如果不能保证他的来源可靠，一旦遭到恶意者的篡改并加以传播，严重的后果可想而知，因此供应链安全势在必行！

说起软件供应链安全，首先想到的便是当下较为火热的软件供应链安全标准 SLSA。SLSA 是一个标准和控制清单的安全框架，用于防止篡改、提高完整性以及保护项目、业务或企业中的包和基础设施。借此软件生产商使其软件更安全，消费者可以根据软件包的安全状况做出选择。其旨在为开发人员和企业提供行业标准、公认且商定的保护和合规级别，任何人都可以采用和使用。用户可以以此来要求所依赖的软件是特定的 SLSA 级别，企业也可以此作为指导原则来加强内部供应链。

我们之所以选择参考 SLSA 来指导我们加强供应链，主要因为它有两点重要的原则：

*   软件供应链中任何软件工件只有在被“受信任的人”的明确审查和批准之后才能进行修改。  
    
*   软件工件可以追溯到原始的来源和依赖项。  
    

_04_ 我们是如何做的
------------

**最好的合作伙伴之一 Eclipse Adoptium**

阿里云于 2020 年加入 Eclipse Adoptium 社区，是 Eclipse Adoptium 工作组的战略基石成员，参与 Eclipse Adoptium 社区治理，为 Java  Ecosystem 提供完全兼容的、基于 OpenJDK 的高质量 JDK 发行版。Alibaba Dragonwell 现有的发布工程大部分都基于 Adoptium 进行了适配和小幅度的开发。当然，我们和社区一直都保持着紧密的合作，对于较为通用的优化和 Bug 修复我们也都贡献给了社区。

**提升 SLSA 的实践**

Alibaba Dragonwell 已经达到了SLSA v0.1 specification 所描述的 2 级要求。近年来，我们致力于参与 Adoptium 社区的建设，基于 Adoptium 社区开源的设施，我们进行了一定量的改动和适配，最终得以实现 Alibaba Dragonwell 的安全等级提升。发布流程概况如下图所示。

 ![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/77fd01bc-830e-44e4-a0bf-c84d286244c3.png?raw=true) 

**SLSA 1 级**

等级 1 意味着我们的构建必须完全脚本化/自动化并生成出处。我们在发布版本中自动生成二进制文件、SBOM（软件材料清单）文件和校验文件。以下是等级 1 的达成条件：

*   构建 \- 脚本化的构建  
    

所有构建步骤必须被定义在类似于“build script”的地方。如果有需要手动执行的地方，那只能是调用构建脚本。我们把 pipeline 流程都完整的定义在了 ci-jenkins-pipelins，下游的构建工程被定义在了 openjdk-build。我们发布期间需要做的只有填写对应的参数和触发工程，然后所有所需的文件都会生成。

*   出处 \- 可用

软件出处需以消费者接受的格式提供给消费者，且格式应该是符合 SLSA 规定的。但如果有另一种格式，生产者和消费者都同意并且认为它能满足所有其他要求，方才能使用。

我们发布构建会以 OWASP CycloneDX 格式生成 SBOM 文件，该文件包含了全部的构建信息，包括环境、组件信息、构建指令和参数等。

**SLSA 2 级**

等级 2 需要我们进行版本管理，以及创建生成经过身份验证的出处的托管构建服务。我们通过 Github 管理我们的源码和版本标签，发布期间在 jenkins 实例上自动对发布产物进行了签名。以下是等级 2 的达成条件：

*   源码 \- 版本管理

这要求源码的每笔修改都应在版本控制系统中进行跟踪，包括记录更改历史记录以及无限期引用此特定的、不可变的提交的方法。

我们在 Github 管理源码，例如 Dragonwell 8。源码中的每一笔提交都必须满足如下的提交格式，否则会被 CI 测试拦截。提交的代码会经过仔细的审查和具体的 CI 测试，通过审查和验证之后，每笔提交都会被记录在历史记录中。通过历史记录我们能获取到对应的 Issue 和 Pull Request 地址。每次发布的时候，我们都会对发布版本创建标签，标签会包含版本号和 dragonwell 版本(extended/standard)。

*     
    

```
\[<tag>\] <One-line description of the patch> Summary: <detailed description of the change> Test Plan: <how this patch has been tested> Reviewed-by: <Github IDs of reviewers> Issue: <full URL or #github_tag>
```

*   构建 \- 构建服务  
    

所有的构建都应该以服务的形式，而不是在私人的工作目录下进行。我们在我们的 Jenkins dragonwell-jenkins 上构建相关文件。所有的文件都会通过 Jenkins 工程上传 Github、阿里云 OSS，并且会自动生成相应的容器镜像，发布在阿里云 ACR 仓库和 DockerHub 仓库。另外，standard 版本还将会发布在 Adoptium Marketplace。

*   出处 \- 身份认证  
    

消费者可以验证出处的真实性和完整性。这应该通过来自私钥的数字签名来进行，只能由生成出处的服务访问。GPG 密钥存储在阿里巴巴编译器团队的 jenkins 实例上，发布时我们会使用该密钥进行签名，用户也可通过我们的验证工程 validate-signature，验证签名是否属于我们团队。

*   出处 \- 服务生成  
    

我们的参数都在工程构建前进行了设置，匿名用户无执行权限，因而我们的构建不会被注入或者更改不安全的内容。

我们的 SBOM 文件和验证文件由构建自动生成，GPG 的密钥存储在 jenkins 实例上，该密钥不对外公开。因此，我们的构建服务是安全可控的。

_05_ 未来展望
---------

我们这就结束了吗？很显然这并不是，现在只是 Alibaba Dragonwell 迈向更高级别SLSA的开始。在过去的几年，我们跟 Adoptium 社区有非常紧密的合作。未来我们也将会携手 Adoptium 社区继续努力，提升 Alibaba Dragonwell 的产品质量和合规等级。

相关链接：

1\. SLSA v0.1 specification：

__[http://slsa.dev/spec/v0.1/levels](http://slsa.dev/spec/v0.1/levels)__

2\. ci-jenkins-pipelins：

__[https://github.com/dragonwell-releng/ci-jenkins-pipelines](https://github.com/dragonwell-releng/ci-jenkins-pipelines)__

3\. openjdk-build：

__[https://github.com/dragonwell-releng/openjdk-build](https://github.com/dragonwell-releng/openjdk-build)__

4\. OWASP CycloneDX 格式：

__[https://owasp.org/www-project-cyclonedx](https://owasp.org/www-project-cyclonedx)__

5\. dragonwell 8：

__[https://github.com/alibaba/dragonwell8](https://github.com/alibaba/dragonwell8)__

6\. dragonwell-jenkins：__[http://ci.dragonwell-jdk.io/](http://ci.dragonwell-jdk.io/)__

7\. DockerHub 仓库：__[https://hub.docker.com/r/alibabadragonwell/dragonwell/tags](https://hub.docker.com/r/alibabadragonwell/dragonwell/tags)__

8\. Adoptium Marketplace：__[https://adoptium.net/marketplace/](https://adoptium.net/marketplace/)__

9\. validate-signature：

__[http://ci.dragonwell-jdk.io/job/build-scripts/job/release/job/validate-signature/build](http://ci.dragonwell-jdk.io/job/build-scripts/job/release/job/validate-signature/build)__

—— 完 ——