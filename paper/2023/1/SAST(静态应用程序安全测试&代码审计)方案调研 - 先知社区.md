# SAST(静态应用程序安全测试&代码审计)方案调研 - 先知社区
0x00 写在前面
---------

最近一直在从事SAST相关的工作。在之前SAST方案调研过程中，选取了几款主流的开源项目/商业产品进行本地部署搭建测试，本文是对之前调研过程中的一些笔记进行的梳理和总结。同时，调研过程中也阅读到了非常多优秀的文章，在本文中也会进行引用，这些都是SAST宝贵的学习资料。

读者可根据本文目录，自行选择感兴趣的部分阅读。

0x01 SAST基础
-----------

### 1.1 SAST简介

**SAST(Static Application Security Testing)**：静态应用程序安全测试技术，通常在编码阶段分析应用程序的源代码或二进制文件的语法、结构、过程、接口等来发现程序代码存在的安全漏洞。

目前有多款静态代码检测的开源项目，通过对几款项目的调研，可总结出目前优秀的静态代码检测工具的基本流程为：

对于一些特征较为明显的可以使用正则规则直接进行匹配，比如硬编码密码、错误的配置等，但正则的效率是个大问题。

对于 OWASP Top 10 的漏洞，通过预先梳理能造成危害的函数，并定位代码中所有出现该危害函数的地方，继而基于**Lex(Lexical Analyzer Generator, 词法分析生成器)**和**Yacc(Yet Another Compiler-Compiler, 编译器代码生成器)**将对应源代码解析为**AST(Abstract Syntax Tree, 抽象语法树)**，分析危害函数的入参是否可控来判断是否存在漏洞。通过操作类的字节码返回解释器执行。

大致流程如下图所示（该流程也是目前主流静态代码检测产品的大致流程）：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905132640-d96c2dfa-0e09-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905132640-d96c2dfa-0e09-1.png)

### 1.2 SAST技术发展阶段

当然静态检测所对应的技术也是经过不断发展。这边推荐阅读**@LoRexxar**师傅写的[《从0开始聊聊自动化静态代码审计工具》](https://lorexxar.cn/2020/09/21/whiteboxaudit/#%E8%87%AA%E5%8A%A8%E5%8C%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1)一文，文中详细介绍了静态代码检测技术的几个发展阶段：

| 技术发展阶段 | 代表工具 | 优点 | 缺点 | 备注 |
| --- | --- | --- | --- | --- |
| 基于正则-关键字匹配的方案 | **Seay**：高覆盖，通过简单的关键字匹配更多的目标，后续通过人工审计进行确认；-**Rips免费版**：高可用性，通过更多的正则约束、更多的规则覆盖多种情况。 | 方案简单，实现起来难度不大 | 1\. 无法保证开发人员的开发习惯及代码编写，误报率及漏报率非常高；2. 无法保证高覆盖及高可用性； 3. 维护成本太大。 |  |
| 基于AST(抽象语法树)的方案 | **Cobra**：侧重甲方的静态自动化代码扫描，侧重低漏报率； **Kunlun-M**：侧重于白帽子自用，只有准确的流才会认可，否则标记为疑似漏洞，侧重低误报率。 | 在编译处分析代码，无需关注开发人员的开发习惯（编译器是相同）。 | 1\. 无法保证完美处理所有的AST结构； 2. 基于单向流的分析方式，无法覆盖100%的场景；3. 忽略了大多数的分支、跳转、循环这类影响执行过程顺序的条件，会造成 | 常见的语义分析库：[PHP-Parser](https://github.com/nikic/PHP-Parser)、[phply](https://github.com/viraptor/phply)、[javalang](https://github.com/c2nes/javalang)、[javaparser](https://github.com/javaparser/javaparser)、[pyjsparser](https://github.com/PiotrDabkowski/pyjsparser)、[spoon](https://github.com/INRIA/spoon) |
| 基于IR/CFG这类统一数据结构的方案 | fortify、Checkmarx、Coverity及最新版的Rips：都使用了自己构造的语言的某一个中间部分。 | 较于AST，该方法带有控制流，只需要专注于从Source到Sink的过程即可。 |  |  |
| 基于QL的方案 | CodeQL：可以将自动化代码分析简化约束为我们需要用怎么样的规则来找到满足某个漏洞的特征。 | 把流的每一个环节具象化，把每个节点的操作具像成状态的变化，并且储存到数据库中。这样一来，通过构造QL语言，我们就能找到满足条件的节点，并构造成流。 |  |

0x02 静态代码检测（代码审计）工具调研
---------------------

本次调研测试，对四个开源项目（两个Java，两个PHP）进行扫描，分别为[DVWA](https://github.com/digininja/DVWA)、[Mutillidae](https://github.com/webpwnized/mutillidae)、[java-sec-code](https://github.com/JoyChou93/java-sec-code)、[OWASP Benchmark](https://github.com/OWASP/benchmark)。其中OWASP Benchmark是OWASP组织下的一个开源项目，又叫作OWASP基准测试项目，它是免费且开放的测试套件。它可以用来评估那些自动化安全扫描工具的速度、覆盖范围和准确性，这样就可以得到这些软件的优点和缺点。OWASP Benchmark计分器的使用可以参考下文 0x03附录-3.1 OWASP Benchmark计分器使用。

### 2.1 Cobra

#### 2.1.1 介绍

一款支持检测多种开发语言中多种漏洞类型的源码审计工具：支持十多种开发语言及文件类型扫描，支持的漏洞类型有数十种。扫描方式支持命令行（通过命令行扫描本地源代码）、界面、API（供内部人员通过GUI的形式访问使用，并可以通过API集成到CI或发布系统中）三种。

对于一些特征较为明显的可以使用正则规则来直接进行匹配出，比如硬编码密码、错误的配置等。Cobra通过预先梳理能造成危害的函数，并定位代码中所有出现该危害函数的地方，继而基于Lex(Lexical Analyzer Generator, 词法分析生成器)和Yacc(Yet Another Compiler-Compiler, 编译器代码生成器)将对应源代码解析为AST(Abstract Syntax Tree, 抽象语法树)，分析危害函数的入参是否可控来判断是否存在漏洞（目前仅接入了PHP-AST，其它语言AST接入中）。

#### 2.1.2 安装部署

\# -安装过程基于docker镜像手动安装-

\# 1\. 拉取docker镜像
docker pull centos:centos7
docker run -itd --name code\_audit\_test centos:7
docker exec -it 容器ID bash

\# 2\. docker内centos容器安装cobra
\## 2.1 升级python2.x版本为3.x
cd tmp/
wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tgz
tar -zxf Python-3.7.3.tgz
yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc  libffi-devel
yum install -y libffi-devel
cd Python-3.7.3
./configure --prefix=/usr/local/python3.7
make && make install
\## 2.2 备份先前系统默认的python2，创建新的软链接
mv /usr/bin/python /usr/bin/python.bak
ln -s /usr/local/python/bin/python3.7 /usr/bin/python
\## 2.3 更改yum配置
vim /usr/bin/yum   #将#!/usr/bin/python改为#!/usr/bin/python2
vim /usr/libexec/urlgrabber-ext-down   #将#!/usr/bin/python改为#!/usr/bin/python2

\# 3\. Cobra安装
yum install flex bison phantomjs
git clone https://github.com/WhaleShark-Team/cobra.git && cd cobra
python -m pip install -r requirements.txt
python cobra.py --help

安装完成后，运行截图如下所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905140050-9f305df0-0e0e-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905140050-9f305df0-0e0e-1.png)

#### 2.1.3 扫描测试

Cobra运行扫描命令如下：

python cobra.py -t /tmp/source\_code\_scan/DVWA-master -f json -o /tmp/report.json

扫描完成截图如下所示，扫描出的漏洞以（C-严重、H=高危、M-中危、L-低危）进行分类：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905140758-9e158728-0e0f-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905140758-9e158728-0e0f-1.png)

#### 2.1.4 扫描结果对比分析

汇总结果如下（**误报率及漏报率仅根据官方最新版本默认配置，实际测试结果进行主观大致预估**）：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210913161311-6fcd6974-146a-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210913161311-6fcd6974-146a-1.png)

