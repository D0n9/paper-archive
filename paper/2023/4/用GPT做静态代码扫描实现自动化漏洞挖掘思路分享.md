# 用GPT做静态代码扫描实现自动化漏洞挖掘思路分享
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/7cbfe4cb-7843-4497-89df-53ea47438339.gif?raw=true)
戳上面的**蓝字**关注我吧！

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/8b31b2eb-9999-4423-8ef5-a4a00cb96cae.gif?raw=true)

* * *

之前一直有一个想法，就是用GPT来做代码审计和反混淆的事情，愈是跟对抗相关的事情愈有训练的意义。

**01**

**GPT反混淆方向**

还是用之前介绍过的工具Cursor来做

话术如下：

```diff
请将下面的字节码用Java代码重写，要求：

- 逻辑与原代码保持一致
- 使用更友好的变量名
- 不存在的函数不需要生成
- 不需要生成类，只要生成函数即可。
- 不要求代码能编译通过
- 不要删除或者省略任何代码
- 添加中文注释
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/9c3dd82e-0c7b-41be-a10b-26b90b3e2476.png?raw=true)

嗯？混淆？我看不懂机器还看不懂吗！（手动狗头![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/b33b594b-0795-4a53-83d1-1ae2d3ff45de.png?raw=true)
）

**0****2**

**结合GPT进行静态扫描**

从推上看到一个提出使用Embeddings的方式将项目喂给GPT的构思图

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/1f63c91c-4f6b-4d31-bd73-fdbaa750feec.png?raw=true)

**初始阶段**：

Script或者App通过读取数据块，如通过Soot生成的代码数据块保存在本地的CSV文件中。

在为读取到的数据块生成向量embeddings调用到OpenAI的Embedding模块的API。

继续将向量embeddings插入到向量数据库中。

**查询阶段**：

Search App通过生成查询字符串到OpenAI，并查询相关向量Embedding。

再通过查询到的向量Embedding在数据库中搜索

**展示阶段**：

从搜索应用程序中获取搜索结果

使用搜索结果构建prompt，并最终调用gpt model

Github上有这么一个项目gpt-index(也叫llama-index)：https://github.com/jerryjliu/llama_index

实现了突破4096 tokens限制的功能，还集成了长文本摘要、个人数据库自然语言查询等  

**步骤**：用OpenAI的embedding模型将文本编程数字向量，再将向量结果存入CSV文件或者DB数据库，最后搜索的时候先将搜索关键词通过embedding模型转换然后再去数据库中匹配相似度结果即可。

在python下安装相关环境

```sql
!pip install llama-index
!pip install langchain
```

装好之后编写如下代码进行查询

```python
from llama_index import SimpleDirectoryReader, GPTListIndex, readers, GPTSimpleVectorIndex, LLMPredictor, PromptHelper
from langchain import OpenAI
import sys
import os
from IPython.display import Markdown, display


def construct_index(directory_path):

    # set maximum input size
    max_input_size = 4096
    # set number of output tokens
    num_outputs = 2000
    # set maximum chunk overlap
    max_chunk_overlap = 20
    # set chunk size limit
    chunk_size_limit = 600
    # define LLM
    llm_predictor = LLMPredictor(llm=OpenAI(temperature=0.5, model_name="text-davinci-003", max_tokens=num_outputs))
    prompt_helper = PromptHelper(max_input_size, num_outputs, max_chunk_overlap, chunk_size_limit=chunk_size_limit)
    documents = SimpleDirectoryReader(directory_path).load_data()
    index = GPTSimpleVectorIndex(
        documents, llm_predictor=llm_predictor, prompt_helper=prompt_helper
    )
    
    index.save_to_disk('index.json')
    
    return index


def ask_ai():
    index = GPTSimpleVectorIndex.load_from_disk('index.json')
    while True:
        query = input("What do you want to ask? ")
        response = index.query(query, response_mode="compact")
        display(Markdown(f"Response: <b>{response.response}</b>"))

if __name__=='__main__':
    os.environ["OPENAI_API_KEY"] = "sk-xxxxxxxx"
    construct_index("context_data/src")
    ask_ai()
