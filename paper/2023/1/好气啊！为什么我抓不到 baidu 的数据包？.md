# 好气啊！为什么我抓不到 baidu 的数据包？
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/3d4df08d-abf8-4f32-84b8-9c12c63c9fa1.png?raw=true)

图解学习网站：xiaolincoding.com

最近，有位读者问起一个奇怪的事情，他说他想抓一个`baidu.com`的数据包，体验下看包的乐趣。

但却发现“**抓不到**”，这就有些奇怪了。

我来还原下他的操作步骤。

首先，通过`ping`命令，获得访问百度时会请求哪个IP。

```
`$ ping baidu.com  
PING baidu.com (39.156.66.10) 56(84) bytes of data.  
64 bytes from 39.156.66.10 (39.156.66.10): icmp_seq=1 ttl=49 time=30.6 ms  
64 bytes from 39.156.66.10 (39.156.66.10): icmp_seq=2 ttl=49 time=30.6 ms  
64 bytes from 39.156.66.10 (39.156.66.10): icmp_seq=3 ttl=49 time=30.6 ms`
```

从上面的结果可以知道请求`baidu.com`时会去访问`39.156.66.10`。

于是用下面的`tcpdump`命令进行抓包，大概的意思是抓`eth0`网卡且`ip`为`39.156.66.10`的网络包，保存到`baidu.pcap`文件中。

```
`$ tcpdump -i eth0 host 39.156.66.10 -w baidu.pcap`
```

此时在浏览器中打开`baidu.com`网页。或者在另外一个命令行窗口，直接用`curl`命令来模拟下。

```
`$ curl 'https://baidu.com'`
```

按理说，**访问baidu.com的数据包肯定已经抓下来了**。

然后停止抓包。

再用`wireshark`打开`baidu.pcap`文件，在过滤那一栏里输入`http.host == "baidu.com"`。

此时发现，一无所获。

在wireshark中搜索baidu的包，发现一无所获

这是为啥？

到这里，有经验的小伙伴，其实已经知道问题出在哪里了。

为什么没能抓到包
--------

这其实是因为他访问的是HTTPS协议的baidu.com。HTTP协议里的Host和实际发送的request body都会被加密。

正因为被加密了，所以没办法通过`http.host`进行过滤。

但是。

虽然加密了，如果想筛选还是可以筛的。

HTTPS握手中的Client Hello阶段，里面有个扩展`server_name`，会记录你想访问的是哪个网站，通过下面的筛选条件可以将它过滤出来。

```
`  tls.handshake.extensions_server_name == "baidu.com"`
```

通过tls的扩展server_name可以搜索到baidu的包

此时选中其中一个包，点击右键，选中`Follow-TCP Stream`。

右键找到tcp 流

这个TCP连接的其他相关报文全都能被展示出来。

HTTPS抓包

从截图可以看出，这里面完整经历了**TCP握手**和**TLS加密握手**流程，之后就是**两段加密信息**和**TCP挥手流程**。

可以看出18号和20号包，一个是从端口56028发到443，一个是443到56028的回包。

一般来说，像`56028`这种比较大且没啥规律的数字，都是**客户端随机生成的端口号**。

而`443`，则是HTTPS的服务器端口号。

> HTTP用的是80端口，如果此时对着80端口抓包，也会抓不到数据。

粗略判断，18号和20号包分别是客户端请求`baidu.com`的请求包和响应包。

点进去看会发现**URL和body都被加密了**，一无所获。

那么问题就来了。有没有办法解密里面的数据呢？

有办法。我们来看下怎么做。

解密数据包
-----

还是先执行tcpdump抓包

```
`$ tcpdump -i eth0 host 39.156.66.10 -w baidu.pcap`
```

然后在另外一个命令行窗口下执行下面的命令，**目的是将加密的key导出，并给出对应的导出地址**是`/Users/xiaobaidebug/ssl.key`。

```
`$ export SSLKEYLOGFILE=/Users/xiaobaidebug/ssl.key`
```

然后在同一个命令行窗口下，继续执行curl命令或用命令行打开chrome浏览器。**目的是为了让curl或chrome继承这个环境变量。** 