#### 2.1.5 调研结果

本次对Cobra的调研结果如下：

| **调研参数** | **调研结果** | **总结** |
| --- | --- | --- |
| 漏洞检测/扫描的原理 | 正则匹配+AST(语义分析，仅PHP) | **优点：** 1\. 代码逻辑及思路较清晰，便于理解及后续拓展；2. 扫描方式支持多种，特别是支持API调用，便于持续集成。 |
| 是否支持持续集成 | 提供web接口，便于持续集成 | **缺点：** 1\. 项目目前已没在维护，规则更新停留在三年前，特别是针对组件CVE规则；2. 目前只支持PHP的语义分析（AST），其他语言都是通过正则匹配；3. 误报率/漏报率都非常高，Java更甚。 |
| 扫描速度 | 较快 |  |
| 数据输入方式 | 命令行模式通过-t参数指定代码工程；API模式通过请求接口指定git工程； |  |
| 部署方式 | python脚本直接运行 |  |
| 支持语言 | 十多种（包括PHP、Java、Python、Ruby、Go、C++等），详细参考：[http://cobra.feei.cn/languages，语义分析只支持PHP。](http://cobra.feei.cn/languages%EF%BC%8C%E8%AF%AD%E4%B9%89%E5%88%86%E6%9E%90%E5%8F%AA%E6%94%AF%E6%8C%81PHP%E3%80%82) |  |
| 漏洞覆盖 | 支持十多种漏洞类型（SQL注入、XSS、命令注入、代码注入、反序列化、不安全第三方组件等等），详细参考：[http://cobra.feei.cn/labels](http://cobra.feei.cn/labels) |  |

#### 2.1.6 参考资料

*   [Cobra官方文档](http://cobra.feei.cn/)；
*   [静态代码审计工具Cobra/Cobra-W/find-sec-bugs](https://blog.csdn.net/caiqiiqi/article/details/104158061/)；
*   [代码审计工具 Cobra 源码分析（一）](https://zhuanlan.zhihu.com/p/32363880)；
*   [代码审计工具 Cobra 源码分析（二）](https://zhuanlan.zhihu.com/p/32751099)；

### 2.2 KunLun-M(昆仑镜)

#### 2.2.1 介绍

以下内容引用自**@LoRexxar**师傅的[《从0开始聊聊自动化静态代码审计工具》](https://lorexxar.cn/2020/09/21/whiteboxaudit/#%E8%87%AA%E5%8A%A8%E5%8C%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1)一文及 \[KunLun-M官方Github\] ([https://github.com/LoRexxar/Kunlun-M)：](https://github.com/LoRexxar/Kunlun-M)%EF%BC%9A)

在Cobra的基础上，深度重构了AST回溯部分，将工具定位在白帽子自用上，侧重于低误报率，进行二次开发。从之前的Cobra-W到现在的KunLun-M，只有准确的流才会认可，否则将其标记为疑似漏洞，并在多个环节定制了自定义功能及记录了详细的日志。

目前的KunLun-M，作者照着phply的底层逻辑，重构了相应的语法分析逻辑。添加了Tamper的概念用于解决自实现的过滤函数。引入了python3的异步逻辑优化了扫描流程等，同时大量的剔除了正则+AST分析的逻辑，因为这个逻辑违背了流式分析的基础，然后新加了Sqlite作为数据库，添加了Console模式便于使用，同时也公开了有关javascript代码的部分规则。

KunLun-M的AST分析流程图如下所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905140724-89fb0506-0e0f-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905140724-89fb0506-0e0f-1.png)

#### 2.2.2 安装部署

\# -安装过程基于docker镜像手动安装-

\# 1\. 拉取docker镜像
docker pull centos:centos7
docker run -itd --name code\_audit\_test centos:7
docker exec -it 容器ID bash

\# 将python2.7升级到python3.7
\# 系统自带的sqlite3版本过低，需要升级一下sqlite3
wget https://www.sqlite.org/2019/sqlite-autoconf-3270200.tar.gz
tar -zxvf sqlite-autoconf-3270200.tar.gz
./configure --prefix=/usr/local
make && make install
\## 备份原先系统版本
mv /usr/bin/sqlite3 /usr/bin/sqlite3_old
\## 软链接将新的sqlite3设置到/usr/bin目录下
ln -s /usr/local/bin/sqlite3 /usr/bin/sqlite3
\## 将路径传递给共享库
export LD\_LIBRARY\_PATH="/usr/local/lib"
source ~/.bashrc

\## 检查python的SQLite3版本
import sqlite3
sqlite3.sqlite_version

git clone https://github.com/LoRexxar/Kunlun-M

\# 安装依赖
pip install -r requirements.txt
\# 配置文件迁移
cp Kunlun\_M/settings.py.bak Kunlun\_M/settings.py
\# 初始化数据库
python kunlun.py init initialize

#### 2.2.3 扫描测试

使用console模式，相关命令如下：

\# 进入console模式
python kunlun.py console

\## 如何扫描源代码
scan   #进入扫描模式
set target /tmp/source_code/DVWA-master/ #设置目标
run

\## 如何查看扫描结果
showt  #查看扫描任务列表
load 1 #加载扫描任务id
show vuls #查看漏洞结果

扫描完成结果如下图所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905140742-94fe08fe-0e0f-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905140742-94fe08fe-0e0f-1.png)

#### 2.2.4 扫描结果分析

汇总结果如下（**误报率及漏报率仅根据官方最新版本默认配置，实际测试结果进行主观大致预估**）：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210913161328-79e2e47a-146a-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210913161328-79e2e47a-146a-1.png)

#### 2.2.5 调研结果

本次对KunLun-M的调研结果如下：

| **调研参数** | **调研结果** | **总结** |
| --- | --- | --- |
| 漏洞检测/扫描的原理 | 正则匹配+深度优化的AST(语义分析，仅PHP) | **优点：** 1\. 该项目开源并持续维护中；2. 比Cobra更优的AST语义分析；3. 代码逻辑及思路较清晰，便于理解及后续拓展。 |
| 是否支持持续集成 | 未提供web接口，但命令行模式也便于持续集成 | **缺点：**  1\. 支持语言较少，目前主要是PHP和Javascript，不支持Java；2. 误报率/漏报率较高。 |
| 扫描速度 | 速度一般 |  |
| 数据输入方式 | 命令行模式通过`-t`指定代码工程；console模式通过`set target`指定代码工程； |  |
| 部署方式 | python脚本直接运行 |  |
| 支持语言 | 目前主要支持**php、javascript**的语义分析，以及**chrome ext, solidity**的基础扫描 |  |
| 漏洞覆盖 | 支持十多种漏洞类型（SQL注入、XSS、命令注入、代码注入、反序列化等等） |  |

#### 2.2.6 参考资料

*   [Kunlun-M 官方Github](https://github.com/LoRexxar/Kunlun-M)；
*   [构造一个CodeDB来探索全新的白盒静态扫描方案](https://lorexxar.cn/2020/10/30/whitebox-2/)；
*   [从0开始聊聊自动化静态代码审计工具](https://lorexxar.cn/2020/09/21/whiteboxaudit/#%E8%87%AA%E5%8A%A8%E5%8C%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1)；

### 2.3 Hades

#### 2.3.1 介绍

以下内容引用[Hades的官方Github](https://github.com/zsdlove/Hades)：

Hades静态代码脆弱性检测系统是默安开源的针对Java源码的白盒审计系统，该系统使用**smali字节码的虚拟解释执行引擎**对Java源码进行分析。

Hades系统整体的执行流程示意图如下：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905141425-851fae32-0e10-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905141425-851fae32-0e10-1.png)

**smali字节码**：程序的编译一般经过六个流程，即词法分析、语法分析、语义分析、中间代码生成、代码优化、目标代码生成。

*   词法分析：主要是对输入的源码文件中的字符从左到右逐个进行分析，输出与源代码等价的token流；
*   语法分析：主要是基于输入的token流，根据语言的语法规则，做一些上下文无关的语法检查，语法分析结束之后，生成AST语法树；
*   语义分析：主要是将AST语法树作为输入，并基于AST语法树做一些上下文相关的类型检查；
*   生成中间代码：语义分析结束后，生成中间代码，而此时的中间代码，是一种易于转为目标代码的一种中间表示形式；
*   代码优化：针对中间代码进行进一步的优化处理，合并其中的一些冗余代码，生成等价的新的中间表示形式，最后生成目标代码。

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905141447-91e7b966-0e10-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905141447-91e7b966-0e10-1.png)

相较java字节码来说，smali字节码更加的简单，因为smali字节码是一种基于寄存器的指令系统，它的指令是二地址和三地址混合的，指令中指明了操作数的地址。而JVM是基于栈的虚拟机，JVM将本地变量放到一个本地变量列表中，在进行指令解释的时候，将变量push到操作数栈中，由操作码对应的解释函数来进行解释执行。所以，java字节码操作无疑会比smali字节码更复杂一些，复杂主要体现在后续的堆栈设计以及代码解释执行。

想更深入的原理及实现思路可参考官方Github，作者编写了很详细的文档介绍。

#### 2.3.2 安装部署

docker build -t hades .
docker run -p 8088:8088 hades
\# 访问
http://127.0.0.1:8088/geekscanner

#### 2.3.3 扫描测试

目前项目bug较多，本地搭建报错，暂未进行详细分析。

#### 2.3.4 扫描结果分析

目前项目bug较多，本地搭建报错，暂未进行详细分析。

根据官方文档的测试结果，对Java的白盒分析效果还可以。

#### 2.3.5 调研结果

本次对Hades的调研结果如下：

| **调研参数** | **调研结果** | **总结** |
| --- | --- | --- |
| 漏洞检测/扫描的原理 | 基于smail字节码的虚拟解释执行引擎 | **优点：** 1\. 该项目思路及方案很好，值得深入研究。 |
| 是否支持持续集成 | 有web页面，便于持续集成 | **缺点：**  1\. 项目部署报错，无法本地搭建测试；2. 目前仅支持Java语言； |
| 扫描速度 | 速度一般 |  |
| 数据输入方式 | 页面手动上传，支持上传.zip, .jar, .apk |  |
| 部署方式 | docker |  |
| 支持语言 | 目前支持Java，Android部分未公开 |  |
| 漏洞覆盖 | 支持多种漏洞类型 |  |

#### 2.3.6 参考资料

*   [Hades官方GitHub](https://github.com/zsdlove/Hades)；
*   [PPT：基于虚拟执行技术的静态代码审计系统内幕揭秘](https://github.com/zsdlove/Hades/blob/master/%E5%9F%BA%E4%BA%8E%E8%99%9A%E6%8B%9F%E6%89%A7%E8%A1%8C%E6%8A%80%E6%9C%AF%E7%9A%84%E9%9D%99%E6%80%81%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E7%B3%BB%E7%BB%9F%E5%86%85%E5%B9%95%E6%8F%AD%E7%A7%98.pdf)；
*   [DevSecOps建设之白盒篇](https://www.freebuf.com/articles/es/259762.html);

### 2.4 Fortify

#### 2.4.1 介绍

Fortify是一款商业级的静态应用程序安全性测试 (SAST) （源码扫描）工具。其工作示意图如下所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905142639-3a6b0394-0e12-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905142639-3a6b0394-0e12-1.png)

其大致扫描原理是：扫描分析分为两个阶段：Translation和Analysis。

*   Translation: 源码词法分析/语义分析，把各种扫描语言源码转为一种统一的中间语言代码（中间表现形式）。
*   Analysis: 再对中间表现形式进行安全性分析。

#### 2.4.2 安装部署

**安装**：.exe安装程序双击运行，安装过程中选择license ，安装完成后，最后一个关于软件更新的选项不进行勾选。安装完成后将`fortify-common-20.1.1.0007.jar`包复制到Fortify安装目录的`\Core\lib`目录下进行破解，然后需要把 `rules` 目录的规则文件拷贝到安装目录下的 `Core\config\rules` 的路径下（该路径下保存的是Fortify的默认规则库）。

**运行**：点击安装目录`\bin`目录下的`auditworkbench.cmd`运行Fortify。运行页面如下所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905142725-55dfca1a-0e12-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905142725-55dfca1a-0e12-1.png)

#### 2.4.3 扫描测试

点击Advanced Scan -> 选择待扫描的项目 -> 做些简单配置（参见下图） -> 点击“scan”开始扫描：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905142803-6c4298be-0e12-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905142803-6c4298be-0e12-1.png)

扫描完成截图如下所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905142854-8aad06cc-0e12-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905142854-8aad06cc-0e12-1.png)

#### 2.4.4 扫描结果分析

汇总结果如下（**误报率及漏报率仅根据官方最新版本默认配置，实际测试结果进行主观大致预估**）：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210913161417-970d92ca-146a-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210913161417-970d92ca-146a-1.png)

Benchmark计分器自动分析Fortify结果如下图所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905143129-e77f3460-0e12-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905143129-e77f3460-0e12-1.png)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905143211-001a840c-0e13-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905143211-001a840c-0e13-1.png)

**上图表中的关键字**：

*   TP: 真实漏洞中，代码分析工具正确扫描出来的真实漏洞数量；
*   FN: 真实漏洞中，代码分析工具误报的漏洞数量；
*   TN: 假的漏洞中，代码分析工具正确未扫描出来的漏洞数量；
*   FP: 假的漏洞中，代码分析工具误报成真漏洞的数量；
*   TPR = TP / ( TP + FN ): 代码分析工具正确检出真实漏洞的检出率；
*   FPR = FP / ( FP + TN ): 代码分析工具将假漏洞报告为真实漏洞的误报率；
*   Score = TPR - FPR: 随机猜测与标准线的差距；

#### 2.4.5 调研结果

本次对Fortify的调研结果如下：

| **调研参数** | **调研结果** | **总结** |
| --- | --- | --- |
| 漏洞检测/扫描的原理 | 词法分析/语义分析，把各种扫描语言源码转为一种统一的中间表现形式，再对该中间表现形式进行安全性分析 | **优点**：1\. 误报率/漏报率表现良好，检出漏洞类型丰富； |
| 是否支持持续集成 | windows版本不好持续集成，Linux版本可以 | **缺点**：1\. 不支持第三方不安全组件引用的检测；2. 持续集成需要定制开发； |
| 扫描速度 | 速度良好 |  |
| 数据输入方式 | windows版本新建任务时选择源码目录；Linux通过命令行指定源码目录； |  |
| 部署方式 | windows/Linux |  |
| 支持语言 | 支持27+种开发语言（参见：[https://www.microfocus.com/en-us/fortify-languages）](https://www.microfocus.com/en-us/fortify-languages%EF%BC%89) |  |
| 漏洞覆盖 | 支持多种漏洞类型（SQL注入、XSS、命令注入、代码注入、反序列化等等） |

#### 2.4.6 参考资料

*   [C/C++源码扫描系列- Fortify 篇](https://xz.aliyun.com/t/9276)；
*   [代码安全审计（二）Fortify介绍及使用教程](https://www.jianshu.com/p/af331efb84a9)；

### 2.5 CheckMarx

#### 2.5.1 介绍

以下内容引用自Checkmarx官方文档：

Checkmarx是以色列的一家科技软件公司开发的产品。Checkmarx CxSAST是其独特的源码分析解决方案，它提供了用于识别、跟踪和修复源代码中的技术和逻辑缺陷（例如安全漏洞，合规性问题和业务逻辑问题）的工具。无需构建或编译源码，CxSAST可以构建代码元素和流程的逻辑图，随后CxSAST可以查询这个内部代码的示意图。CxSAST带有一个广泛的列表，其中包含数百个针对每种编程语言的已知安全漏洞的预配置查询。使用 CxSAST Auditor 工具，也可以为安全、合规、业务逻辑问题等写规则配置附加查询。

CxSAST可以集成到软件开发周期的多个流程中。例如，使用软件构建自动化工具（Apache Ant和Maven）， 软件开发版本控制系统（GIT），问题跟踪和项目管理软件（JIRA），存储库托管服务（GitHub），应用程序漏洞管理平台（ThreadFix），持续集成平台（Bamboo和Jenkins），持续代码质量检查平台（SonarQube）和源代码管理工具（TFS）等。

CxSAST部署在服务器上，可以通过web页面或IDE插件进行访问（Eclipse，Visual Studio和IntelliJ）。CxSAST系统架构支持集中式架构（所有服务器组件都安全在同一台主机上）、分布式架构（服务器组件都安装在不同的专用主机上）、高可用架构（多个管理器可用于控制系统管理，确保在一个管理器发生故障时，系统将继续全面运行）。

CxSAST 包含组件如下：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905144828-46a7ef16-0e15-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905144828-46a7ef16-0e15-1.png)

**CxSAST** **服务器组件**：

*   **CxEngine**：执行代码扫描；
*   **数据库**：保存扫描结果和系统设置；
*   **CxManager**：管理系统，执行所有系统功能和集成系统组件；
*   **CxSAST Web客户端** ：控制 CxManager 操作的主界面（例如开始扫描、查看结果和生成报告）。

#### 2.5.2 安装部署

之前在win10系统进行安装部署，安装过程花了很长时间，安装完成后，一直无法登录成功，后面抓包看了下，登录会返回500错误。后面了解了下，可能是因为操作系统版本问题（但是官方文档说是支持win10系统安装的），随后装了个Windows Server 2012的虚拟机，安装过程一遍成功，且可正常使用。

相关机器配置需求可参考官方文档：服务器主机要求([https://checkmarx.atlassian.net/wiki/spaces/KC/pages/126491572/Server+Host+Requirements+cn)，Windows](https://checkmarx.atlassian.net/wiki/spaces/KC/pages/126491572/Server+Host+Requirements+cn)%EF%BC%8CWindows) Server 2012虚拟机的配置如下：

> 6G内存、60GB硬盘空间、处理器核心总数4个

安装过程如下：

1.  双击`CxSetup_8.6.0.exe`运行安装，当进行到“提示输入license文件”时，选择“Request New License”，随后等待安装完成；
2.  待安装完成后，将`Crack`目录拷贝到Checkmarx的安装目录下（例如默认的安装目录：`C:\Program Files\Checkmarx`）；
3.  随后进入Checkmarx安装目录下的`Crack`目录，以管理员权限运行`CRACK.bat`,待脚本执行完成，Checkmarx就被破解成功了；
4.  双击桌面上的"Checkmarx Portal"图标开始使用Checkmarx 。

相关过程安装截图如下：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905144934-6dd26814-0e15-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905144934-6dd26814-0e15-1.png)  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145007-8173f176-0e15-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145007-8173f176-0e15-1.png)

*   **运行**：

第一次运行页面会提示设置用户名密码，随后即可使用用户名密码登录：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145035-92b0c6da-0e15-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145035-92b0c6da-0e15-1.png)

登录成功后，页面如下所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145113-a9224bf0-0e15-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145113-a9224bf0-0e15-1.png)

在“我的配置”页面，可配置语言为中文简体：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145238-db96e622-0e15-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145238-db96e622-0e15-1.png)

#### 2.5.3 扫描测试

点击“项目组和扫描”-\> “创建新的扫描”，一步步进行配置，在源码获取方式处上传zip格式代码文件，配置调度（现在执行/或配置其他时间执行），随后即可进行扫描：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145319-f3fc88ca-0e15-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145319-f3fc88ca-0e15-1.png)

