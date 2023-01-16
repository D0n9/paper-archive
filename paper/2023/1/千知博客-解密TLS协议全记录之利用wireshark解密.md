# 千知博客-解密TLS协议全记录之利用wireshark解密
版权声明：本文为CSDN博主「wallEVA96」的原创文章，遵循CC 4.0 BY-SA版权协议。  
原文链接：[https://blog.csdn.net/walleva96/article/details/106844033](https://blog.csdn.net/walleva96/article/details/106844033)

### 引言

为什么会突然有使用wireshark学习TLS的想法，主要是为了一个抢票设计，结果一入TLS，无法自拔，最后发现路子好像走歪了，唯一的价值好像就是多了这么一篇博客，查阅了很多有根据，没根据的博客内容，总结出这篇自以为还算全面的文章。如有问题，欢迎讨论交流。  
学习网络协议之前，必须先找对最基本的协议学习网站，  
rfc协议官网 [https://www.rfc-editor.org](https://www.rfc-editor.org/)  
ietf协议官网 [https://www.ietf.org/](https://www.ietf.org/) ，其实这个网站，我经常搜索不出比较理想的文件，标题不直观，杂七杂八的会议记录文件一大堆夹杂在规范文件之中。  
通过上面的网站，我们可以获取到tls的几个协议说明链接.  
TLS1.0: [https://www.rfc-editor.org/rfc/rfc2246.html](https://www.rfc-editor.org/rfc/rfc2246.html)  
TLS1.2: [https://www.rfc-editor.org/rfc/rfc5246.html](https://www.rfc-editor.org/rfc/rfc5246.html) [https://tools.ietf.org/html/rfc5246](https://tools.ietf.org/html/rfc5246)

### 1\. 写在解密前

这个分析TLS报文的环节中引出很多问题，一步步被引入深坑。  
学习的时候，被不少没有实践的博客坑得浪费了不少时间，之前参考过下面的内容：

#### 1.1 安装wireshark

从Wireshark官网下载软件wireshark官网下载略慢，  
也可以选择从下面链接获取：  
链接:[https://pan.baidu.com/s/1mXzS4XMGxYObz_O56Rgphw%3E](https://pan.baidu.com/s/1mXzS4XMGxYObz_O56Rgphw%3E) 提取码：t4om

#### 1.2 有用的知识

从wireshark的wiki中，[https://wiki.wireshark.org/](https://wiki.wireshark.org/), 可以查阅到网络协议在wireshark上使用相关的教程，比如本文中TLS，[https://wiki.wireshark.org/TLS](https://wiki.wireshark.org/TLS)

##### 1.2.1 易混淆点

SSLv2/SSLv3不安全，因此，发展出了使用TLS来代替，利用传输层的TCP协议来传输报文，端口号 443, 将应用层（HTTP）封装在tls的加密密文中。X.509 认证证书有时候也叫 SSL 证书。  
由于一些历史原因， 诸多的软件中(包括wireshark)提到 SSL 或者 SSL/TLS ，指的都是 目前大家都在使用的TLS协议，比如 wireshark早些版本就叫SSL，现在都统一成TLS。  
典型地， TLS使用TCP 作为他的传输层协议， 这几年也逐步发展出了DTLS(Datagram TLS)， 也就是基于UDP传输的TLS报文协议。 UDP传输协议（英语：User Datagram Protocol，用户数据包协议），网络编程的时候，经常通过SOCK_DGRAM参数来配置socket。  
自从wireshark 3.0之后， TLS解析工具就从SSL改成了TLS，如果使用SSL的话，软件将会发出警告。

##### 1.2.2 wireshark 使用技巧

*   配置过滤选项为： tcp.port = 443 ,便可以过滤TLS报文
*   当我们想追踪某一次数据通信的相关报文的时候， 可以右键该报文，然后选择追踪流， 追踪TLS 协议,便可以将相关的TLS报文 都筛选出来，也有其他选项，比如追踪TCP报文之类的，我们就可以很清晰观察到报文的一些 握手， 挥手信息， TLS的交换密钥的报文信息。
*   配置过滤的时候， 同样可以右键报文的ip地址 ，选择作为过滤器使用，然后可以组合各种过滤的逻辑语法， 之后会作用到 Current Filte中
*   在Current Filter中， 如果不小心删除之前使用的逻辑判断语句，如ip.addr=192.168.1.1,可以点击右侧的 ▼(倒三角符号)，状态栏便会下拉出历史使用的过滤选项。

##### 1.2.3 学习TLS协议内容

学习TLS协议的话， 可以查看我的另外一篇专门解析TLS协议的博文： [https://blog.csdn.net/walleva96/article/details/107093408](https://blog.csdn.net/walleva96/article/details/107093408)

### 2\. 通过wireshark 解密TLS报文的两种途径

进行wireshark报文解密的话， 可以直接抓取现成网页服务器的数据， 也可以自己动手通过nginx搭建HTTPS Server进行学习。

在我的另外一篇博客中， 有详解如何搭建利用openssl 搭建 https 服务器教程： [https://blog.csdn.net/walleva96/article/details/107093408](https://blog.csdn.net/walleva96/article/details/107093408)  
Wireshak 主要支持以下两种方法进行解密：

1).在RSA密钥交换算法中，客户端会临时生成预主密钥(随机数)，这个数在会被生成主密钥之后就会被马上删除， 那么此时只能通过服务端的私钥，才能够解开加密密钥进而解密报文，但是这大多数情况下，不现实。  
2).而例如chrome，firefox， curl 等应用， 当设置了SSLKEYLOGFILE的环境变量， 就能够获取到每次对话产生的key log文件。利用每次对话中，存储下来的key log来解密报文是一种非常普遍的手段，即使是使用DH密钥交换算法， 而RSA私钥解密就比较有局限性，它只能应用于以下几种环境：

*   服务器加密套件使用的不是(EC)DHE.
*   协议版本是SSLv3， TLS1.0-1.2， 因为TLS1.3 已经不再支持RSA密钥交换算法了， 可查看我的博客： TLS协议剖析.
*   RSA私钥需要和服务器证书匹配，正常通信中，服务器肯定会下发证书， 这时候证书中公钥就会被用来加密预主密钥，如果能够拿到证书对应的私钥，便可以解密被加密的报文内容。（结合非对称加密理解。不能搞混，不是客户端的证书也不是CA证书。
*   对话窗不能是通过恢复协议而恢复的窗口， 报文中一定要包含有`ClientKeyExchange`的握手消息,这个地方我吃过亏，先前实践的过程中，一直十分郁闷为何有的报文有被解密，有的没有解密
*   RSA私钥唯一的优点就在于，只要导入一次服务器的私钥文件就可以实现内容解密。

> Q： wireshark 有时候无法解密是不是就是因为报文通信的密钥使用的是之前商议好的密钥？  
> A: 确实如此，如果是使用RSA密钥交换算法，那么报文中一定要包含ClientKeyExchange的消息， 这样wireshark 才能拿到客户端产生的随机数(预主密钥)，才能把密钥和报文匹配起来， 而恢复会话就很难确定之前的密钥是什么值了。

#### 2.1 首先，配置测试网络：

网络拓扑图：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/ea278c87-1ace-4443-b33b-f047e99d29a5.png?raw=true)

配置wiresshark 过滤出tls的报文：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/91c87a12-eb3d-4d2c-b837-651855aceb6a.png?raw=true)

或者配置 tcp.port == 443

#### 2.2 使用服务器端的RSA 私钥进行解密

这个途径要拿到各种官方服务器基本不太现实，但是，可以自己搭建https 服务器进行测试，我的博客[https://blog.csdn.net/walleva96/article/details/107093408](https://blog.csdn.net/walleva96/article/details/107093408), 有讲述如何搭建https 服务器来进行加密通信。

##### 2.2.1 配置服务器使用RSA的加密套件：

```
#HTTPS server
server {
listen 443 ssl;
server_name localhost;
ssl_certificate /usr/local/nginx/tls_file/cert_2.crt;
ssl_certificate_key /usr/local/nginx/tls_file/cert.key;
ssl_session_cache shared:SSL:1m;
ssl_session_timeout 5m;
ssl_ciphers AES256-GCM-SHA384;
#ssl_ciphers ECDHE-RSA-CHACHA20-POLY1305;
#ssl_ciphers HIGH:!aNULL:!MD5;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
location / {
root html;
index index.html index.htm;
}
}
```

##### 2.2.2 访问页面之后，可以抓到典型的RSA密钥交换算法的报文，也就是只有client这一侧进行key exchange，通过公钥加密了随机数，然后上传给服务器。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/3b6e19b2-b6bf-492a-8f4f-2eff7a2e9fcd.png?raw=true)
  

##### 2.2.3 通过编辑—> 首选项—>RSA密钥：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/6c16cade-fb19-4c1c-84e2-c82fe257737b.png?raw=true)
  

导入服务器上的私钥文件，类似以下文件内容

```
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDReQzlKVeAK8b5
TRcRBhSi9IYwHX8Nqc8K4HeDRvN7HiBQQP3bhUkVekdoXpRLYVuc7A8h1BLr93Qw
…
KOi8FZl+jhG+p8vtpK5ZAIyp
-----END PRIVATE KEY-----
```

这样即可解密，解密后的报文如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/8eae8a73-e335-4989-af28-e8fcc55872a3.png?raw=true)

则可以在wireshark 上查看到http的明文报文。需要注意的是RSA算法，一定要让wireshark 抓到 Client Key Exchange的包，因为预主密钥 在这个报文里面，否则结果会像下面这样， 无法看到明文。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/bc11e06c-400e-4a5f-8bab-1ccfc557d248.png?raw=true)

#### 2.3 使用(Pre)-Master-Secret的keylog 文件，配置TLS_debug 文件

*   这个是利用wireshark 解密报文较为普遍的一种方式。
*   通过TLS debug file (tls.debug_logfile)文件， 我们可以知道为什么解密失败，这其中将会记录wireshark解密过程的相关log信息。

##### 2.3.1 创建文件 tls_key.txt （用来记录环境变量SSLKEYLOGFILE的值，chrome 和firefox 都会将值记录到这个变量上）

##### 2.3.2 创建文件tls_debug.log (用来保存wireshark 解析报文时的log记录)

##### 2.3.3 打开wireshark，打开Edit–> Preferences -> Protocols -> TLS，将上述文件按如下导入：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/3958814e-d36b-40f8-9956-9a93d0b7e1e1.png?raw=true)
  

##### 2.3.4 配置 SSLKEYLOGFILE的环境变量 特别说明：这种设置临时环境变量的方式，打开浏览器只能通过cmd窗口执行命令的方式，才可以正常解密tls包。

```
# windows下，该方法作用范围只局限于当前的CMD窗口。
C:\Program Files (x86)\Google\Chrome\Application>set SSLKEYLOGFILE=C:\Users\chupa\Desktop\tls_key.txt
C:\Program Files (x86)\Google\Chrome\Application>chrome.exe
# linux下
$ export SSLKEYLOGFILE=~/path/to/sslkeylog.log
```

##### 2.3.5 配置nginx服务器，然后访问网页，可以看到解析的报文如下，可以看到application报文变成http明文。

*   下图为ECDHE密钥交换算法的报文：
    
*   ![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/12c03d1c-90c0-4f3d-9166-b6322fb73544.png?raw=true)
    
*   下图为ECDHE算法中相关参数key值的文件内容：
*   ![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/4d14ecbc-62d5-4f88-a7d1-88c3d15fd302.png?raw=true)
    
*   下图为RSA密钥交换算法的报文解析：
    
*   ![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/bc5f799b-2a26-429c-8b61-f2dcd8bea1ee.png?raw=true)
    
*   下图为RSA算法中，客户端产生的随机值(预主密钥)：
    
*   ![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/700c05dc-2b2b-4f9d-bacd-3f6270d86a5f.png?raw=true)
    
*   以下是访问 nike 官网的报文截图：
    
*   ![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/fa60b850-10ac-4981-9c81-d38d24dffd9e.png?raw=true)
    

可以看到nike官网开始支持http2了，

但是，会发现一个问题，就是部分报文无法解密，比如

查看wireshark 记录的tls_debug.log发现：

```
tls_decrypt_aead_record seq 624
nonce[12]:
| 3d 3d 4a 3f 66 01 6a 69 65 af bd 3e |==J?f.jie…> |
AAD[5]:
| 17 03 03 04 11 |… |
tls_decrypt_aead_record auth tag mismatch
auth_tag(expect)[16]:
| 62 d4 44 5f 34 02 87 73 72 b4 80 0a ba 1f b9 1d |b.D_4…sr…|
auth_tag(actual)[16]:
| 1a 3e ac 27 0b 1a 6a 9a a1 28 1b 89 0e 73 77 42 |.>.’…j…(…swB|
```

> Google搜索发现，已经有人报了相关bug 给wireshark ， bug 链接： [https://bugs.wireshark.org/bugzilla/show_bug.cgi?id=15537](https://bugs.wireshark.org/bugzilla/show_bug.cgi?id=15537) 目前我也报了bug给wireshark,bug链接： [https://bugs.wireshark.org/bugzilla/show_bug.cgi?id=16713，](https://bugs.wireshark.org/bugzilla/show_bug.cgi?id=16713%EF%BC%8C) 等待回复中…
> 
> 当然，wireshark解密TLS 报文还是有一定的局限性，我们可以通过搭建代理服务器，或者 fiddler来解密， 也可以直接通过 chrome 浏览器也可以直接解析，fiddler 其实就是模拟的代理服务器，网上教程很多，不再赘述。