```
`$ curl 'https://baidu.com'  
或者  
$ open -a Google\ Chrome #在mac里打开chrome浏览器`
```

此时会看到在`/Users/xiaobaidebug/`下会多了一个`ssl.key`文件。

这时候跟着下面的操作修改`wireshark`的配置项。

打开wireshark的配置项

找到Protocols之后，使劲往下翻，找到`TLS`那一项。

在配置项中找到Protocols

将导出的`ssl.key`文件路径输入到这里头。

在Protocols中找到TLS那一栏

点击确定后，就能看到**18号和20号数据包已经被解密**。

解密后的数据包内容

此时再用`http.host == "baidu.com"`，就能过滤出数据了。

解密后的数据包中可以过滤出baidu的数据包

到这里，其实**看不了数据包的问题**就解决了。

但是，新的问题又来了。

**ssl.key文件是个啥？**

这就要从HTTPS的加密原理说起了。

### HTTPS握手过程

HTTPS的握手过程比较繁琐，我们来回顾下。

先是建立TCP连接，毕竟HTTP是基于TCP的应用层协议。

在TCP成功建立完协议后，就可以开始进入HTTPS阶段。

HTTPS可以用TLS或者SSL啥的进行加密，下面我们以`TLS1.2`为例。

总的来说。整个加密流程其实分为**两阶段**。

**第一阶段**是TLS四次握手，这一阶段主要是利用**非对称加密**的特性各种交换信息，最后得到一个"会话秘钥"。

**第二阶段**是则是在第一阶段的"会话秘钥"基础上，进行**对称加密**通信。

TLS四次握手

我们先来看下第一阶段的TLS四次握手是怎么样的。

**第一次握手**：

*   • `Client Hello`：是客户端告诉服务端，它支持什么样的加密协议版本，比如 `TLS1.2`，使用什么样的加密套件，比如最常见的`RSA`，同时还给出一个**客户端随机数**。
    

**第二次握手**：

*   • `Server Hello`：服务端告诉客户端，**服务器随机数** \+ 服务器证书 \+ 确定的加密协议版本（比如就是TLS1.2）。
    

**第三次握手**：

*   • `Client Key Exchange`: 此时客户端再生成**一个随机数**，叫 `pre_master_key `。从第二次握手的**服务器证书**里取出服务器公钥，用公钥加密 `pre_master_key`，发给服务器。
    
*   • `Change Cipher Spec`: 客户端这边**已经拥有三个随机数**：客户端随机数，服务器随机数和pre\_master\_key，用这三个随机数进行计算得到一个"**会话秘钥**"。此时客户端通知服务端，后面会用这个会话秘钥进行对称机密通信。
    
*   • `Encrypted Handshake Message`：客户端会把迄今为止的通信数据内容生成一个摘要，用"**会话秘钥**"加密一下，发给服务器做校验，此时客户端这边的握手流程就结束了，因此也叫**Finished报文**。
    

**第四次握手**：

*   • `Change Cipher Spec`：服务端此时拿到客户端传来的 `pre_master_key`（虽然被服务器公钥加密过，但服务器有私钥，能解密获得原文），集齐三个随机数，跟客户端一样，用这三个随机数通过同样的算法获得一个"**会话秘钥**"。此时服务器告诉客户端，后面会用这个"会话秘钥"进行加密通信。
    
*   • `Encrypted Handshake Message`：跟客户端的操作一样，将迄今为止的通信数据内容生成一个摘要，用"**会话秘钥**"加密一下，发给客户端做校验，到这里，服务端的握手流程也结束了，因此这也叫**Finished报文**。
    

四次握手中，客户端和服务端最后都拥有**三个随机数**，他们很关键，我特地加粗了表示。

第一次握手，产生的客户端随机数，叫`client random`。

第二次握手时，服务器也会产生一个服务器随机数，叫`server random`。

第三次握手时，客户端还会产生一个随机数，叫`pre_master_key`。

这三个随机数共同构成最终的**对称加密秘钥**，也就是上面提到的"**会话秘钥**"。

三个随机数生成对称秘钥

你可以简单的认为，**只要知道这三个随机数，你就能破解HTTPS通信。** 

而这三个随机数中，`client random` 和 `server random` 都是**明文**的，谁都能知道。**而`pre_master_key`却不行，它被服务器的公钥加密过，只有客户端自己，和拥有对应服务器私钥的人能知道。** 

所以问题就变成了，**怎么才能得到这个`pre_master_key`？**

怎么得到pre\_master\_key
--------------------

服务器私钥不是谁都能拿到的，所以问题就变成了，**有没有办法从客户端那拿到这个`pre_master_key`。** 

有的。

客户端在使用HTTPS与服务端进行数据传输时，是需要先基于TCP建立HTTP连接，然后再调用客户端侧的TLS库（OpenSSL、NSS）。触发TLS四次握手。

这时候如果加入环境变量SSLKEYLOGFILE就可以干预TLS库的行为，让它输出一份含有`pre_master_key`的文件。这个文件就是我们上面提到的`/Users/xiaobaidebug/ssl.key`。

将环境变量注入到curl和chrome中

但是，虽然TLS库支持导出key文件。但前提也是，上层的应用程序在调用TLS库的时候，支持通过`SSLKEYLOGFILE`环境触发TLS库导出文件。实际上，也**并不是所有应用程序都支持将SSLKEYLOGFILE**。只是目前常见的curl和chrome浏览器都是支持的。

SSLKEYLOGFILE文件内容
-----------------

再回过头来看`ssl.key`文件里的内容。

```
`# SSL/TLS secrets log file, generated by NSS  
CLIENT_RANDOM 5709aef8ba36a8eeac72bd6f970a74f7533172c52be41b200ca9b91354bd662b 09d156a5e6c0d246549f6265e73bda72f0d6ee81032eaaa0bac9bea362090800174e0effc93b93c2ffa50cd8a715b0f0  
CLIENT_RANDOM 57d269386549a4cec7f91158d85ca1376a060ef5a6c2ace04658fe88aec48776 48c16429d362bea157719da5641e2f3f13b0b3fee2695ef2b7cdc71c61958d22414e599c676ca96bbdb30eca49eb488a  
CLIENT_RANDOM 5fca0f2835cbb5e248d7b3e75180b2b3aff000929e33e5bacf5f5a4bff63bbe5 424e1fcfff35e76d5bf88f21d6c361ee7a9d32cb8f2c60649135fd9b66d569d8c4add6c9d521e148c63977b7a95e8fe8  
CLIENT_RANDOM be610cb1053e6f3a01aa3b88bc9e8c77a708ae4b0f953b2063ca5f925d673140 c26e3cf83513a830af3d3401241e1bc4fdda187f98ad5ef9e14cae71b0ddec85812a81d793d6ec934b9dcdefa84bdcf3`
```

这里有三列。

**第一列**是CLIENT_RANDOM，意思是接下来的**第二列**就是**客户端随机数**，再接下来的**第三列**则是`pre_master_key`。

但是问题又来了。

**这么多行，wireshark怎么知道用哪行的pre\_master\_key呢？**

`wireshark`是可以获得数据报文上的`client random`的。

比如下图这样。

Client Hello 里的客户端随机数

注意上面的客户端随机数是以 `"bff63bbe5"`结尾的。

同样，还能在数据报文里拿到**server random**。

找到server random

此时将`client random`放到ssl.key的第二列里挨个去做匹配。

就能找到对应的那一行记录。

ssl.key里的数据

注意第二列的那串字符串，也是以 `"bff63bbe5"`结尾的，它其实就是前面提到的`client random`。

再取出这一行的**第三列**数据，就是我们想要的`pre_master_key`。

那么这时候`wireshark`就集齐了三个随机数，此时就可以计算得到**会话秘钥**，通过它对数据进行解密了。

反过来，正因为需要客户端随机数，才能定位到`ssl.key`文件里对应的`pre_master_key`是哪一个。而只有TLS第一次握手（`client hello`）的时候才会有这个随机数，所以如果你想用解密HTTPS包，就必须将TLS四次握手能抓齐，才能进行解密。如果连接早已经建立了，数据都来回传好半天了，这时候你再去抓包，是没办法解密的。

总结
--

*   • 文章开头通过抓包baidu的数据包，展示了用wireshark抓包的简单操作流程。
    
*   • HTTPS会对HTTP的URL和Request Body都进行加密，因此直接在`filter栏`进行过滤`http.host == "baidu.com"`会一无所获。
    
*   • HTTPS握手的过程中会先通过非对称机密去交换各种信息，其中就包括3个随机数，再通过这三个随机数去生成对称机密的会话秘钥，后续使用这个会话秘钥去进行对称加密通信。如果能获得这三个随机数就能解密HTTPS的加密数据包。
    
*   • 三个随机数，分别是客户端随机数（client random），服务端随机数（server random）以及pre\_master\_key。前两个，是明文，第三个是被服务器公钥加密过的，在客户端侧需要通过SSLKEYLOGFILE去导出。
    
*   • 通过设置SSLKEYLOGFILE环境变量，再让curl或chrome会请求HTTPS域名，会让它们在调用TLS库的同时导出对应的sslkey文件。这个文件里包含了三列，其中最重要的是第二列的client random信息以及第三列的pre\_master\_key。第二列client random用于定位，第三列pre\_master\_key用于解密。
    

参考资料
----

极客时间 -《网络排查案例课》

历史好文
----

之前写过很多 HTTPS 文章了，还不清楚 HTTPS 握手流程的同学，该好好重新复习了。

整理了一下，以前写过的 HTTPS 文章：

*   [硬核！40 张图解 HTTP 常见面试题](https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247508461&idx=1&sn=a39cf9a62cb4564a488c829f7c365412&scene=21#wechat_redirect)
    
*   [图解 HTTPS RSA 算法握手流程](https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247487650&idx=1&sn=dfee83f6773a589c775ccd6f40491289&scene=21#wechat_redirect)
    
*   [图解 HTTPS ECDHE 算法握手流程](https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247487977&idx=1&sn=f7cf6cd6b9c2f77f50eaf0fa73a43da2&scene=21#wechat_redirect)
    
*   [字节一面：HTTPS 一定安全可靠吗？](https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247518909&idx=1&sn=a97a218899abf5310385d84e01d30851&scene=21#wechat_redirect)
    
*   [字节一面：HTTPS 会加密 URL 吗？](https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247518634&idx=1&sn=b1d36ef60ed5c53d89b9b2f953d149f8&scene=21#wechat_redirect)