```

其中construct_index的参数就是当前目录下需要审计的代码路径，"sk-xxxxxxxx"就是在OpenAI官网申请到的Token

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/3e2514c7-bef8-4749-932c-fe2240717ccc.png?raw=true)

执行进行查询

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/a5761906-bc6f-4df2-9cc7-6190cc6687eb.png?raw=true)

总结一下，GPT3的API用来处理这种大型程序的程序语言还是太拉了，根本无法准确获取相关内容，如果要做估计还是得通过单个文件结合GPT-4来定位。

稍微翻阅了下文档，当中有个GithubRepositoryReader，不知道能不能用这个来识别代码

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/5d5c4505-cf3f-4ac4-a502-9f4591eb2956.png?raw=true)

后面问了下网上的大佬，说是估计一次性喂的项目有点大，获取可以先根据路径建立自己的Tree索引，之后如果想问漏洞，就将相关的文件embedding后投喂进去。

举个例子：

RCE的漏洞是有调用Runtime.exec等危险函数的，则可以提前用字符串匹配工具将包含危险Sink的相关文件整合投喂给GPT，再由GPT帮你定位漏洞，如果危险函数在Utils工具类里面，则可以通过调用关系分析工具(许少的jar-analyzer工具中有实现，可以参考学习)构造CallGraph，将Request请求的Source和Sink之间的所有函数整合在一个Java文件中，并用Embedding方式绕过Token限制，再进行漏洞查询。之后再调教一下Prompt，要求Source跟Sink必须有一条可达路径，以此来确定唯一可控的路径。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/03ee72d5-65a6-421b-9402-b8a5bf24075c.png?raw=true)

如上图所示，光看是看不出有啥漏洞，需要进入到HttpUtils.URLConnection(url)中继续查看，这时候可以借助其他工具，或者把HttpUtils类也加入construct_index的指定目录下。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/15244632-deb0-495e-b83f-f234c9d14b67.png?raw=true)

当然，GPT-index还能结合Nmap-Scan来分析，已经有网友实现了：https://github.com/morpheuslord/GPT_Vuln-analyzer

估计更多安全方向的应用已经在实现中了，GPT-index这个工具确实能让很多方向结合GPT来，比如之前看在博客上看到的截图@nishuang

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/ddc1de7f-3185-4274-bd02-5067943a401f.png?raw=true)

把所有的对话，以及心理细节记录下来存入CSV或数据库中，再由gpt-index进行token解析，就能创造出一个有用你说话和生活习惯的自己！数字生命计划雏形？？？

以上就是我这两天了解的一些知识，很片面，但是却很有意思，希望未来GPT能提升更多的效率吧。关于更多的资料和案例在这里可以找到：https://github.com/openai/openai-cookbook

而且甲方真实环境中很多地方的代码不是完整的，有可能是没有依赖、不是完整的maven项目等，这种方式是很难用CodeQL去做扫描的。

**03**

**后续！！！**

周一来公司上班后还是恋恋不忘，一直想将GPT结合在自动化漏洞发现上，但是不管怎么测试总感觉蠢蠢的。于是中午休息时间再看了下代码，直到我发现我模型选错了之后...

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/67398fcf-1a2b-443d-b8b4-02068191f81a.png?raw=true)

更换模型为code-davinci-002

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/c1472f69-73fa-47cb-94f9-881521e2924c.png?raw=true)

终于成功了，虽然给出的demo中确实没有安全漏洞存在，于是乎我找一个存在安全漏洞的靶场来做。

当我又问它，有什么漏洞的时候，API给我挤爆了？？？？

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/8fdf45a0-0122-4b33-8ad4-12cf9e4e50c1.png?raw=true)

问了师傅得知是要用英文提问和回答，不过效果不太好，我这套是jspxcms，是有后台ssrf漏洞的，但是换了英文提问后还是提示没有安全风险（就算给出入口方法也没用，唉，累了），只好继续换Demo试试，看看能不能测出来。

```typescript
Please provide a detailed analysis of the source code for any vulnerabilities, and if any are found, please specify the file path and line number.
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/f4e5eabe-2356-476e-a5e6-469a06b3f448.png?raw=true)

非常正确！！！！

如果把关键逻辑改成工具类呢，比如下图中的HttpClient.URLConnection方法

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/42b519ae-770f-43eb-b4e5-4be4f365852c.png?raw=true)

GPT一样可以测出来

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/1d96581f-5779-475c-9af1-b0405bcdb70f.png?raw=true)

目前来说最简单的方式，就是通过分模块批次喂给GPT，它能为你找到漏洞，还有一个就是我前面提到的自己编写程序将所有方法调用整合在一个文件中或者将相关的文件结合在一起投喂给GPT。不过这个方向我相信未来的前景还是挺多的，不过代码审计的门槛或许就拉低了，你只需要学过编程，就能看懂其他语言代码的大概逻辑，剩下具体的语法糖框架部分，那就交给GPT吧！

还有一个可突破的方向，就是我之前研究的Sourffle静态分析那篇文章，其中提到了DSL是一种声明式语言，只需要将相关的文档喂给GPT，再通过GPT来编写对应的查询规则即可，如：“请根据现有的方法，生成一个污点跟踪规则，Source点是xxx，Sink点是xxx，请将生成好的规则给我”。然后将Response直接带到Sourffle引擎中进行查询即可，即可关联上SAST。

创作不易，如果觉得这篇文章不错，欢迎转发点赞加关注~

**04**

**Reference**

\[1\].https://twitter.com/chuangbo/status/1631461656151887873

\[2\].https://uxdesign.cc/i-built-an-ai-that-answers-questions-based-on-my-user-research-data-7207b052e21c

\[3\].https://gpt-index.readthedocs.io/en/latest/

\[4\].https://github.com/jerryjliu/llama_index

\[5\].https://mp.weixin.qq.com/s/-FhFEgi5YGSGmXFbRWpPHA

* * *

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/a714a889-fe27-410b-9f2a-a3e44350cda6.png?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/969b7a31-0b11-46cc-9a53-f1ddf7fa6740.png?raw=true)