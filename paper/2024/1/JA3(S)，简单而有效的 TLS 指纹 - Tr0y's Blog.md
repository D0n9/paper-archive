# JA3(S)，简单而有效的 TLS 指纹 - Tr0y's Blog
JA3(S)，简单而有效的 TLS 指纹。这是一篇很简单的介绍文章，附带一丢丢技术细节。

[](#背景)背景
---------

最近在看 `Suricata`，一个开源的 NIDS。Suricata 自带了很多的规则，然后里面有些比较特殊的规则引起了我的注意：  

```php

```

经过 “美化”，去掉没啥用的信息之后，再加点注释，如下：

```python

```

这个规则最主要的就是这个 `ja3s.hash` 了。

[](#ja3-与-ja3s)ja3 与 ja3s
-------------------------

`ja3(s)` 是为特定客户端与服务器之间的加密通信提供了具有更高的识别度的指纹，说白了就是 TLS 协商的指纹。那么这个有什么用呢？

例如，现在的 C2 服务器与恶意客户端之间的通信往往都是套上 TLS 的，将其流量隐藏在噪声中来躲避 IDS/IPS，这样光从 ip/域名这个维度去检测难免会漏掉一些。如果我们掌握了 C2 服务器与恶意客户端的 ja3(s)，即使恶意流量被加密且不知道 C2 服务器的 IP 地址或域名，我们仍然可以通过 TLS 指纹来识别恶意客户端和服务器之间的 TLS 协商。

那么难道 ja3(s) 不能改变吗？当然是可以的，但是会提高成本：改个 ip 或者域名，比修改客户端方便多了吧？

### [](#原理)原理

回想一下我们在初三就学过的知识，客户端会在 TCP 3 次握手后发送 TLS 客户端的 Hello 数据包，而程（da）序（hei）员（ke）在写客户端的时候其实就已经确定了这个数据包里的一些特定内容会是什么样的，我们只需要将这些特定的内容提取出来，排好队，进行 hash，就是客户端的 TLS 指纹，即 `ja3`。

服务器收到 Hello 之后，会构造 TLS Server Hello 数据包进行响应。同样，这个响应数据包中的一些特定内容，也是由服务器应用程序决定的，这就是 `ja3s`。

当然，我们初三就知道，上述通信过程的是以明文的方式传输的，所以不存在`没法解出 TLS => 没法计算指纹`这样的套娃情况。

### [](#计算-ja3)计算 ja3

刚才说了，特定内容，那么这个特定内容到底是哪几个字段呢？一共有 5 个：`ClientHello 的版本`、`可接受的加密算法`、`扩展列表中的每一个 type 值`、`支持的椭圆曲线`和`支持的椭圆曲线格式`。然后，用`,`来分隔各个字段、用使用`-`来分隔各个字段中的各个值（十进制哦），将这些值串联在一起之后，计算 `MD5`，就是一个 ja3 了。注意，如果没有某个字段，则这些字段的值为空（连接用的逗号别忘了）。

举个例子，curl 一下百度：  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/3879df60-4f11-4c46-9666-3ac217313e4e.png?raw=true)
](https://rzx1szyykpugqc-1252075454.piccd.myqcloud.com/ja3/20200628022756065.png!blog)

这样算下来，指纹应该是：  

```apache

```

经过 md5 就是 `e6573e91e6eb777c0933c5b8f97f10cd`。这就是我的 curl 的 ja3 啦。

### [](#计算-ja3s)计算 ja3s

ja3s 与 ja3 类似，提取 Server Hello 数据包中的：`Server Hello 版本`、`可接受的加密算法`和`扩展列表中的每一个 type 值`。然后同样用`,`来分隔各个字段、用使用`-`来分隔各个字段中的各个值（十进制哦），将这些值串联在一起之后，计算 `MD5`，就是一个 ja3s 了。

[](#一些杂谈)一些杂谈
-------------

> 为什么要用 md5？

md5 的确有点过时了。ja3(s) 开发者（John Althouse）给出的理由是他希望 ja3(s) 在任意硬件上都可以使用：“...即使是最古老的 NetScreen 防火墙也可以支持大批量的MD5计算，所以，我们还是选择了MD5算法...此外，考虑到有限的数据集，这里根本就不需要考虑哈希值的碰撞问题...”。我基本上是赞同他的看法的，用更好的 hash 可以，但是没必要。

> ja3(s) 的误报率如何？

说实话，一般只有高度定制化的恶意软件会自己去实现 TLS，也是在这种情况下，ja3 指纹很可能对该恶意软件来说是唯一的。但是现在研发一般都会用第三方的库，不管是诸如 Python 的官方模块还是 win 下的组件，如果是这种情况，那么 ja3 会重复，误报率很高。这其实就是为什么要用 ja3s。

John Althouse 也举了个例子，翻译如下：

“...例如，MetaSploit 的 Meterpreter 和 CobaltStrike 的 Beacon 都使用 Windows 套接字来启动 TLS 通信。在 Windows 10 上，`JA3=72a589da586844d7f0818ce684948eea`（指定 IP 地址），`JA3=a0e9f5d64349fb13191bc781f81f42e1`（指定域名）。由于 Windows 上的其他普普通通的应用程序也使用相同的套接字，因此，我们很难识别其中的恶意通信。但是，Kali Linux 上的 C2 服务器对该客户端应用程序的响应方式与 Internet 上的普通服务器对该套接字的响应方式相比来说是独一无二的。因此，如果结合 ja3+ja3s，就能够识别这种恶意通信，而不用考虑目的地 IP、域名或证书等细节信息...”

总而言之，ja3 不是非常准确，所以要用 ja3s；ja3+ja3s 依旧不会非常准确，但是可以丰富我们检测威胁的维度，增加了攻击者的攻击成本，事实上现在很多 nids 都集成了 ja3(s) 的提取与匹配。

[](#最后)最后
---------

1.  这是 John Althouse 的 repo：https://github.com/salesforce/ja3 ，里面有一些工具还有介绍
2.  👆的工具贼不好用，所以我自己写了一个：https://github.com/Macr0phag3/ja3box
3.  这是 John Althouse 的 ja3(s) 的文章：https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967

现在越来越多网址在用这个做反爬。有一个橘友问我有没有什么办法伪造 ja3？

其实是有的。以 Python 为例，requests 依赖是其实是对 urllib 的一个封装，https 底层还是依赖的 OpenSSL。我尝试找过 OpenSSL 有没有提供修改字段的方法，并没有发现。不过 cipher 的算法倒是可以直接修改，`urllib3.util.ssl_.DEFAULT_CIPHERS = 'EECDH+AESGCMEDH+AESGCM'` 即可。这样生成的 ja3 就不是 requests 默认的了，但只能骗过不是太高明的反爬机制。

同时，我尝试过用 scapy 写一个代理，然后劫持 curl/requests 发出的请求，篡改 ClientHello 包里的相关字段，然后再发出，达到伪造的目的。一番尝试之后，我发现这是不可行的，OpenSSL 会校验 extension，如果和自己发出的不一致，则会报错：`OpenSSL: error:141B30D9:SSL routines:tls_collect_extensions:unsolicited extension`。我的理解是这样的：ServerHello 的 Extension 是作为 ClientHello 的 Extension 的回应，前者应当是后者的子集。如果 Client 发现 ServerHello 中有一个扩展类型不存在与 ClientHello 中，那么它必须用一个 unsupported_extension alert 消息来丢弃此握手响应。显然 OpenSSL 对此是严格遵守的，这一点虽然让人很遗憾，但是我作为安全人员，还是很赞赏这种坚守的。

所以如果准备用劫持的方式去伪造 ja3，这种依赖 OpenSSL 的库（比如 requests、curl）是不可行的，你只能祈祷 Server 别瞎返回，最好都不要响应任何 Extension。要么就去魔改 OpenSSL 以及 Python 调用 OpenSSL 的那个 .so，非常麻烦，反正我是懒得搞。

最后我找到几个可以用来伪造 ja3 的库：

*   [`ja3transport`](https://github.com/CUCyber/ja3transport)，golang 的库，这个的原理也是劫持 ClientHello 篡改，我认为不太靠谱。
*   [`curl-impersonate`](https://github.com/lwthiker/curl-impersonate)，魔改的 curl，支持修改 ciphers 以及 curves。至于 extensions，我简单看了下作者的文章，他是通过使用与浏览器相同的 SSL 组件来模拟浏览器的 extensions，例如 Chrome 用的是 `BoringSSL`，FireFox 用的是 `NSS`，这个办法很聪明，在大多数场景下这个已经可以满足绕过的需要了，不过它就没办法模拟任意的 ja3 了。基于这个代码，还有一个 Python 版本的 [`curl_cffi`](https://github.com/yifeikong/curl_cffi/tree/main)
*   [`CycleTLS`](https://github.com/Danny-Dasilva/CycleTLS)，有 golang 和 nodejs 的库，这个看代码是自己实现了 TLS 握手，实在是令人佩服。为了兼容 HTTP2 以及各种复杂的 TLS 参数，这个库目前还在艰难地维护当中， 不过只要不是特殊情况应该还是可以使用的。