正在扫描中的任务，会在“项目组和扫描” -> “队列”中显示。扫描完成的任务，可以在“项目组和扫描” -> “全部扫描”或“仪表盘” -\> "项目状态"中显示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145429-1dc907aa-0e16-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145429-1dc907aa-0e16-1.png)

扫描完成截图如下所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145454-2caceb42-0e16-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145454-2caceb42-0e16-1.png)

#### 2.5.4 扫描结果分析

汇总结果如下（**误报率及漏报率仅根据官方最新版本默认配置，实际测试结果进行主观大致预估**）：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210913161454-ad382c7c-146a-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210913161454-ad382c7c-146a-1.png)

Benchmark计分器自动分析Checkmarx结果如下图所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145646-6f96d882-0e16-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145646-6f96d882-0e16-1.png)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905145838-b23057d6-0e16-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905145838-b23057d6-0e16-1.png)

**上表中的关键字**：

*   TP: 真实漏洞中，代码分析工具正确扫描出来的真实漏洞数量；
*   FN: 真实漏洞中，代码分析工具误报的漏洞数量；
*   TN: 假的漏洞中，代码分析工具正确未扫描出来的漏洞数量；
*   FP: 假的漏洞中，代码分析工具误报成真漏洞的数量；
*   TPR = TP / ( TP + FN ): 代码分析工具正确检出真实漏洞的检出率；
*   FPR = FP / ( FP + TN ): 代码分析工具将假漏洞报告为真实漏洞的误报率；
*   Score = TPR - FPR: 随机猜测与标准线的差距；

