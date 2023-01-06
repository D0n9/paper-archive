# 17岁高中生详述如何攻破 GitHub Private Pages 并获$3.5万赏金
本文作者是一名17岁的美国高中生，在本文中他详述了自己和16岁的好友@ginkoid如何发现并报告 GitHub Private Pages 中的漏洞，以及最后获得3.5万美元的故事。文章编译如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/f1cee58d-4a34-4c35-86b8-6e5d9dc6d60d.png?raw=true)

这个漏洞是我和@ginkoid一起发现并报告的。

这是我在 HackerOne 平台上报告并获得赏金的第一个漏洞。3.5万美元的赏金是我迄今为止从 HackerOne 平台上获得的最高赏金（而且我觉得可能是 GitHub 迄今为止颁发的最高赏金）。

很多漏洞看似是运气和直觉的产物。在本文中我将说明自己发现该漏洞的思考过程。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/3b152dbc-a13e-4d00-a3ea-440e36ef1de2.png?raw=true)

背景

新冠肺炎疫情在我上高一春季学期时爆发。网课闲暇之余无事可做，于是我开始投入猎洞活动。

这个漏洞是 GitHub 非公开页面非公开漏洞奖励计划的一部分。具体而言，GitHub 提供了两种 CTF 赏金：

*   $1万：在无需用户交互的情况下读取 flag.private-org.github.io 处的 flag。如果该 flag 师从 private-org 组织机构之外的账户读取的，则额外给予 $5000奖励。
    
*   $5000：在用户交互情况下读取 flag.private-org.github.io 处的 flag。
    

认证流

由于 GitHub 页面托管在单独的 github.io 域名，因此 github.com 认证 cookie 并未发送到非公开页面服务器上。如此，非公开页面认证无法在不和 github.com 集成的情况下判断用户身份。因此，GitHub 创造了一个自定义认证流（从而可能引入 bug!）

在本报告发布之时，GitHub 的非公开页面认证流如下：

更具体来说，访问非公开页面时，服务器检查是否存在 \_Host-gh\_pages_token cookie。如该 cookie 并未或者并未正确设置，则非公开页面服务器将重定向至 https://github.com/login。这个初始重定向也在 \_\_Host-gh\_pages_session cookie 中存储了一个随机数。

注意，该 cookie 使用了 _Host-cookie 前缀，从理论上讲，作为额外的纵深防御措施，它会阻止 JavaScript 对非主机（父）域名进行设置。

/login 之后重定向至 /pages/auth?nonce=&page_id=&path=。该端点随后生成临时认证 cookie 并将其传递给 token 参数中的 https://pages-auth.github.com/redirect。nonce、page_id 和 path 都以类似方式转发。

/redirect 址会转发到 https://repo.org.github.io/__/auth。之后，该最终端点在 repo.org.github.io 域名、\_Host-gh\_pages_token 和 \_Host-gh\_pages_id 上设置认证 cookie。它还会验证之前所设置 \_Host-gh\_pages_session 的 nonce。

在整个认证流中，像原始请求路径和页面 id 等信息被分别存储在查询参数 path 和 page_id 中。该 nonce 也在 nonce 参数中传递。

尽管对该认证流可能已被做出了微调，不过整体理念是一致的。

利用

**CRLF** **返回**

第一个漏洞是位于 https://repo.org.github.io/__/auth 上 page_id 参数中的CRLF 注入。

可能发现漏洞的最佳方式是静观其变。调查认证流时，我发现 page_id 解析似乎忽略了空格。更耐人寻味的是，它还将该参数直接渲染到 Set-Cookie 标头。

例如，page_id=12345%20 将会给出：

```sql
Set-Cookie: __Host-gh_pages_id=13212257 ; Secure; HttpOnly; path=/
```

伪代码如下：

```makefile
page_id = query.page_id

do_page_lookup(to_int(page_id))
set_page_id_cookie(page_id)
```

换句话说，page_id 转换为整数，但同时直接渲染到 Set-Cookie 标头。

问题就在于，我们无法直接渲染任何文本。尽管存在一个经典的 CRLF 注入，但任意非空格字符导致整数解析崩溃。我们可发送 page_id=12345%0d%0a%0d%0a 破解该认知流，但除了一个响应之外并未产生任何直接影响。

```cs
; Secure; HttpOnly; path=/
Cache-Control: private
Location: https:
X-GLB-L
```

另外提一句，由于 Location: 标头附加在 Set-Cookie 标头之后，我们的响应将 Location 推出所发送的 HTTP 标头外。即使这是一个302重定向，Location 标头将被忽视而主题内容被渲染。

**从零登顶**

对 GitHub Enterprise 稍微进行了遍历（获得源代码访问权限）后，我认为非公开页面服务器在 openresty nginx 中执行。它相对比较底层，因此可能和 null 字节有关。要不再试试一次？

结果，附加 null 字节导致整数解析。换句话说，我们可以使用如下 payload：

```javascript
"?page_id=" + encodeURIComponent("\r\n\r\n\x00<script>alert(origin)</script>")
```

结果，我们获得了 XSS！

注意，如果标头中存在一个 null 字节，则响应被拒绝。因此，该 null 字节必须位于主体开头（意味着我们无法执行标头注入攻击）。

