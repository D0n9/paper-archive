# SSRF vulnerabilities caused by SNI proxy misconfigurations | Invicti --- SNI 代理错误配置导致的 SSRF 漏洞 |因维克蒂
A typical task in complex web applications is routing requests to different backend servers to perform load balancing. Most often, a reverse proxy is used for this. Such reverse proxies work at the application level (over HTTP), and requests are routed based on the value of the `Host` header (`:authority` for HTTP/2) or parts of the path.  
复杂 Web 应用程序中的一个典型任务是将请求路由到不同的后端服务器以执行负载平衡。大多数情况下，反向代理用于此。此类反向代理在应用程序级别（通过 HTTP）工作，并且请求根据 `Host` 标头（对于 HTTP/2 为 `:authority` ）或部分路径的值进行路由。

One typical misconfiguration is when the reverse proxy directly uses this information as the backend address. This can lead to [server-side request forgery (SSRF)](http://invicti.com/learn/server-side-request-forgery-ssrf/) vulnerabilities that allow attackers to access servers behind the reverse proxy and, for example, steal information from AWS metadata. I decided to investigate similar attacks on proxy setups operating at other levels/protocols – in particular, SNI proxies.  
一种典型的错误配置是反向代理直接使用此信息作为后端地址。这可能导致服务器端请求伪造 (SSRF) 漏洞，允许攻击者访问反向代理后面的服务器，例如，从 AWS 元数据中窃取信息。我决定调查对在其他级别/协议上运行的代理设置的类似攻击——特别是 SNI 代理。

What is TLS SNI?什么是 TLS SNI？
----------------------------

Server Name Indication (SNI) is an extension of the TLS protocol that provides the foundation of HTTPS. When a browser wants to establish a secure connection to a server, it initiates a TLS handshake by sending a `ClientHello` message. This message may contain an SNI extension field that includes the server domain name. In its `ServerHello` message, the server can then return a certificate appropriate for the specified server name. The typical use case for this is when there are multiple virtual hosts behind one IP address.  
服务器名称指示 (SNI) 是 TLS 协议的扩展，它提供了 HTTPS 的基础。当浏览器想要与服务器建立安全连接时，它会通过发送 `ClientHello` 消息来启动 TLS 握手。此消息可能包含一个包含服务器域名的 SNI 扩展字段。在其 `ServerHello` 消息中，服务器随后可以返回适合指定服务器名称的证书。典型的用例是当一个 IP 地址后面有多个虚拟主机时。

What is an SNI proxy?什么是 SNI 代理？
--------------------------------

When a reverse proxy (more correctly, a load balancer) uses a value from the SNI field to select a specific backend server, we have an SNI proxy. With the widespread use of TLS and HTTPS in particular, this approach is becoming more popular. (Note that another meaning of SNI proxy refers to the use of such proxies to bypass censorship in some countries.)  
当反向代理（更准确地说，负载均衡器）使用 SNI 字段中的值来选择特定的后端服务器时，我们就有了一个 SNI 代理。特别是随着 TLS 和 HTTPS 的广泛使用，这种方法变得越来越流行。 （请注意，SNI 代理的另一个含义是指使用此类代理来绕过某些国家/地区的审查。）

There are two main options for running an SNI proxy: with or without SSL termination. In both cases, the SNI proxy uses the SNI field value to select an appropriate backend. When running with SSL termination, the TLS connection is established with the SNI proxy, and then the proxy forwards the decrypted traffic to the backend. In the second case, the SNI proxy forwards the entire data stream, really working more like a TCP proxy.  
运行 SNI 代理有两个主要选项：有或没有 SSL 终止。在这两种情况下，SNI 代理都使用 SNI 字段值来选择合适的后端。使用 SSL 终止运行时，会与 SNI 代理建立 TLS 连接，然后代理将解密后的流量转发到后端。在第二种情况下，SNI 代理转发整个数据流，实际上更像 TCP 代理。

A typical SNI proxy configuration  
典型的 SNI 代理配置
------------------------------------------------

Many reverse proxies/load balancers support SNI proxy configurations, including Nginx, Haproxy, Envoy, ATS, and others. It seems you can even use an [SNI proxy in Kubernetes](https://gist.github.com/kekru/c09dbab5e78bf76402966b13fa72b9d2#choose-upstream-based-on-domain-pattern).  
许多反向代理/负载均衡器支持 SNI 代理配置，包括 Nginx、Haproxy、Envoy、ATS 等。看来您甚至可以在 Kubernetes 中使用 SNI 代理。

To give an example for Nginx, the simplest configuration would look as follows (note that this requires the Nginx modules `ngx_stream_core_module` and `ngx_stream_ssl_preread_module` to work):  
以 Nginx 为例，最简单的配置如下所示（请注意，这需要 Nginx 模块 `ngx_stream_core_module` 和 `ngx_stream_ssl_preread_module` 才能工作）：

Here, we configure a server (TCP proxy) called `stream` and enable SNI access using `ssl_preread on`. Depending on the SNI field value (in `$ssl_preread_server_name`), Nginx will route the whole TLS connection either to `backend1` or `backend2`.  
在这里，我们配置一个名为 `stream` 的服务器（TCP 代理）并使用 `ssl_preread on` 启用 SNI 访问。根据 SNI 字段值（在 `$ssl_preread_server_name` 中），Nginx 会将整个 TLS 连接路由到 `backend1` 或 `backend2` 。

SNI proxy misconfigurations leading to SSRF  
SNI 代理错误配置导致 SSRF
---------------------------------------------------------------

The simplest misconfiguration that would allow you to connect to an arbitrary backend would look something like this:  
允许您连接到任意后端的最简单的错误配置如下所示：

Here, the SNI field value is used directly as the address of the backend.  
这里直接使用SNI字段值作为后端的地址。

With this insecure configuration, we can exploit the SSRF vulnerability simply by specifying the desired IP or domain name in the SNI field. For example, the following command would force Nginx to connect to _internal.host.com_:  
通过这种不安全的配置，我们可以通过在 SNI 字段中指定所需的 IP 或域名来利用 SSRF 漏洞。例如，以下命令将强制 Nginx 连接到 internal.host.com：

```
openssl s_client -connect [](http://lab.io:10003/) target.com:443 -servername "internal.host.com" -crlf
```

In general, according to [RFC 6066](https://www.rfc-editor.org/rfc/rfc6066#page-6), IP addresses should _not_ be used in SNI values, but in practice, we can still use them. What’s more, we can even send arbitrary symbols in this field, including null bytes, which can be useful for exploitation. As you can see below, the server name can be changed to an arbitrary string. Though for this specific Nginx configuration, unfortunately, I did not find a way to change the backend port:  
一般来说，根据 RFC 6066 ，IP 地址不应该用在 SNI 值中，但在实践中，我们仍然可以使用它们。更重要的是，我们甚至可以在这个字段中发送任意符号，包括空字节，这对漏洞利用很有用。如下所示，服务器名称可以更改为任意字符串。虽然对于这个特定的 Nginx 配置，不幸的是，我没有找到改变后端端口的方法：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/8b0901be-91c1-4169-9a9f-4ba2075b4014.png?raw=true)
](https://cdn.invicti.com/app/uploads/2022/12/29154609/image-23-1024x159.png)

Another class of vulnerable configurations is similar to typical HTTP reverse proxy misconfigurations and involves mistakes in the regular expression (regex). In this example, traffic is forwarded to the backend if the name provided via SNI matches the regex:  
另一类易受攻击的配置类似于典型的 HTTP 反向代理配置错误，涉及正则表达式 (regex) 中的错误。在此示例中，如果通过 SNI 提供的名称与正则表达式匹配，则流量将转发到后端：

This regex is incorrect because the first period character in `www.example.com` is not escaped, and the expression is missing the `$` terminator at the end. The resulting regex matches not only _www.example.com_ but also URLs like _www.example.com.attacker.com_ or _wwwAexample.com_. As a result, we can perform SSRF and connect to an arbitrary backend. While we can’t use the IP address directly here, we can bypass this restriction simply by telling our DNS server that _www.example.com.attacker.com_ should resolve to 127.0.0.1.  
此正则表达式不正确，因为 `www.example.com` 中的第一个句点字符未转义，并且表达式末尾缺少 `$` 终止符。生成的正则表达式不仅匹配 www.example.com，还匹配 www.example.com.attacker.com 或 wwwAexample.com 等 URL。因此，我们可以执行 SSRF 并连接到任意后端。虽然我们不能在这里直接使用 IP 地址，但我们可以简单地通过告诉我们的 DNS 服务器 www.example.com.attacker.com 应该解析为 127.0.0.1 来绕过这个限制。

Potential directions for SNI proxy research and abuse  
SNI 代理研究和滥用的潜在方向
------------------------------------------------------------------------

In a 2016 [article about scanning IPv4 for open SNI proxies](https://www.bamsoftware.com/computers/sniproxy/), researchers managed to find about 2500 servers with a fairly basic testing approach. While this number may seem low, SNI proxy configurations have become more popular since 2016 and are widely supported, as evidenced even by a quick search of GitHub.   
在 2016 年一篇关于扫描 IPv4 以寻找开放 SNI 代理的文章中，研究人员设法找到了大约 2500 台服务器，并采用了相当基本的测试方法。虽然这个数字可能看起来很低，但 SNI 代理配置自 2016 年以来变得越来越流行并得到广泛支持，甚至可以通过快速搜索 GitHub 来证明这一点。

As a direction for further research, I can suggest a couple of things to think about for configurations without TLS termination. An SNI proxy checks only the first `ClientHello` message and then proxies all the subsequent traffic, even if it’s not correct TLS messages. Also, while the RFC specifies that you can only have one SNI field, in practice, we can send multiple different names ([TLS-Attacker](https://github.com/tls-attacker/TLS-Attacker) is a handy tool here). Because Nginx only checks the first value, there could (theoretically) be an avenue to gain some additional access if a backend accepts such a `ClientHello` message but then uses the second SNI value.  
作为进一步研究的方向，对于没有 TLS 终止的配置，我可以提出一些需要考虑的事项。 SNI 代理仅检查第一个 `ClientHello` 消息，然后代理所有后续流量，即使它不是正确的 TLS 消息。此外，虽然 RFC 指定你只能有一个 SNI 字段，但实际上，我们可以发送多个不同的名称（TLS-Attacker 是一个方便的工具）。因为 Nginx 只检查第一个值，所以如果后端接受这样的 `ClientHello` 消息但随后使用第二个 SNI 值，那么（理论上）可能有一条途径获得一些额外的访问权限。

Avoiding SNI proxy vulnerabilities  
避免 SNI 代理漏洞
------------------------------------------------

Whenever you configure a reverse proxy, you should be aware that any misconfigurations may potentially lead to SSRF vulnerabilities that expose backend systems to attack. The same applies to SNI proxies, especially as they are gaining popularity in large-scale production systems. In general, to avoid vulnerabilities when configuring a reverse proxy, you should understand what data could be controlled by an attacker and avoid using it directly in an insecure way.  
每当您配置反向代理时，您都应该意识到任何错误配置都可能导致 SSRF 漏洞，从而使后端系统受到攻击。这同样适用于 SNI 代理，尤其是当它们在大型生产系统中越来越受欢迎时。一般来说，为了避免在配置反向代理时出现漏洞，您应该了解攻击者可以控制哪些数据，并避免以不安全的方式直接使用它。