#### 2.5.5 调研结果

本次对Checkmarx的调研结果如下：

| **调研参数** | **调研结果** | **总结** |
| --- | --- | --- |
| 漏洞检测/扫描的原理 | 将代码进行语法树分析，分析代码中的数据流，并将整个代码中的数据流存储到sql server数据库中，可以理解成一张庞大的数据流网，这个分析比较吃内存。随后匹配规则（checkmarx自带了各种规则，也可以自己编写规则），规则也叫query，意思就是从整个数据流网中查找我们关心的数据流。 | **优点**：1\. 误报率/漏报率表现良好；2. 对CI集成很友好，支持多种集成方式；3. 提供接口，可进行定制开发；4. web页面操作简单，报告类型较丰富； |
| 是否支持持续集成 | Checkmarx支持与Jenkins、TFS、 Bamboo、TeamCity等做持续集成：[CI/CD Plugins](https://checkmarx.atlassian.net/wiki/spaces/SD/pages/1339129990/CI+CD+Plugins)，也有提供接口可进行二次开发做持续集成 | **缺点：** 1\. 部署及扫描分析过程依赖机器配置；2. 默认的规则基本上实用性比较差，自己定制规则比较复杂费时；3. 前后端分离的项目分析无法支持，比如后端spring mvc，前端vue就无法关联分析了；4. 也不支持 vue、react等框架文件的分析； |
| 扫描速度 | 速度一般（依赖机器配置） |  |
| 数据输入方式 | 通过git、svn、或者上传.zip格式的源码包 |  |
| 部署方式 | Windows + SQL Server，支持集中部署、分布式部署、高可用部署 |  |
| 支持语言 | 支持25+种开发语言（参见：[https://checkmarx.atlassian.net/wiki/spaces/KC/pages/244810229/8.6.0+Supported+Code+Languages+and+Frameworks）](https://checkmarx.atlassian.net/wiki/spaces/KC/pages/244810229/8.6.0+Supported+Code+Languages+and+Frameworks%EF%BC%89) |  |
| 漏洞覆盖 | 支持漏洞类型非常多（参见：[https://checkmarx.atlassian.net/wiki/spaces/KC/pages/244745198/8.6.0+Vulnerability+Queries?preview=/244745198/244712034/Vulnerability%20Queries%20for%20v8.6.0.pdf）](https://checkmarx.atlassian.net/wiki/spaces/KC/pages/244745198/8.6.0+Vulnerability+Queries?preview=/244745198/244712034/Vulnerability%20Queries%20for%20v8.6.0.pdf%EF%BC%89) |

#### 2.5.6 参考资料

*   [Checkmarx官方文档-知识中心](https://checkmarx.atlassian.net/wiki/home)；

### 2.6 SonarQube

#### 2.6.1 介绍

SonarQube通过检查代码并查找错误和安全漏洞来提供静态代码分析。SonarQube是由SonarSource开发的一款开源工具。其中Community Edition版本提供静态代码分析功能，可支持Java，JavaScript to Go和Python等约15种语言。通过 SonarQube 可以检测出项目中潜在的Bug、漏洞、代码规范、重复代码、缺乏单元测试的代码等问题，并提供了 UI 界面进行查看和管理。

架构如下图所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150041-fba94530-0e16-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150041-fba94530-0e16-1.png)

#### 2.6.2 安装部署

*   **docker部署SonarQube：** 

\# 创建个独立网段（因为在测试机部署，可能跟其他同事的业务网段冲突，所以我这边直接建个新网段）
docker network create --subnet 172.41.0.0/16 --driver bridge sonarnet

docker pull sonarqube:8.8-community

docker run --name sonarqube --net=sonarnet \
   --restart always  \
   -p 9000:9000 \
   -v /sonarqube/data:/opt/sonarqube/data \
   -v /sonarqube/extensions:/opt/sonarqube/extensions \
   -v /sonarqube/logs:/opt/sonarqube/logs \
   -d sonarqube:8.8-community

*   **安装中文汉化包：** 

Administration -> Marketplace ，搜索 chinese ， install Chinese Pack，随后重启即可：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150157-292a990a-0e17-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150157-292a990a-0e17-1.png)

*   **安全相关规则集成：** 

SonarQube自带的规则对代码Bug的检查支持比较友好，跟安全相关的规则比较少。网上查找了很多相关资料，针对PHP的安全相关规则很少，故后文对PHP的扫描测试只能使用默认规则；针对Java比较好的方案是：在SonarQube使用Dependency-Check + SpotBugs的FindBugs Security Audit规则：

*   [Dependency-Check](https://www.owasp.org/index.php/OWASP_Dependency_Check)：Owasp开发的一款工具，用户检测项目中的依赖关系中是否包含公开披露的漏洞；
*   [SpotBugs](https://github.com/spotbugs/sonar-findbugs/)：是Findbugs的继任者（Findbugs已经于2016年后不再维护），用于对Java代码进行静态分析，查找相关的漏洞，SpotBugs比Findbugs拥有更多的校验规则。

> 安装FindBugs：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150246-460bae6a-0e17-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150246-460bae6a-0e17-1.png)

> 安装Dependency-Check：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150330-6024a338-0e17-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150330-6024a338-0e17-1.png)

#### 2.6.3 扫描测试

由于SonarQube对于安全问题的扫描，依赖第三方的规则。故下文的扫描测试，我们分别使用不同的规则进行测试。

##### 2.6.3.1 扫描PHP项目（使用SonarQube默认规则）

使用默认的规则，故直接默认配置即可。步骤如下：

新建项目 -> 创建令牌 -\> 构建技术选择PHP、操作系统选择Linux：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150411-7910327c-0e17-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150411-7910327c-0e17-1.png)

参照上图中的示例命令，扫描端执行的命令如下：

/home/sonar-scanner-4.6.0.2311-linux/bin/sonar-scanner \
-Dsonar.projectKey=Dvwa-Scan \
-Dsonar.sources=. \
-Dsonar.host.url=http://10.0.3.158:9000 \
-Dsonar.login=f3111c1b761d376c5091d5cf674390efdd41df09

待扫描完成后，页面上就可以看到扫描数据。

##### 2.6.3.2 扫描Java项目（使用SonarQube默认规则）

使用默认的规则，故也直接默认配置即可。步骤如下：

新建项目 -> 创建令牌 -\> 构建技术选择Maven、操作系统选择Linux：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150448-8ef03a2e-0e17-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150448-8ef03a2e-0e17-1.png)