此时，我们实现了在非公开页面域名中执行任意 JavaScript 的目的。唯一问题是，我们需要绕过该 nonce。虽然 page_id 和path 参数是已知的，但 nonce 阻止我们通过被投毒的 page_id 向受害者发送认证流。

我们需要控制或者预知该 nonce。

**绕过 Nonce**

首先观察到的是相同组织机构中的同级非公开页面能够互相设置 cookie。这是因为 *.github.io 并不在 Public Suffix List 上。因此，在 private-org.github.io 上设置 cookie 将被传递到 private-page.private-org.github.io。

如果我们能够绕过 _Host- 前缀防护措施，那么就能绕过 nonce。只需在将被传递的同级页面中设立虚假 nonce 即可。幸运的是，该前缀并未在所有浏览器上执行。

看似只有 IE 浏览器易受该绕过影响。

攻击 nonce 本身会如何？它似乎是以安全方式生成的，而且说实话，密码学并非我的强项。看来我们不可能为 nonce 生成所使用的熵找到绕过了。那么我们该如何控制nonce？

又回到了源头……或 RFCs。我最后想到一个好办法：cookie 是如何规范化的。具体讲，cookie 的大写情况是如何被处理的？__HOST- 和 __Host- 一样吗？

在浏览器上，我们很容易就能证实它们的处理方式不同。

```javascript
document.cookie = "__HOST-Test=1"; 

document.cookie = "__Host-Test=1"; 
```

结果显示，GitHub 非公开页面服务器在解析 cookie 时忽视了大写问题。我们达成前缀绕过！截止目前，可以给出完整 XSS 的 PoC 了！

```perl
<script>

const id = location.search.substring("?id=".length)

document.cookie = "__HOST-gh_pages_session=dea8c624-468f-4c5b-a4e6-9a32fe6b9b15; domain=.private-org.github.io";

location = "https://github.com/pages/auth?nonce=dea8c624-468f-4c5b-a4e6-9a32fe6b9b15&page_id=" + id + "%0d%0a%0d%0a%00<script>alert(origin)%3c%2fscript>&path=Lw";

</script>
```

这个漏洞本身就值5000美元的奖金。但我想看下是否可以走得更远。

**缓存投毒**

另外一个设计缺陷是，/__/auth? 端点上的响应似乎单独缓存在被解析的 page_id 的整数值上。这样做本身是无害的；该端点设置的令牌被归为非公开页面且不存在其它权限。

同时，这个设计实践存在问题。如果令牌后续获得其它权限，那么它就会成为潜在的安全问题来源。

无论如何，这种缓存行为使我们能够轻易提升攻击的严重程度。由于攻击针对的是解析后的整数值，因此通过 XSS payload 实施缓存投毒可影响甚至未与恶意 payload 交互的其它用户。

如上所示：

攻击者控制unprivileged.org.github.io 并想访问privileged.org.github.io。他们首先投毒 unprivileged.org.github.io 的认证流，XSS payload 被缓存。

现在，当权限用户访问 unprivileged.org.github.io 时，他们会在该域名经历 XSS 攻击。由于 cookie 可在共享父域名 org.github.io 上设置，因此攻击者当前可执行对 privileged.org.github.io 的攻击。

结果使对非公开页面具有读权限的任意攻击者永久性地对该页面的认证流进行投毒。

**公开-非公开页面**

为获得1.5万美元的赏金，我们需要以不在该组织机构内的用户账户执行该攻击。幸运的是，我们可以滥用另外一种看似不相关的错误配置。输入“public-private pages（公开-非公开页面）”。

非公开页面中可能存在的错误配置情况可使公开仓库也具有自己的“非公开”页面。在遍历正常的认证周期时，这些“非公开”页面是对所有人公开可见。如某组织机构想要拥有其中某一个公开-非公开页面，则拥有 GitHub 账户的任何用户将具有“读取权限”。

示例如下：  

当非公开页面仓库更改为公开时就会发生这种情况。该场景说服力较强。例如，某组织机构最初可能创建具有相应非公开页面的非公开仓库。之后，该组织机构可能决定开源该项目，将仓库状态更改为“公开”状态。

结合来看，低权限的外部用户可以从“公开-非公开”页面跳转，攻陷内部非公开页面的认证流。

总体来看，我们就拥有了一个良好的 PoC，演示了外部攻击者如何通过内部员工跳转到其它非公开页面。

到此，我们确保获得了最多的 CTF 赏金。

从现在开始，我们可以通过 AppCache 或其它技术实现持久性，感兴趣的读者可以一试。

总结

这类漏洞产生的几率是百万分之一。很多组件必须以正确的方式出现，就像穿针引线一样。同时，我认为要找到这样的漏洞必须同时具有一定的直觉和技术。

不管怎样，从 GitHub找到这样一种相对少见的漏洞（CRLF注入）非常酷。尽管多数代码是用 Ruby 编写的，但某些组件如非公开页面认证并非通过 Ruby 编写且可能易受更底层的攻击影响。总体而言，只要存在复杂的交互，就可能会存在 bug。

总体来看，该漏洞属于“高危”级别中的高位，基本赏金是2万美元。加上 CTF 赏金，我们共获3.5万美元的赏金。

时间线

*   2020/05/21——向 HackerOne 平台上的 GitHub 非公开计划报告
    
*   2020/06/20——GitHub 解决漏洞问题并颁发奖金
    
*   2021/04/03——博客文章发布