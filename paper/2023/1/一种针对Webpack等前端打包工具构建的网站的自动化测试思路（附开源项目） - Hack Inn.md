# 一种针对Webpack等前端打包工具构建的网站的自动化测试思路（附开源项目） - Hack Inn
![](https://data.hackinn.com/photo/xcxstwm/banner.png)

> 由于传播、利用此文所提供的信息而造成的任何直接或者间接的后果及损失，均由使用者本人负责，雷神众测以及文章作者不为此承担任何责任。
> 
> 雷神众测拥有对此文章的修改和解释权。如欲转载或传播此文章，必须保证此文章的完整性，包括版权声明等全部内容。未经雷神众测允许，不得任意修改或者增减此文章内容，不得以任何方式将其用于商业目的。

作者：Poc Sir 全文共分为六个片段，首发于“雷神众测”公众号具有原创声明。本文全长1.3万余字，预计阅读时间30分钟，请做好准备。

> 开源项目名称：**Packer Fuzzer** 版本号：V1.1
> 
> 地址：[https://github.com/rtcatc/Packer-Fuzzer](https://github.com/rtcatc/Packer-Fuzzer)

### 0x01 前言&初衷

半年之前当疫情第一次席卷全球之时，法国也选择了封城，早已自我封闭许久的作者自然也延长了自己的“闭关”时间。我独坐小屋之中早已日夜颠倒、昼夜不分，如今回想起来已经没有了对那时窗外景色的记忆，一切都是灰蒙蒙的。压抑、惧怕、无奈笼罩着一切，唯有坐在电脑之前成功渗透一个个网站才能让身在异国他乡的我有所开心。走到电脑桌前，打开网站：又是一个Webpack的站，熟练地单击右键，点击“审查元素”调出开发者工具，来到“资源文件”一栏。找到对应的JS文件，点击一键格式化按钮，然后快速的看起上万行的JS脚本，从中寻找出最有用的信息。或许是我早已习惯了吧，JS行数虽多但也不影响整体测试的效率，虽然有过不止一次想写工具的想法，但可能是因为“懒”吧，自己拒绝了自己，觉得就这样挺好。

随后在一次和我徒弟的聊天之中，我发现虽然自己渗透测试这些站很快速，但是对一些新人来说如此庞大的内容还是十分不友好的，并且如果将时间浪费在这些可由工具代替的信息搜集环节上周而复始的劳做岂不是太无聊了，那么说做就做吧！老话说：前人栽树、后人乘凉，在互联网“白嫖”了如此之久的我也是应该回馈于互联网，回馈各位亲爱的读者朋友们了。笔者自愧，这是我第一次带领自己的小伙伴们开发扫描类工具，与其他扫描器不同，打包器的解析是未曾有前人系统化做过的，目前的第一版本必然是不完美的，若有不足之处还望前辈您赐教。Poc-Sir代表开发团队谢谢各位的耐心阅读，您们所提的任何疑问都将会是价值连城的！

可能是最近两年吧，国内越来越多的站开始用起了前端打包工具，它一下子火了。这是一类很好玩的工具，它将开发者所需要的全部资源都打包在了一起，后期只要按需调用即可，节省了不少开发时间；与此同时这还是一类让我们这些“搞安全”的人忍不住偷笑的工具，正是因为他将所有的资源都打包在了一起，我们便可以不费吹灰之力拿到整个站的全部资源文件，包括所有的API。举一个不太恰当的例子，开发A写了一个未授权访问的API，安全研究员B“含着泪”把这个漏洞提交了上去赚了3000块钱。API不拿何以拿SHELL，当我们拿到一个目标，若能快速提取出API及其对应的参数并加以模糊测试，那真是太顺心了，而如此美好的事儿得益于前端打包工具的流行不再是一个白日梦了。

Packer Fuzzer工具目前支持自动化提取JS资源文件并利用现成规则和暴力提取模式提取其中的API及对应参数，在提取完成之后支持对：未授权访问、敏感信息泄露、CORS、SQL注入、水平越权、弱口令、任意文件上传七大漏洞进行模糊高效的快速检测。在扫描结束之后，本工具还支持自动生成扫描报告，您可以选择便于分析的HTML版本以及较为正规的doc、pdf、txt版本。而且您完全不用担心因为国际化带来的语言问题，本工具附带五大主流语言语言包（包括报告模板）：简体中文、法语、西班牙语、英语、日语（根据翻译准确度排序），由我们非常~不~专业的团队翻译。然而由于时间、实例及其他因素限制，目前第一版工具只对Webpack打包器的规则做了优化，其他打包器规则敬请期待！

也希望读者朋友们在遇上不一样的例子时，能为我们提供新的规则；在遇上新的报错时，能及时反馈给我们，这也是我们开源的目的。我们是渺小的，我们只不过是为大家规划了一个框架，真正能让此工具完美还得靠各位前辈们的帮助，Poc-Sir再次代表小伙伴们谢谢各位的支持！

![](https://data.hackinn.com/photo/PF-A/demo-terminal.png)

### 0x02 JS文件提取

#### 0X021 主要JS提取

前端打包工具的成果都将在网站上以JS文件的形式呈现，故要想解析此类网站要做的第一步便是提取此站所有相关的JS文件。可能初入安全不久的读者朋友们对此类打包工具不太了解，您可以打开雷神众测的官网（[https://www.bountyteam.com），并查看其源代码，您便可以看到类似如下简化之后的内容，这是一个非常标准的Webpack站点：](https://www.bountyteam.com），并查看其源代码，您便可以看到类似如下简化之后的内容，这是一个非常标准的Webpack站点：)

```


2.  `<!DOCTYPE html><html  lang=en><head><meta  charset=utf-8>`
3.   `<title>雷神众测</title></head>`
4.   `<body><noscript><strong>We're sorry but dasbounty doesn't work properly without JavaScript enabled. Please enable it to continue.</strong></noscript>`
5.   `<script  src=/js/chunk-vendors.d1m05ef2.js></script>`
6.   `<script  src=/js/app.d1m0h997.js></script></body>`
7.  `</html>`


```

前端打包工具所生成的是一个纯JS的网站 —— 源码之中并无多余的HTML内容，所有的前端内容都依赖于浏览器对JS文件的解析，并且此类网站之中通常存在一行`noscript`提示告诉没有启用JS的访问者启用JS功能才能正常浏览网站。并且Webpack所生成的js有他的命名规律（并非全部如此），一般都为`xxx.随机字母数字.js`的两段式，且每一个JS文件具体内容均以`webpackJsonp`开头。

Packer Fuzzer工具首先调用`lib/ParseJs.py.py`文件来处理用户输入的目标站点网页中html所嵌入的最基础JS文件，首先在第47行处程序会将所有的HTML注释内容全部去掉`<!-- XXX -->`，这是因为本模块将在随后使用`bs4`库来对所获取到的网页元素解析，而在某些JS文件被注释时是无法被正常解析。随后自49行开始程序将会遍历当前HTML中的所有`<script>`标签并提取其中的`src`资源路径，同时判断标签内是否有JS代码，若有则一并保存，防止漏掉任何一行JS代码（有些开发者喜欢将app等主js文件的内容直接写在html的`<script>`标签中）：

![](https://data.hackinn.com/photo/PF-A/image-20201008180556268.png)

接着针对一些网站会把JS地址写在`<link>`标签中的情况，程序也针对此做了兼容：

![](https://data.hackinn.com/photo/PF-A/image-20201008181258749.png)

在提取完当前页面全部JS路径之后程序会根据URL路径规则针对不同的情况对JS文件的`src`资源路径进行拼接：

| 资源格式 | 处理方式 |
| --- | --- |
| https:// 或 http:// | 完整URL路径，不做任何拼接处理 |
| ../ 或 ../../ （多级） | 含有一个或多个上级路径，循环拼接直至根域名 |
| // 自适应格式 | 更具当前协议（http或https）进行添加 |
| / 单一开头 | 使用根域名 \+ 当前src值方式拼接 |
| 无任何开头 | 使用当前路径 \+ 当前src值方式拼接 |
| ./ 方式开头 | 删去“./”并按照“无任何开头”方式处理 |

具体通过调用位于92行处的`dealJS()`函数实现：

![](https://data.hackinn.com/photo/PF-A/image-20201008185804105.png)

另外针对JS提取时因种种原因造成遗漏、提取不全的情况（基本不可能），本程序支持使用`-j（或 --js)`的方式在命令行中额外附件多个JS文件的URL路径，并一起存入`extJSs`数组中以便使用`downloadJs()`函数统一下载：

```


1.  `DownloadJs(self.jsRealPaths,self.options).downloadJs(self.projectTag, res.netloc,  0)`
2.  `extJS =  CommandLines().cmd().js`
3.  `if extJS !=  None:`
4.   `extJSs = extJS.split(',')`
5.   `DownloadJs(extJSs,self.options).downloadJs(self.projectTag, res.netloc,  0)`


```

#### 0X022 JS过滤及下载

在程序开始下载文件时，首先会调用`lib/ParseJs.py`文件下的`jsBlacklist()`函数，此函数用于过滤一些非打包器所生成的无关紧要文件，例如：第三方js文件、公共cdn缓存文件等，用于提高后期提取、识别准确率。其核心代码如下：

![](https://data.hackinn.com/photo/PF-A/image-20201008221905542.png)

可以看到文件会从黑名单列表中读取黑名单JS名称以及黑名单域名并在数组中加以排除，其配置文件位于程序根目录下，名为`config.ini`：

```


1.  `[blacklist]`
2.  `filename = jquery.js,flexible.js,data-set.js,monitor.js,umi.js,honeypot.js,.min.js,angular.js`
3.  `domain = api.map.baidu.com,alipayobjects.com`


```

在下载完成之后程序会对JS文件做去重处理，并将JS文件的基础信息写入对应缓存数据库的`js_file`表中方便后期调用，其表说明如下：

| 列名称 | 类型 | 说明 |
| --- | --- | --- |
| id | INT 主键 自增 | 自动排序编号 |
| name | TEXT | JS名称 |
| path | TEXT | JS的URL路径 |
| local | TEXT | JS的本地保存名称 |
| success | INT | 是否下载成功，成功置1 |
| spilt | INT | 异步加载ID号，默认留空 |

一些读者朋友到此可能就会有所疑问为何要大费周章自己去解析JS文件呢，直接用`Selenium`库调用浏览器模拟加载并提取出JS文件不是更加方便，为何要这样多此一举？首先`Selenium`的浏览器支持安装比较麻烦，在模拟的同时也需要消耗用户大量的内存资源，另外最为重要的是模拟器解析也无法提取出全部JS文件，请您细听我随后道来。

#### 0X023 针对异步JS处理

当我们在实际测试过程中，我们发现我们过于天真了，由于一直依赖浏览器控制台会自动加载被调用的文件的特性，作者直到开发此工具之前从未发现打包工具还存在一个非常实用的功能：异步加载（或者称为按需加载）— 用于将体积较大的js文件拆分为多个小文件，当某个页面需要相关JS文件时才会进行调用，达到减轻网络负担的目的。由于是按需加载JS，故在我们来到某个页面之下时，浏览器中所加载的文件并不是完整的，这也是我们不用`Selenium`的重要原因，到头来还是自己脚踏实地的解析靠谱。

其JS异步加载代码多存在于`manifest`或是`app`开头的js文件之中，其中大部分特征如下（X、Y、Z代表一位小写纯字母）：

```


1.  `X.src = Z.p +  "static/js/"  + Y +  "."  +  {`
2.   `0:  "d1m0ecbdb81953f82a",`
3.   `1:  "d1m0qe11bb81zz3f81"`
4.  `}[Y]  +  ".js";`


```

或是(或其他类型不一一举例)：

```


1.  `X.src = Z.p +  "static/js/"  + Y +  "."  +  {`
2.   `"gre5g4e":  "d1m0ecbdb81953f82a",`
3.   `"y5h1tr5":  "d1m0qe11bb81zz3f81"`
4.  `}[Y]  +  ".js";`


```

控制台模拟运行这两个示例结果如下：

![](https://data.hackinn.com/photo/PF-A/image-20201008225640498.png)

在知道了其代码特征之后我们便可开始提取了，首先在`lib/Recoverspilt.py`的`checkCodeSpilting()`函数内第71行处：

```


1.  `if  "document.createElement(\"script\");"  in jsFile:`


```

我们通过判断JS文件中是否存在创建`<script>`dom标签的代码，若无则必然是不存在JS异步加载的，通过此判断我们可以很快速的排除掉一批JS文件，省去一个个正则匹配的麻烦。接下去在第73行处我们使用了如下正则表达式进行提取：

```


1.  `pattern = re.compile(r"\w\.p\+(.*?)\.js", re.DOTALL)`
2.  `jsCodeList = pattern.findall(jsFile)`
3.   `for jsCode in jsCodeList:`
4.   `jsCode = jsCode +  ".js\""`


```

最终提取的代码可归纳为如下(Y可为a-z任意一个字母）：

```


1.  `"XXXXXX"  + Y +  "."  +  {`
2.   `......`
3.  `}[Y]  +  ".js";`


```

在成功提取之后程序会将其传入`jsCodeCompile()`函数中进行进一步调用，在此函数中程序首先会将提取内容重新拼接为一个可运行的新JS函数，其代码实现如下：

![](https://data.hackinn.com/photo/PF-A/image-20201008233056553.png)

最终拼接而出的可实际运行函数为：

```


1.  `function js_compile(Y)  {`
2.   `js_url=  "XXXXXX"  + Y +  "."  +  {`
3.   `......`
4.   `}[Y]  +  ".js";`
5.  `return js_url}`


```

为了防止“黑”吃“黑”，在第52行处程序首先会判断是否存在可被命令执行的函数关键字，若存在则直接放弃运行（~不会有人这么无聊吧…~）。之后程序将会使用`execjs`模块调用`node.js`(需先安装对应软件，若JS模拟引擎不为`node.js`则会报错，无法成功解析)运行生成的函数并获取运行结果：

![](https://data.hackinn.com/photo/PF-A/image-20201008233409647.png)

在JS异步代码解析完成之后程序将JS异步结果的一些信息写入对应缓存数据库的`js_split_tree`表中方便后期调用，其表说明如下：

| 列名称 | 类型 | 说明 |
| --- | --- | --- |
| id | INT 主键 自增 | 自动排序编号 |
| jsCode | TEXT | JS异步代码内容 |
| js_name | TEXT | 被解析的JS名称 |
| js_result | TEXT | 异步解析结果 |
| success | INT | 是否解析成功，成功置1 |

在成功获取全部异步JS的文件名之后会调用`getRealFilePath()`函数获取完整URL路径并进行下载。但如果此时您认为异步加载反解析模块到此结束，那就是您太天真了。

在JS异步加载的情况中，如同我们前面所介绍的以纯规律数字开头的两段式命名js文件较多，例如：`0.ef6vbi5iopi561j8.js、1.ef6vbi5iopi561j8.js、2.ef6vbi5iopi561j8.js ...`此类JS的文件命名有规律可循，即使无法运行原生代码进行提取，也可使用暴力破解的方式将其下载下来。暴力破解功能位于`checkSpiltingTwice()`函数内，首先程序会将所有下载完毕的JS文件名全部提取一遍并以“.”做为分割符对其处理：

![](https://data.hackinn.com/photo/PF-A/image-20201013153154448.png)

在Packer Fuzzer工具工具中为防止JS文件名重复（例如同时存在`/js/index.js`和`/static/index.js`文件），程序会自动将原JS文件做添加随机tag的方式重命名处理（例如原远程存储文件名为`1.index.js`则本地将以`tag + . + 1.index.js`的方式进行处理，最终本地保存的文件名将为：`15zfe.1.index.js`），故在使用“.”分割处理后符合Webpack文件命名要求的文件长度将有四段：TAG、文件名、随机统一id、JS后缀。我们将所有符合规则的JS文件名只保留后两段（例如`15zfe.1.index.js`只保留有用的`.index.js`部分）并存入数组中，做去重处理。针对去重之后有三个及以下的结果，我们将会对其添加500以内的纯数字进行循环拼接：`1.ef6vbi5iopi561j8.js ... 500.ef6vbi5iopi561j8.js`。而针对三种以上的结果，我们认为其爆破结果意义不大，大概率为纯随机JS文件（例如`gre5g4e.d1m0ecbdb8195.js`）爆破成功可能性较低，故我们采取随缘算法只取第一条结果进行拼接处理（~挖漏洞是一种缘分，漏洞就在那边，你若没挖到就说明你们暂时无缘~）：

![](https://data.hackinn.com/photo/PF-A/image-20201013160437761.png)

一般来说异步加载的JS文件在5至200个左右，故笔者认为选择五百以内的数字进行爆破将不会漏掉任何一个文件出现爆破不全的情况，也不用过多的进行空爆破。在爆破名单生成完毕之后工具会调用`/lib/Recoverspilt.py`文件中的`GroupBy().stat()`函数进行实际爆破处理。当然若直接对五百以内的文件进行逐一爆破是非常不明智的行为，因为最高将会有1500（3x500）次的http请求操作，会消耗大量网络资源，也容易触发目标网站的风控机制。得益于此类异步加载的JS文件都是顺序命名的（1、2、3）故我们将请求名单每20分为一组，并以组为单位依次爆破，若第N组开始请求结果连续为不存在则自动退出不进行后续爆破操作。比如在某网站中，最高编号的一个异步文件为`165.ef6vbi5iopi561j8.js`，则工具只会对前9组（2x9=180)进行爆爆破，在第9组内自165之后的请求都未成功获取到有效文件内容，故不会进行第10组请求从而大大降低了无效爆破请求。当然有些网站坏的很，明明文件不存在，硬要返回`HTTP 200`和一个像模像样的网页企图蒙蔽我们的双眼。不过应对方式也很简单：JS文件的Content-Type可以为`text/javascript`或是`application/javascript`等但绝对不可能为`text/html`（也不是不可能，这个时候就是配置不当了，若js内容可控则可能造成XSS漏洞），故只要将返回头中存在`text/html`的响应全部当做`HTTP 404`处理即可：

```


1.  `if  "text/html"  not  in text.headers['Content-Type']:`
2.   `flag =  1`


```

不过很遗憾的是针对类似`sd7g9.ef6vbi5iopi561j8.js、b2tr9.rg5rr85iotr5grt5.js`等的前段随机或者两段均为随机内容毫无任何规律可循的JS文件命名方式，其解析难度如同口算32位数MD5一般，除非对应的模块对其进行了引用，不然没有任何的人或者工具可以快速的将它们给猜测出来(物理入侵和存在目录浏览漏洞除外)。至此JS文件提取模块全部结束，在经过三个大功能之后，针对Webpack打包器的JS文件的提取成功率可达到95%以上。

### 0x03 API及其参数提取

#### 0x031 平台API提取

提取JS文件显然不是不是Packer Fuzzer工具的重点，我们的目标是通过JS来寻找可以被利用的API。那么紧着着要做的便是对API的提取，在提取API之前我们一起来了解一下哪些API是符合一个正常API格式的（加粗示例为”合理”API路径）：

| 示例 | 说明 |
| --- | --- |
| **/user/login** | 这是一个API |
| **api/backdoor** | 这也是一个API |
| / | 这个明显不是 |
| a | 哪有这么短的 |
| /video/demo.mp4 | 后缀直接pass |
| /_&§:./+= | 你是何方妖孽？ |
| **/bushishell.do** | 谁还不是个API了 |
| **/v1/test?dev=1** | 把”?”去了也是API |
| /安恒雷神众测 | 没人想不开用中文 |

通过上表我们不难得出一个过滤非API的方式：首先API不能为空或者单纯“/”，其次API内不能存在有类似“*、§、&、=、+”等的怪异字符并且在正常情况下不会含有中文，另外他将不是以“mp4、mp3、ppt”等扩展名做结尾的，若提取出内容中有“?”则需把其后面内容一并去除，而且API长度不会过短（一个字符），排出了这些之后剩下的均是符合情理的。

那么如何提取API，作者想到的第一种方式便是使用正则提取，根据Webpack的特征，我们目前在工具内内置了5条最为常见的API特征规则（位于`/lib/ApiCollect.py`文件内）：

```


1.  `self.regxs =  [r'\w\.get\(\"(.*?)\"\,',`
2.   `r'\w\.post\(\"(.*?)\"\,',`
3.   `r'\w\.post\(\"(.*?)\"',`
4.   `r'\w\.get\(\"(.*?)\"',`
5.   `r'\w\+\"(.*?)\"\,']`


```

首先工具会循环读取每一个被下载至本地的js文件并调用`self.apiCollect()`函数进行规则解析，函数会读取传入文件的内容并在第36行处使用正则匹配对应内容，接着在第38行处使用if语句来判断API是否为空或者为纯“/”内容：

![](https://data.hackinn.com/photo/PF-A/image-20201013223657933.png)

接着我们可以看到在第39行处开始有一个for语句循环，判断“apiExts”数组内的字符是否存在于我们所提取出的“API路径之中”（`if apiExt not in apiPath`），那么这个“apiExt”变量从何而来呢？我们可以看到在第29行处通过读取`config.ini`文件内的内容来实现：

```


1.  `self.apiExts = readConfig.ReadConfig().getValue('blacklist',  'apiExts')[0]`


```

在`config.ini`文件内，我们可以看到对应的配置如下，`apiExts`内为一些不会在正常API中出现的特殊字符以及一些可直接排除的特殊扩展名：

```


1.  `[blacklist]`
2.  `apiExts =  *,+,=,{,},[,],(,),<,>,@,#,",',@,:,?,!, ,^,\,.docx,.xlsx,.jpeg,.jpg,.bmp,.png,.svg,.vue,.js,.doc,.ppt,.pptx,.mp3,.png,.doc,.pptx,.xls,.mp4`


```

若“apiExt”名单的内容在“apiPath”中有所出现则会调用`self.apiTwiceCollect()`再对其内容进行一次正则匹配防止误伤（正则匹配时可能会出现“套娃”情况）：

![](https://data.hackinn.com/photo/PF-A/image-20201014152058140.png)

在工具执行完常规正则提取之后我们再回到主函数`apireCoverStart()`中，此时若通过正则提取出的API路径少于30条则系统将会自动认为没有提取全部而自动开启API暴力提取模式（因为大多数情况下一个有着用户系统的网站的API不会少于30条），若大于或等于30条则将询问用户是否选择开启API暴力提取模式。API暴力提取功能由`apiViolentCollect()`函数实现，其实现原理十分简单也很粗暴：

```


1.  `violentRe = r'(?isu)"([^"]+)'`
2.  `with open(filePath,  "r", encoding="utf-8")  as jsPath:`
3.   `apiStr = jsPath.read()`
4.   `apiLists = re.findall(violentRe, apiStr)`
5.   `for apiPath in apiLists:`
6.   `if apiPath !=  ''  and  '/'  in apiPath and apiPath !=  "/":`
7.   `for apiExt in self.apiExts.split(","):`


```

首先我们使用`r'(?isu)"([^"]+)'`正则语句匹配所有的双引号内容，我们知道一个API路径必然是以字符串的形式保存在JS代码之中，而这些变量是使用双引号保存的。此外我们将会确保“/”符号存在于我们提取的API路径内容内 ，和正常提取规则一样在提取完毕之后排除“apiExt”名单内的无效结果，如此一过滤便得到了相对准确度较高的暴力提取结果（但即使如此，仍会提取出许多无关结果，但还是宁可多跑一点也不要漏下任何一个API）。

在结束了API的提取之后我们还不能获取到完整的API路径，在许多Webpack所构建的网站中使用如下两种形式拼接API地址：1. 域名 + BaseURL + API路径、2. 域名 + API路径 (例如某API完整路径为`https://woshipinyin.cn/v1/api/hello`,那么他的BaseURL可能为`v1`)，故在工具中我们使用`getBaseurl()`函数来提取BaseURL。其提取方式和提取API路径的的方式类似，我们首先使用如下规则的正则表达式来对BaseURL进行提取：

```


1.  `self.baseUrlRegxs =  [r'url.?\s?\:\s?\"(.*?)\"',`
2.   `r'url.?\s?\+\s?\"(.*?)\"',`
3.   `r'url.?\s?\=\s?\"(.*?)\"',`
4.   `r'host\s?\:\s?\"(.*?)\"',  ]`


```

我们并没有在此环境上多花功夫，因为正常的网站一般拥有的BaseURL不会超过3个，并且用户只要在网站中触发任意一个API便可以快速的找出其对应的BaseURL，正因如此本工具支持用户使用`-b（或 --base)`的方式在命令行中自定义多个BaseURL（若启用自定义BaseURL功能则不会执行规则提取BaseURL模块，没有意义）：

```


1.  `baseURL =  CommandLines().cmd().baseurl`
2.  `if baseURL ==  None:`
3.   `xxx...`
4.  `else:`
5.   `baseURLs = baseURL.split(',')`
6.   `self.baseUrlPaths = baseURLs`


```

当然考虑到在许多实际例子中并不存在BaseURL部分，故本工具会默认添加内容为“/”的空BaseURL: `self.baseUrlPaths.append("/")`。 在API路径、BaseURL路径提取完毕之后，我们要做的最后一步便是将域名 + BaseURL路径 + API路径拼接在一起，此功能由`apiComplete()`函数实现并生成一批完整的API路径：

![](https://data.hackinn.com/photo/PF-A/image-20201016140620866.png)

在API路径拼接完成之后程序会对API路径做去重处理，并将API路径的一些信息写入对应缓存数据库的`api_tree`表中方便后期调用，其表说明如下：

| 列名称 | 类型 | 说明 |
| --- | --- | --- |
| id | INT 主键 自增 | 自动排序编号 |
| name | TEXT | API名称 |
| path | TEXT | API的URL路径 |
| option | TEXT | API参数内容 |
| result | TEXT | API返回结果 |
| success | INT | 是否下载成功，成功置1或2 |
| from_js | INT | 被提取的JS文件ID号 |

当然由于存在多个BaseURL并且对于每个BaseURL都会进行一次遍历拼接的操作，并且在使用暴力模式提取的情况下也有许多API路径是无效的，那么如何判断这些API完整路径结果的有效性呢？很简单，全部结果拿去使用多线程跑一遍，在者则在。想法虽然很简单，但并不只是将所有结果GET请求一遍去除其他返回状态码只保留响应为200的结果这么简单，首先我们看如下例子：

![](https://data.hackinn.com/photo/PF-A/image-20201017105124479.png)

在上述例子中，如果我们使用GET请求去访问会返回405报错，因为对应的API只支持POST请求访问（PUT等请求方式不在考虑之内），若我们只简单的保留200结果则将会误杀许多类似只允许使用POST请求但却实际存在的例子，故我们需要对此HTTP响应状态码做一个逻辑判断。相关判断位于`/lib/getApiResponse.py`文件中，兼容401、404、405等HTTP响应状态码，位于第60至72行处：

```


1.  `code = str(s.get(url, headers=headers, timeout=6, proxies=self.proxy_data, verify=False).status_code)  # 正常的返回code是int类型`

3.  `if code !=  "404":`
4.   `self.res[url]  =  1`

6.  `if code ==  "405"  or code ==  "401":`
7.   `self.res[url]  =  2`

9.  `elif code ==  "404":  # 406 请求方法 不一致`
10.   `self.res[url]  =  0`


```

对于GET请求来说无疑是简单的，GET请求的内容固定不变为`?a=1&b=1`的简朴格式，但对于POST请求来说却不一样，例如在上述例子中若我们使用常规请求内容格式及常规请求头`application/x-www-form-urlencoded`去访问则会返回415报错：

![](https://data.hackinn.com/photo/PF-A/image-20201017105242437.png)

在上述例子中，API只支持以JSON的格式传入的数据及使用`application/json`请求头的请求内容其他情况一律报错，但此时API确是存在的并且JSON格式的POST内容在API服务器中非常常见，也是不能忽略的，对此我们需要对这类特殊情况进行一个兼容。首先我们可以看到如下表中三种常规的POST请求内容格式以及其对应的`Content-Type`：

| 请求 | Content-Type | 说明 |
| --- | --- | --- |
| `a=1` | application/x-www-form-urlencoded | 最常见的普通请求内容 |
| `{"a": 1}` | application/json | 使用JSON格式请求 |
| `<a>1</a>` | application/xml | 使用XML格式请求 |

`Content-Type`判断功能位于`/lib/PostApiText.py`中的 `check()`函数内，会依次遵照上表内的格式执行请求，若以上三种请求均返回415状态码则自动放弃该API。在此功能全部运行完毕之后，本工具会将更新当前项目缓存数据库的`api_tree`表中对应API的内容值：1. 若为GET请求`success`值为1，若为POST请求`success`值为2；2. 将成功请求的API响应内容写入`result`中。

#### 0x032 参数模糊提取

注：此为高级模式功能，在默认的简单模式下并不会对API参数进行提取，因为此项功能可能会比较消耗时间。

贪婪是无止境的，在获取完API之后自然而然的想要进一步获取其参数内容，而基于前端打包工具的特性想要实现参数的提取也并非纸上谈兵。首先由于Webpack等打包器所生成的JS文件都是高度压缩的，上万行命令可能被压缩在了数行之内，所以第一步我们需要对所有本地JS文件进行一个十分基础的美化（分行）便于后期能够更加准确的进行参数模糊提取，这里通过调用`/lib/common/beautyJS.py`中的`rewrite_js()`函数实现：

![](https://data.hackinn.com/photo/PF-A/image-20201017135641231.png)

在进行了JS美化之后我们便可以开始正式提取工作，API的参数分为两种：第一类为GET请求的API参数，第二类便是POST请求时所使用的参数。首先我们来到`/lib/FuzzParam.py`文件中看到`collect_api_str()`函数：第一阶段此函数会从当前项目数据库中提取存在的API名称并在所有JS文件中循环寻找此API名称，若找到匹配内容则自动提取上下五行JS脚本内容便于之后提取缩小范围；第二阶段此函数会判断这十一行之内是否存在“GET”、“POST”字符串，若存在则将会把当前内容归为对应的请求方式参数待提取文本，若这两个字符串关键字均不存在则默认归为“POST”（考虑到API特性及POST请求频率）一类。在处理完成之后便进入到`FuzzerCollect()`函数中开始正式处理API参数内容。

和API提取相类似，本工具首先会使用规则进行正则提取，目前版本内置一个提取规则位于`result_method_1()`函数中，其核心正则如下：

```


1.  `regxs_1 = r'method\:.*?\,url\:.*?\,data\:({.*?})'`
2.  `regx_key = r'(.*?)\:.*?\,|(.*?)\:.*?\}'`
3.  `regx_value = r'\:(.*?)\,|\:(.*?)\}'`


```

随后若无匹配规则则自动使用暴力提取模式，位于`violent_method()`函数内，其核心正则如下：

```


1.  `violent_regx = r'(?isu)"([^"]+)'`


```

在提取完成之后本函数还会通过读取`config.ini`文件内相关配置内容来进行过滤判断操作：

```


1.  `[FuzzerParam]`
2.  `param = success,post,get`
3.  `default  = id,num,number,code,type`


```

`param`配置项中所代表的为参数黑名单内容，便于排除一些由暴力模式所提取出的但绝非API参数的项目；`default`配置项中所代表的为INT数字类型参数，若对应项目存在于此名单内，它将会被标记为数字型参数。随后程序会自动对每个数字型参数随机生成一个三位数纯数字的参数内容，对于默认存在参数内容的参数则保留先前的默认参数内容不做修改，对于剩下的参数则自动视为字符型参数并随机生成三位数字符作为参数内容填充。

我们对API参数提取结果的保存方式示例入下：

```


1.  `{`
2.   `"get":  [`
3.   `{`
4.   `"name":  "id",`
5.   `"default":  152`
6.   `},`
7.   `{`
8.   `"name":  "userPass",`
9.   `"default":  "lQG"`
10.   `}`
11.   `],`
12.   `"post":  [`
13.   `{`
14.   `"name":  "deviceType",`
15.   `"default":  "PC"`
16.   `}`
17.   `],`
18.   `"type":  "post"`
19.  `}`


```

可以看到`get`和`post`栏目内分别存入了对应请求类型的请求参数`name`和请求参数的默认（随机）参数内容`default`，`type`一栏则代表当前API的请求类型，例如上述例子内`type`一栏值为`post`则代表此API的请求方式为POST请求。随后程序会将以上JSON内容写入当前项目缓存数据库的`api_tree`表内对应API的`option`一栏中便于后面漏洞检测流程中调用。

#### 0x033 目前的不足

至此API及API参数提取模块也全部结束，本项目最核心的内容便告一段落。但可以看到项目内目前还存在许多不足之处，笔者在此对其一一列出，另若读者朋友们对此有高见及其他想法还愿您提出，感谢：

1.  API提取规则不足，目前仅有3个成熟规则；
2.  API参数提取规则不足，目前仅有1个成熟规则；
3.  API参数提取需要逐个进行正则匹配，若遇上较大JS文件或API结果较多时则会陷入假卡死状态消耗较长时间；
4.  API参数的POST和GET类型判断模块存在一些逻辑问题可能会导致结果不准确；
5.  API参数判断规则及API判断规则库目前还不够丰富；
6.  API参数默认参数内容可能无法较好的提取；
7.  若API与API参数内容段分的很开，或者混淆异常严重则无法成功提取API参数内容；
8.  BaseURL提取规则不足，推荐使用自定义模式…

### 0x04 开始漏洞检测

本工具会将所有漏洞信息写入对应缓存数据库的`vuln`表中方便后期调用，其表说明如下：

| 列名称 | 类型 | 说明 |
| --- | --- | --- |
| id | INT 主键 自增 | 自动排序编号 |
| api_id | INT | 对应API的ID编号 |
| js_id | INT | 对应JS的ID编号 |
| type | TEXT | 漏洞类型 |
| sure | INT | 漏洞置信度 |
| request_b | TEXT | 请求数据内容 |
| response_b | TEXT | 响应数据内容 |
| response_h | TEXT | 响应头内容 |
| des | TEXT | 漏洞附加描述 |

#### 0x041 未授权访问漏洞

什么是API的未授权访问，笔者个人理解他与某个系统、某个数据库未授权访问不同，它不是一套的而是一个个的，可能在实际测试的时候发现这个API有未授权漏洞那个却没有。另外在没有登录Web系统的状态下，一个可被访问的公共（公开）API也算是未授权访问API，只不过是没有敏感内容不能进行敏感操作罢了。下表为常见的API请求返回内容类型及笔者注释：

| API返回内容 | 说明 |
| --- | --- |
| {“msg”:”hello”,”code”:”200”} | 可未授权访问的API，只是不能进行敏感操作或无敏感内容 |
| {“msg”:””,”errcode”:”1”,”log”:”您还没有登录！”} | 不存在未授权访问漏洞 |
| {“status”:”200”,”message”:”success”,”data”:{“id”:”1”,”user”:”admin”,”pass”:”1143720b05b5daf6f2bb83f9e4a9b5ba”}} | 存在未授权访问漏洞，还需结合参数内容进行进一步测试 |

并且由于在“平台API提取”一步中本工具已经将所有存在的API的返回内容写入了数据库内，故我们只需“人工智障”的判断对应数据库内的返回内容便可以简单判断是否存在未授权访问漏洞。只要名单足够多足够丰富，准确率便会很高，以下是现有规则名单：

```


1.  `[vulnTest]`
2.  `resultFilter =  未登录,请登录,权限鉴定失败,未授权,鉴权失败,unauth,状态失效,没有登录,会话超时,token???,login_failure`
3.  `unauth_not_sure =  系统繁忙,系统错误,系统异常,服务器繁忙,参数错误,异常错误,服务端发生异常,服务端异常`


```

可以看到我们将名单的关键词通过语义情感分为两类，一类是`resultFilter`中明确表示需要登录或是未授权访问的，此类API必然不存在未授权访问漏洞；第二类是`unauth_not_sure`中带有不确定因素语气的词汇，这类词汇并没有直接说明不让你访问也没有返回正常返回的内容，可能是由于缺少API参数及参数内容造成服务端处理异常，故我们可以将存在此类关键词的API归为“疑似存在未授权访问漏洞”。

#### 0x042 敏感信息泄露漏洞

这里的敏感信息泄露并非指服务器某个文件泄露或者某个API泄露敏感信息，而是针对我们所解析的当前网站的所有JS文件进行排查，寻找是否有开发残留的测试用或用作后门的密码、Token等敏感信息。

首先本工具会读取`config.ini`文件内已经预设的相关敏感变量名称配置信息内容：

```


1.  `[infoTest]`
2.  `info = REDIS_PM§§§Redis  Password,APP_KEY§§§Third APP Key,password§§§Password  Info,BEGIN RSA PRIVATE KEY§§§RSA PRIVATE KEY,email§§§email address`


```

所有的变量配置均用逗号分隔，其中`§§§`后接的为对应敏感变量的描述信息，例如：REDIS_PM§§§Redis Password则代表敏感变量名称为`REDIS_PASS`,若检测到有泄露则泄露的内容为Redis数据库访问密码。本模块实现原理非常简单，会循环判断每个JS文件并判断其中是否存在对应的变量名称，若存在则提取变量的赋值内容（双引号内内容），并对泄露位置的上下文的77个字符也一并提取随后存入数据库中。

#### 0x043 CORS漏洞

本模块将会也只会对目标主站进行一次请求测试，并在请求头部中添加`Origin`头，其头部内容值的生成方式如下：

```


1.  `self.expUrl =  "https://"  + self.baseurl.netloc +  ".example.org"  +  "/"  + self.baseurl.netloc`


```

若目标站点为`lei-god666.com`，则对应的检测EXP为：`Origin: https://lei-god666.com.example.org/lei-god666.com`。这样我们便可以使用一个Payload便绕过一些服务器对于CORS漏洞的常见的几种过滤、判断机制。在请求成功之后，本模块将会对返回包的返回头做如下检测：

```


1.  `text = requests.get(self.url, headers=self.header, timeout=6, allow_redirects=False).headers`
2.  `if  'example.org'  in text['Access-Control-Allow-Origin']  and text['Access-Control-Allow-Credentials']  ==  'true':`
3.   `#print("已检测到cors漏洞")`
4.   `self.flag =  1`


```

若返回头部中不存在`Access-Control-Allow-Credentials`一栏并且其值不为`true`或是`Access-Control-Allow-Origin`的值中不存在我们的预设网站`example.org`则程序将会自动判断当前目标站点不存在CORS漏洞。若返回头内容同时满足以上两点规则，则表明存在CORS漏洞，之后本模块会将返回头的头部内容写入缓存数据库中。

#### 0x044 SQL注入漏洞

针对SQL注入漏洞，本模块支持并会依次进行以下三种快速检测方式（不会对SQL漏洞进行深入测试，比如进行提取数据等操作，仅做基础判断，若想进一步检测请使用SQLMAP、XARY等检测工具）：

*   报错注入
    
    此功能位于`/lib/vuln/SqlTest.py`文件`errorSQLInjection()`函数中，首先我们会解析“API参数模糊提取”功能所生成的参数及参数内容，并在请求参数中追加单引号及双引号字符进行请求，促使后端数据库返回经典报错内容：
    
    ![](https://data.hackinn.com/photo/PF-A/image-20201018222910310.png)
    
    之后本函数会对API请求返回内容做检测，若存在如下预设内容的其中一个则代表存在SQL报错注入：
    
    ```
    
    
    1.  `errors =  ["You have an error in your SQL syntax","Oracle Text error","Microsoft SQL Server"]`
    2.  `for error in errors:`
    3.   `if error in get_resp_text:`
    4.   `#print("目标疑似存在SQL报错注入")`
    5.   `self.error =  1`
    
    
    ```
    
*   布尔型盲注
    
    此功能位于`/lib/vuln/SqlTest.py`文件`boolenSQLInjection()`函数中，首先我们会解析“API参数模糊提取”功能所生成的参数及参数内容，并在请求参数中添加`and 1=1`或`and 1=2`字符串并请求，促使后端数据库返回内容长度不同的返回包数据：
    
    ![](https://data.hackinn.com/photo/PF-A/image-20201018223152112.png)
    
    之后本函数会通过如下命令判断返回内容长度是否一致，若不一致则表明存在SQL布尔型盲注：
    
    ```
    
    
    1.  `post_len1 = len(requests.post(self.path,headers=self.header,data=post_data1,proxies=self.proxy_data).text)`
    2.  `post_len2 = len(requests.post(self.path,headers=self.header,data=post_data2,proxies=self.proxy_data).text)`
    3.  `post_len_default = len(requests.post(self.path,headers=self.header,data=post_data_default,proxies=self.proxy_data).text)`
    4.   `if  (post_len1 == post_len_default)  and  (post_len2 != post_len_default):`
    5.   `#print("疑似存在布尔盲注")`
    6.   `self.boolen =  1`
    
    
    ```
    
*   基于时间延迟注入
    
    此功能位于`/lib/vuln/SqlTest.py`文件`timeSQLInjction()`函数中，首先我们会解析“API参数模糊提取”功能所生成的参数及参数内容，并在请求参数中添加`and sleep(10)`字符并请求，促使后端服务器对在处理返回包时有所延迟：
    
    ![](https://data.hackinn.com/photo/PF-A/image-20201018223551045.png)
    
    之后本函数会通过判断服务器响应时间来判断是否存在基于时间延迟的SQL注入：
    
    ```
    
    
    1.  `if json_code ==  '415':`
    2.   `pass`
    3.  `elif default_time<2  and json_sec>9:`
    4.   `#print("疑似存在sql时间盲注")`
    5.   `self.time =  1`
    6.   `try:`
    7.   `DatabaseType(self.projectTag).insertSQLInfoIntoDB(self.api_id, self.from_js, post_json, post_json_resp.text)`
    8.   `except  Exception  as e:`
    9.   `self.log.error("[Err] %s"  % e)`
    
    
    ```
    

#### 0x045 水平越权漏洞

水平越权模块位于`/lib/vuln/BacTest.py`文件中，本功能只对数字类型参数进行检测，若无符合API将不会做后续测试。其简便检测逻辑如下：首先函数会生成五个纯数字，并将其拼接为HTTP请求内容:

```


1.  `for value in range(1,6):`
2.   `get_req = str("".join(name))  +  "="  + str(value)  # 进行拼接`
3.   `get_burp.append(get_req)  # 遍历后的参数`


```

之后会用所产生的五个请求内容去进行HTTP请求获取请求的请求返回内容长度：

```


1.  `get_resp_lens =  {}`
2.  `try:`
3.   `get_obj =  ApiText(self.get_results,self.options)`
4.   `get_obj.run()`
5.   `get_texts = get_obj.res`
6.  `except  Exception  as e:`
7.   `self.log.error("[Err] %s"  % e)`


```

随后本功能将会对5个返回包内容长度进行检测，5个请求中存在若3个及以上的返回包内容长度不一致，则认定此API存在水平越权漏洞：

```


1.  `get_repeat_nums = dict(Counter(get_all_list))`
2.  `get_repeat_num = int("".join([str(key)  for key, value in get_repeat_nums.items()  if value >  1]))  # 获取到我们重复的数据`
3.   `if len(get_select_list)  >=3:`
4.   `...`


```

虽然说可能会导致误报，但使用此方式检测起来非常的快速并且基本不会错杀，之后再对其进行进一步人工检测即可。

#### 0x046 弱口令漏洞

首先本检测模块并不会对所有API都进行密码爆破测试，工具会首先读取在`config.ini`内的`passwordtest_list`、`passworduser_list`、`passwordpass_list`三个配置，其对应预设配置内容如下：

```


1.  `[vuln]`
2.  `passwordtest_list = login.do,signin,login,user,admin`
3.  `passworduser_list = userCode,username,name,user,nickname`
4.  `passwordpass_list = userPass,password,pass,code`


```

随后会调用`passwordTest()`函数做如下三重检测：1\. API名称是否存在于`passwordtest_list`名单内，2\. 对应API中是否存在`passworduser_list`名单内的任意一个参数名，3\. 对应API中是否存在`passwordpass_list`名单内的任意一个参数名。若三个条件同时满足，则会调用`startTest`函数并将参数信息等内容传入，执行弱口令爆破功能：

```


1.  `if  (pass_list[3]  !=  "none")  and  (pass_list[2]  !=  "none"):`
2.   `self.startTest(pass_list[0],pass_list[1],pass_list[2],pass_list[3])`


```

其中用户名爆破字典及密码爆破字典分别储存在本工具`/doc/dict/username.dic`和`/doc/dict/password.dic`文件之内，用户可以随意添加、修改及替换。在多线程请求爆破结束之后，程序会判断返回包内容中是否存在以下`login`参数中的任意一个配置值，若存在则表明成功爆破：

```


1.  `[vulnTest]`
2.  `login =  登录成功,login success,密码正确,成功登录`


```

#### 0x047 任意文件上传漏洞

本块位于`/lib/vuln/UploadTest.py`文件中，在第18-20行内会首先读取`config.ini`配置文件内如下预设配置：

```


1.  `[vuln]`
2.  `uploadtest_list = upload,file,doc,pic,update`
3.  `upload_fail =  上传失败,不允许,不合法,非法,禁止,fail,失败,错误`
4.  `upload_success =  上传成功,php,asp,html,jsp,success,200,成功,已上传`


```

其中`uploadtest_list`配置用于筛选符合检测条件的API名称，`upload_fail`为上传返回内容黑名单，`upload_success`为上传返回内容白名单。在`startTest()`函数内存在如下名为`ext_fuzz`的变量，预设了用于模糊测试用的各类文件扩展名：

![](https://data.hackinn.com/photo/PF-A/image-20201020091404894.png)

随后函数会执行如下拼接操作：1\. 文件名使用“随机纯数字” + “.” + “特定扩展名”，2. 文件内容使用“PNG图片文件头” + 随机内容：

```


1.  `files =  {"file":  (`
2.   `"{}.{}".format(random.randint(1,100), ext),  (b"\x89\x50\x4E\x47\x0D\x0A\x1A\x0A\x00\x00\x00\x0D\x49\x48\xD7"+rands))}`


```

之后本程序会将所生成的所有文件内容进行一次请求操作，并记录其返回包内容。在获取返回包之后本模块采用“黑 \+ 白”模式，首先对内容进行一遍黑名单过滤，随后在剩余内容中通过白名单提取有效内容：

![](https://data.hackinn.com/photo/PF-A/image-20201020095728841.png)

若通过白名单提取，则本工具会将其视作存在任意文件上传漏洞，并写入数据库中。至此漏洞检测全部结束，其中SQL注入漏洞、水平越权漏洞、弱口令漏洞及任意文件上传漏洞只有在高级版模式下才会启用（因为要提取API参数内容）。

### 0x05 报告生成与其他

#### 0x051 报告，很清爽

作为一款扫描工具，报告生成部分自然是不可缺少的，本程序支持生成交互式报告、正式报告两类报告，四种格式：HTML、DOC、PDF、TXT，在默认情况下将会生成HTML报告及DOC报告。

首先我们先来看如下示例HTML报告，本HTML报告采用“ModSF主题”（移动安全框架（MobSF）是一种自动化的多合一移动应用程序（Android / iOS / Windows）可以进行静态和动态分析的笔测试，恶意软件分析和安全评估框架。）。本报告分为基础信息、漏洞详情、API清单、安全建议、附录五大部分：1. 在基础信息部分中存在报告摘要、JS信息、免责声明三个小部分；2. 在漏洞详情及安全建议中，其子目录将根据所存在的漏洞自动生成：

![](https://data.hackinn.com/photo/PF-A/image-20201020204631883.png)

在每个漏洞及API清单处均有一个“More”按钮，单击之后便可以看到对应漏洞/API的详细信息：

![](https://data.hackinn.com/photo/PF-A/image-20201020204712964.png)

注意：若想将HTML报告移动，则需一并复制对应目录下的`res`资源文件夹。

随后我们可以看到DOC报告如下（PDF报告、TXT报告基于DOC报告生成，可能需要装Office Word控件，WPS等其他控件可能会造成生成样式混乱），可以看到相较于HTML更为正式，但是缺少了便捷性：

![](https://data.hackinn.com/photo/PF-A/image-20201020204251724.png)

与HTML报告类似，DOC报告存在：报告摘要、漏洞详情、API清单、安全建议、附录五个大环节，具体细节根据漏洞详情等自动生成。

#### 0x052 语言，全球化

您完全不用担心国际化所带来的问题，本工具内置五大主流语言语言包（包括报告模板）：简体中文、法语、西班牙语、英语、日语（根据翻译准确度排序），由我们非常~不~专业的团队翻译。语言文件位于`/doc/lang.ini`内，为标准的INI文件：

![](https://data.hackinn.com/photo/PF-A/image-20201022133247534.png)

其语言读取原理实现如下，工具会先获取用户本机的`locale`信息并取其前两位字符至语言配置文件内寻找，若寻找失败（其他不支持的语言）自动启用英文，并且支持用户通过命令行指定全局语种：

![](https://data.hackinn.com/photo/PF-A/image-20201020205448454.png)

在`locale`内我们可以看到他的值构成为：小写两位语言码 \+ （“_” + 大写两位国家/地区码） + （“@” + 小写国家/地区英文名） + （“.” + 当前编码格式），例如在笔者电脑环境中其默认语言就为法语（fr）：

![](https://data.hackinn.com/photo/PF-A/image-20201020205642246.png)

工具可使用如下命令格式对不同语言进行自动调用输出：`print(Utils().getMyWord("{xxx_name_nom}"))`

除日语外，中文、法语、英语、西班牙语均为联合国官方语言，若您也对其他语言的翻译感兴趣或者认为当前任意翻译的某个词、句存在翻译不准确、翻译错误，欢迎您将您的翻译内容提交给我们！

#### 0x053 扩展，不局限

本工具支持添加扩展插件，只需将所有插件依照格式放入程序`ext/`目录下即可，本工具将会在漏洞扫描结束之后、报告生成之前自动执行所有开启的符合标准的扩展插件。其插件运行模块位于`/lib/LoadExtensions.py`文件内（如下图），其原理如下：首先本模块将会读取所有扩展目录下扩展名为`.py`的文件并且导入每个文件内的所有模块，随后执行对应模块内`ext`类的`start()`函数。

![](https://data.hackinn.com/photo/PF-A/image-20201022134844717.png)

其DEMO示例如下，首先会导入`Utils`和`DatabaseType`库，并由`self.statut`变量值决定当前插件开启状态（1为开启，0或其他为关闭）；若当前插件处于开启状态将会自动调用`run()`函数（用户自定义插件内容段）：

```


1.  `#!/usr/bin/env python3`
2.  `# -*- encoding: utf-8 -*-`

4.  `from lib.common.utils import  Utils`
5.  `from lib.Database  import  DatabaseType`

8.  `class ext():`

10.   `def __init__(self, projectTag, options):`
11.   `self.projectTag = projectTag`
12.   `self.options = options`
13.   `self.statut =  0  #0 disable  1 enable`

15.   `def start(self):`
16.   `if self.statut ==  1:`
17.   `self.run()`

19.   `def run(self):`
20.   `print(Utils().tellTime()  +  "Hello Bonjour Hola 你好 こんにちは")`


```

#### 0x054 日志，我们有

本工具在执行每次扫描任务时不论任务成功或者失败均会自动实时生成以当前项目TAG为名的日志文件保存于`log/`目录下：

![](https://data.hackinn.com/photo/PF-A/image-20201020215502622.png)

其模块内容位于`/lib/`目录内，其他模块可使用`from lib.common.CreatLog import creatLog`引入此模块，并按照如下方式调用即可。本模块将会记录三种类型日志：信息日志（所有的屏幕输出）、调试及提示信息日志、程序运行报错日志，除这三类之外本工具不会收集任何额外的隐私信息亦不会将用户的任何信息上传至服务器：

```


1.  `creatLog().get_logger().info("Start!")`
2.  `try:`
3.   `xxx()`
4.   `creatLog().get_logger().debug("OK!")`
5.  `except  Exception  as e:`
6.   `creatLog().get_logger().error("[Err] %s"  % e)`


```

我们可以看到生成的示例日志格式如下：

![](https://data.hackinn.com/photo/PF-A/image-20201020222037731.png)

若您遇到错误需要提交issue时，为了方便我们快速进行错误定位，还请您将您对应项目的扫描日志一并发送给我们。（若您认为目标敏感不宜直接公开，您可以将对应log文件通过电子邮件附件的形式共享给我们。）

### 0x06 结语&谢言

第一版全部的思路及框架到这里便结束了，但此刻的终点亦是第二版的全新起点。笔者深知自己的思路有许多不足及缺陷，希望能多听听读者朋友的意见及建议，多看看那一个个大家贡献的ISSUE。Poc Sir再次感谢各位能花上一些时间来阅读拙作，若您认为我们的思路能对您今后的工作有所帮助还望您再稍许浪费一些时间给我们的Github仓库点下一个“Star”，您的“Star”将会是一个巨大的肯定！此外，我还要感谢我的一位挚友，感谢您对我的支持及鼓励！

我抬头望了望窗外色彩斑斓的秋，稍许有了些熟悉的感觉，抗疫也还远远不能结束啊。

![](https://data.hackinn.com/photo/PF-A/image-20201008193221710.png)

\[关于本文作者\]: Poc-Sir (谢邀，人在法国，刚上飞机) 非著名Ctrl C/V工程师、不专业漏洞复现研究员、雷神众测实习扫地白帽子、各大SRC低危贡献选手、资深CFT倒茶专家  
« [https://www.hackinn.com](https://www.hackinn.com/) 收集整理国内外安全会议资料 »