参照上图中的示例命令，扫描端执行的命令如下：

mvn sonar:sonar \
-Dsonar.projectKey=benchmark\_findbugs\_scan \
-Dsonar.java.binaries=target/classes \
-Dsonar.host.url=http://10.0.3.158:9000 \
-Dsonar.login=27a9b110822fcfd1828d8ac0f1a07c6b857d331a

##### 2.6.3.3 扫描Java项目（使用Dependency-Check + SpotBugs的FindBugs Security Audit规则）

扫描端执行的命令如下使用及扫描过程会相对繁琐，具体流程如下：

(1) 编译项目：

\# 扫描端：待扫描的源码目录下先编译项目
mvn clean install -DskipTests

(2) Dependency扫描并生成xml报告：

\# 扫描端：使用Dependency进行扫描，并生成xml报告
/home/dependency-check/bin/dependency-check.sh -s /home/source_code/java-sec-code-master/target/java-sec-code-1.0.0.jar -f XML -o /home/java-sec-code-report.xml

(3) SonarQube页面配置：

新建项目 -> 创建令牌 -\> 构建技术选择其他、操作系统选择Linux：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150603-bbb2c342-0e17-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150603-bbb2c342-0e17-1.png)

(4) Dependency的xml报告导入SonarQube并执行SonarScanner扫描:

