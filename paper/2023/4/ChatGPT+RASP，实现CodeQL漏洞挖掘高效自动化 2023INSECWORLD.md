# ChatGPT+RASP，实现CodeQL漏洞挖掘高效自动化 | 2023INSECWORLD
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/09affddc-19c6-4529-ae58-0416e73162c7.gif?raw=true)

_传统用CodeQL进行漏洞挖掘，生成的结果误报较多，影响效率；那么，如果融入ChatGPT和RASP呢？_

3月21日，由Informa Markets主办的**2023 INSEC WORLD 世界信息安全大会**以“聚焦安全 打造多元新格局“为主题，在西安国际会展中心会议楼拉开帷幕。作为一次汇聚各界意见领袖和技术专家的行业开年盛会，大会邀请到逾50位海内外优秀行业大咖分享安全实践经验及最新技术。

**深信服创新研究院安全技术专家高勇杰**受邀在**「漏洞与攻防安全」分论坛**中进行演讲，分享议题为**《CodeQL漏洞挖掘之旅》**。议题详细讲解CodeQL使用思路，以及一个区别于主流使用的利用CodeQL进行半自动化挖掘漏洞的方式。结合当下热门的chatGPT工具，分享了在CodeQL的使用和工作方式的新感悟，可以有效提升漏洞挖掘效率。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/949a085b-d2bb-4198-8b36-e0c19210ad40.jpeg?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/dd7861fb-a844-4011-87c9-ff8601398514.gif?raw=true)

**区别于主流使用CodeQL的漏洞挖掘方式**

在演讲中，高勇杰使用SecExample开源靶场进行实际案例的演示，讲解不将CodeQL作为扫描工具使用，而是利用CodeQL的分析能力，协助进行代码审计，尽可能地减少在代码审计时的工作量。  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/b8a7bdda-de1a-4c81-a060-17570ecf2745.jpeg?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/1d0a74bf-ea54-4ad4-b7db-5ff60543b7d4.jpeg?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/06c3c506-511d-4a69-b78b-4e44839483e0.gif?raw=true)

**CodeQL+ChatGPT+RASP，实现高效自动化**

高勇杰总结道，CodeQL目前还存在着一些缺陷，例如QL规则需要不断迭代、扫描结果存在误报等等，仅使用CodeQL也无法完成完整的代码审计流程闭环等问题。  

针对以上问题，高勇杰也分享了一套借助ChatGPT与RASP技术的全自动化漏洞挖掘方式，不仅可解决QL规则的迭代、扫描规则误报问题，且完成了代码审计流程的闭环。

CodeQL做初步审计，提取关键代码块后交由ChatGPT验证，并基于RASP技术避免了POC生成不稳定对结果的影响，使结果更加稳定、可靠。最终实现高效、精准的漏洞挖掘。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/e5cc6161-3edc-4e94-b6de-73e5033d6e74.jpeg?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/bb45cd60-061d-42bc-99fe-6074a6bd6672.gif?raw=true)

  
深信服千里目安全技术中心-创新研究院一直致力于安全和云计算领域的核心技术前沿研究，推动技术创新变革与落地，拥有安全和云计算领域500+ 专利，实现攻击和检测技术的相互赋能，并及时把能力输入到业务线中，实现自身产品的迭代优化。未来，深信服千里目安全技术中心也将不断提高专业技术造诣，深度洞察网络安全威胁，持续为网络安全赋能。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/686d229c-5679-4e24-8650-8a1364b3cd90.jpeg?raw=true)