/home/sonar-scanner-4.6.0.2311-linux/bin/sonar-scanner -Dsonar.host.url=http://192.168.50.168:9000 -Dsonar.login=28f66633ad3b5c89fcbd80022011f519f0868b85 -Dsonar.projectKey=java\_sec\_scan_dependency -Dsonar.java.binaries=/home/source_code/java-sec-code-master -Dsonar.dependencyCheck.reportPath=/home/java-sec-code-report.xml -Dsonar.dependencyCheck.htmlReportPath=/home/java-sec-code-report.html

> **故针对Java语言的扫描，使用Dependency-Check + SpotBugs的FindBugs Security Audit规则的方案，效果最好。** 

扫描结果对比如下图所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150722-eaa50d4a-0e17-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150722-eaa50d4a-0e17-1.png)

#### 2.6.4 扫描结果分析

汇总结果如下（**误报率及漏报率仅根据官方最新版本默认配置，实际测试结果进行主观大致预估**）：

_注：针对PHP使用的是SonarQube默认规则，扫描结果中确认的安全漏洞基本很少，故我们使用需要界面复审Security Reports中的结果和确认的安全漏洞进行对比分析。_

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210913161542-c9f416a0-146a-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210913161542-c9f416a0-146a-1.png)

#### 2.6.5 调研结果

| **调研参数** | **调研结果** | **总结** |
| --- | --- | --- |
| 漏洞检测/扫描的原理 | 基于AST抽象语法树 | **优 点**：1\. 项目开源，便于做一些定制开发；2. 可针对不同的漏洞类型自定义规则；3. 对可持续集成支持较好；4. 插件支持，方便使用（例如findbugs等）; |
| 是否支持持续集成 | 对CI集成支持较好且配置较简单 | **缺点：** 1\. 偏代码质量检测，对安全漏洞检测效果一般；2. 因为findbugs、Dependency等插件支持，针对Java的安全扫描效果优于PHP；3. Sonarqube与Rips合并之后，社区版本的部分功能被移到商业版本中，例如报告生成。 |
| 扫描速度 | 良好 |  |
| 数据输入方式 | 支持git链接输入或在sonar-scanner指定项目目录 |  |
| 部署方式 | 可支持单节点部署/集群部署， SonarQube(提供web页面)可通过docker部署，sonar-scanner支持Linux/Windows等终端运行。详见：（SonarQube Server 安装方法：[https://docs.sonarqube.org/latest/setup/install-server/）](https://docs.sonarqube.org/latest/setup/install-server/%EF%BC%89) |  |
| 支持语言 | 社区版本支持15种常见开发语言 |  |
| 漏洞覆盖 | 覆盖漏洞类型较多 |

#### 2.6.6 参考资料

*   [SonarQube实现自动化代码扫描](https://mp.weixin.qq.com/s/L5WeEFvu6etVTAigx6jjcQ)；
*   [SonarQube使用手册](http://www.zhechu.top/2020/05/23/SonarQube%E4%BD%BF%E7%94%A8%E6%89%8B%E5%86%8C/);
*   [SonarQube 之 gitlab-plugin 配合 gitlab-ci 完成每次 commit 代码检测](https://blog.csdn.net/aixiaoyang168/article/details/78115646)；
*   [Jenkins构建Maven项目](https://www.yuque.com/sunxiaping/yg511q/tl7t1u)；
*   [SonarQube 搭建代码质量管理平台（一）](https://beckjin.com/2018/09/24/sonar-build/);

### 2.7 CodeQL

#### 2.7.1 介绍

以下介绍的内容引用自:[《58集团白盒代码审计系统建设实践1：技术选型》](https://mp.weixin.qq.com/s/d9RzCFkYrW27m1_LkeA2rw)：

CodeQL是 Github 安全实验室推出的一款静态代码分析引擎，其利用QL语言对代码、执行流程等进行“查询”，以此实现对代码的安全性白盒审计，进行漏洞挖掘（`codeql`的工作方式是首先使用`codeql`来编译源码，从源码中搜集需要的信息，然后将搜集到的信息保存为代码数据库文件，用户通过编写`codeql`规则从数据库中搜索出匹配的代码）。

**整体流程**

*   通过适配各个语言的AST解析器，并将代码的AST解析结果按照预设好的数据模型将代码AST数据及其依赖关系存储到CodeDB里；
*   通过QL语言定义污点追踪漏洞模型；
*   执行QL时通过高效的搜索算法对CodeDB的AST元数据进行高效查询，从而在代码中搜索出漏洞结果。

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905150946-40422b52-0e18-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905150946-40422b52-0e18-1.png)

CodeQL为白盒漏洞扫描提供了一些新的思路：

*   通过格式化AST数据将它们进行结构化存储，再通过高效率的有向图搜索/裁剪算法支持对这些元数据进行基本查询；
*   通过对不同语言的适配，把一些常用的查询封装成QL语言，对AST数据进行一种类SQL的查询；
*   通过QL语言定义漏洞查询三元组<sources,sinks,sanitizers>（即污点追踪）可进行漏洞查询建模，并且由于查询可以直接搜索AST元数据，可以对DataFlow以及ControlFlow进行更细致的判断，以减少误报、漏报；
*   通过悬赏开源社区收录QL规则，保障了规则的迭代更新；
*   通过CodeQL的开源规则以及Github庞大的开源代码及迭代数据，实现打标能力为后续LGTM平台的神经网络学习提供学习样本。

#### 2.7.2 安装部署

本次调研仅对CodeQL环境进行部署搭建，同时参考文章：[《如何用CodeQL数据流复现 apache kylin命令执行漏洞》](https://xz.aliyun.com/t/8240)进行该漏洞的本地复现，其他未深入进行研究。

**(1) 代码编译及代码数据库创建（Linux机器）：** 

下载Linux版本的CodeQL分析程序，下载链接：[https://github.com/github/codeql-cli-binaries/releases，解压即可。](https://github.com/github/codeql-cli-binaries/releases%EF%BC%8C%E8%A7%A3%E5%8E%8B%E5%8D%B3%E5%8F%AF%E3%80%82)

**(2) 编写查询及编写CodeQL规则（本地windows机器）：** 

*   下载安装VSCode ；
*   VSCode中安装CodeQL插件；
*   下载Windows版本的CodeQL分析程序（下载链接与Linux版本同），解压即可；
*   下载[vscode-codeql-starter](https://github.com/github/vscode-codeql-starter)，用作VSCode的CodeQL启动器。

\# vscode-codeql-starter命令行安装（Windows需安装git）
git clone https://github.com/github/vscode-codeql-starter
cd .\\vscode-codeql-starter\
git submodule update --init --remote

\# vscode-codeql-starter手动安装：
\## 下载zip包：https://github.com/github/vscode-codeql-starter，随后解压。
\## 下载codeql其他语言（Java、PHP等）的规则文件：https://github.com/github/codeql/tree/lgtm.com，将包中所有文件复制到vscode-codeql-starter目录的\\ql目录下。
\## 下载codeql GO语言的规则文件：https://github.com/github/codeql-go/tree/lgtm.com，将保重所有文件复制到vscode-codeql-starter目录的\\codeql-go目录下。

**(3) VSCode配置（本地windows机器）：** 

**配置codeql路径：** CodeQL的Windows版分析程序下载解压后，在VSCode中的CodeQL插件中配置`Executable Path`为`codeql.exe`的路径，如下图所示：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905151031-5b2e4f2c-0e18-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905151031-5b2e4f2c-0e18-1.png)

**打开vscode-codeql-starter的工作目录：** 文件 > 打开工作区 \> 选择`vscode-codeql-starter`目录下的`vscode-codeql-starter.code-workspace`即可:

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905151125-7b3230c2-0e18-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905151125-7b3230c2-0e18-1.png)

#### 2.7.3 扫描测试

**(1) 以Apache Kylin命令注入漏洞（CVE-2020-1956）为例：** 

/home/codeql_test/codeql/codeql database create java-kylin-test --language=java

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905151202-9191c558-0e18-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905151202-9191c558-0e18-1.png)

将数据库打包：

/home/codeql_test/codeql/codeql database bundle -o java-kylin-test.zip java-kylin-test

随后将数据库的.zip包拷贝到本机，使用vscode进行分析：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905151242-a94d2b06-0e18-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905151242-a94d2b06-0e18-1.png)

编写查询代码：

/**
 \* @name cmd_injection
 \* @description 命令注入.
 \* @kind path-problem
 \* @problem.severity error
 \* @precision high
 \* @id java/cmd-injection-raul17
 \* @tags security
 \* external/cwe/cwe-089
 */

import semmle.code.java.dataflow.FlowSources
import semmle.code.java.security.ExternalProcess
import DataFlow::PathGraph
// import DataFlow::PartialPathGraph 
import semmle.code.java.StringFormat
import semmle.code.java.Member
import semmle.code.java.JDK
import semmle.code.java.Collections

class WConfigToExec extends TaintTracking::Configuration {
  WConfigToExec() { this = "cmd::cmdTrackingTainted" }

  override predicate isSource(DataFlow::Node source) {
    source instanceof RemoteFlowSource 
   }

  override predicate isSink(DataFlow::Node sink) {
    sink.asExpr() instanceof ArgumentToExec     
  }

}

class CallTaintStep extends TaintTracking::AdditionalTaintStep {
  override predicate step(DataFlow::Node n1, DataFlow::Node n2) {
    exists(Call call |
      n1.asExpr() = call.getAnArgument() and
      n2.asExpr() = call
    )
  }
}

from  DataFlow::PathNode source,DataFlow::PathNode  sink,WConfigToExec c
where c.hasFlowPath(source, sink)
select source.getNode(), source, sink, "comes $@.", source.getNode(), "input"

查询出三个利用链：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905151318-bebed91c-0e18-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905151318-bebed91c-0e18-1.png)

#### 2.7.4 调研结果

| **调研参数** | **调研结果** | **总结** |
| --- | --- | --- |
| 漏洞检测/扫描的原理 | 利用QL语言对代码、执行流程等进行“查询”，以此实现对代码的安全性白盒审计，进行漏洞挖掘。 | 优缺点引用自[《58集团白盒代码审计系统建设实践1：技术选型》](https://mp.weixin.qq.com/s/d9RzCFkYrW27m1_LkeA2rw)：**优点**：1\. 以CodeDB的模式存储源代码数据，并提供高效的搜索算法及QL语言对漏洞进行查询，支持数据流与控制流，使用者无需考虑跨文件串联的算法难度；2. 支持除PHP以外的常见语言类型；3. QL规则开源且正在迭代，有很非常强的可扩展性及支持定制化规则能力；4. QL规则有相应的文档学习，无需依赖厂商支持； 5. 可以深入Jar包进行漏洞扫描。 |
| 是否支持持续集成 | 不支持企业集成到CI/CD流程 | **缺点：** 1\. AST分析引擎不开源，无法针对AST的元数据进行调整修改，并且官方申明只用于研究用途，不允许企业集成至CI/CD流程；2. 不支持运行时动态绑定的重载方法分析（其他SAST产品的也不支持）；3. 不支持R·esource文件的扫描，不做二次开发的情况下无法支持类似Mybatis XML配置的场景；4. 不支持软件成分分析，无法结合软件版本进行漏洞判断。 |
| 扫描速度 | 较快 |  |
| 数据输入方式 |  |  |
| 部署方式 | Linux安装分析程序进行分析，Windows安装分析程序结合vscode进行查询语句编写 |  |
| 支持语言 | 支持除PHP以外的常见语言类型 |  |
| 漏洞覆盖 | 覆盖常见漏洞类型 |

#### 2.7.5 参考资料

*   [CodeQL官方文档](https://codeql.github.com/docs/)；
*   [C/C++源码扫描系列- codeql 篇](https://xz.aliyun.com/t/9275)；
*   [代码分析引擎 CodeQL 初体验](https://paper.seebug.org/1078/)；
*   [如何用CodeQL数据流复现 apache kylin命令执行漏洞](https://xz.aliyun.com/t/8240)；
*   [系列 | 58集团白盒代码审计系统建设实践1：技术选型](https://mp.weixin.qq.com/s/d9RzCFkYrW27m1_LkeA2rw)；
*   [系列 | 58集团白盒代码审计系统建设实践2：深入理解SAST](https://mp.weixin.qq.com/s/jQfsUg4vhEs3XwTcXkqhyQ)；

0x03 附录
-------

### 3.1 OWASP Benchmark计分器使用

#### 3.1.1 docker镜像安装

下载链接：

https://github.com/OWASP/Benchmark/releases/tag/1.2beta

将Fortify、CheckMark对Benchmark的扫描结果导出为特定格式的扫描报告，格式支持参见：[https://owasp.org/www-project-benchmark/](https://owasp.org/www-project-benchmark/)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905151450-f5b8c50e-0e18-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905151450-f5b8c50e-0e18-1.png)

将扫描报告复制到`/results`目录下，随后进入`/VMs`目录，直接运行`docker build -t benchmark .`进行镜像build。

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905151516-053bb982-0e19-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905151516-053bb982-0e19-1.png)

#### 3.1.2 计分器使用

使用如下命令进入容器：

docker container run -it benchmark1:latest /bin/bash

在容器内部直接运行`./createScorecards.sh`脚本：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20210905151540-13297c8c-0e19-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20210905151540-13297c8c-0e19-1.png)

运行完成后，随即就会在`/scorecard`目录下生成.html、.csv、.png格式的计分结果报告。

* * *

**<全文完>**