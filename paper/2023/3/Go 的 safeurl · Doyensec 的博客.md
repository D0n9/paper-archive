# Go 的 safeurl · Doyensec 的博客
2022 年 12 月 13 日 - 由 Alessandro Cotto 和 Viktor Chuchurski 发布

**您是否需要 Go HTTP 库来保护您的应用程序免受 SSRF 攻击？**如果是这样，请尝试[safeurl](https://github.com/doyensec/safeurl)。它是 Go 客户端的在线替代品`net/http`。

Go Web 应用程序中不再有 SSRF
--------------------

在构建 Web 应用程序时，向内部微服务甚至外部第三方服务发出 HTTP 请求并不少见。每当用户提供 URL 时，确保正确缓解_服务器端请求伪造(SSRF) 漏洞非常重要。_正如 PortSwigger 的[网络安全学院](https://portswigger.net/web-security/ssrf)页面中雄辩地描述的那样，SSRF 是一种网络安全漏洞，允许攻击者诱导服务器端应用程序向非预期位置发出请求。

虽然存在多种编程语言的库可以缓解 SSRF，但 Go 没有易于使用的解决方案。到目前为止！

`safeurl for Go`是一个内置 SSRF 和 DNS 重新绑定保护的库，可以轻松替换 Go 的默认`net/http`客户端。解析、验证和发出请求的所有繁重工作都由库完成。该库以最少的配置开箱即用，同时为开发人员提供他们可能需要的自定义和过滤选项。开发人员不应为解决应用程序安全问题而苦苦挣扎，而应专注于为客户提供高质量的功能。

该库的灵感分别来自[Jack Whitton](https://twitter.com/fin1te)和[Include Security](https://blog.includesecurity.com/2016/08/introducing-safeurl-a-set-of-ssrf-protection-libraries/)的 SafeCURL 和 SafeURL 。由于不存在用于 Go 的 SafeURL，Doyensec 将其提供给社区。

提供什么`safeurl`？
--------------

通过[最少的配置](#configuration)，库可以防止对内部、私有或保留 IP 地址的未经授权的请求。所有 HTTP 连接都根据白名单和黑名单进行验证。默认情况下，库会阻止所有流向专用或保留 IP 地址的流量，如[RFC1918](https://datatracker.ietf.org/doc/html/rfc1918)所定义。此行为可以通过 的`safeurl`客户端配置进行更新。图书馆将优先考虑允许的项目，无论是主机名、IP 地址还是端口。通常，白名单是构建安全系统的推荐方式。事实上，明确设置允许的目的地更容易（也更安全），而不是在当今不断扩大的威胁环境中必须处理更新阻止列表。

安装
--

通过简单地添加到您的项目文件中，将模块包含`safeurl`在您的 Go 程序中。`github.com/doyensec/safeurl``go.mod`

```
go get -u github.com/doyensec/safeurl 
```

用法
--

该`safeurl.Client`库提供的 可以用作 Go 原生`net/http.Client`.

以下代码片段显示了一个使用该`safeurl`库的简单 Go 程序：

```
import (
    "fmt"
    "github.com/doyensec/safeurl"
)
func main() {
    config := safeurl.GetConfigBuilder().
        Build()
    client := safeurl.Client(config)
    resp, err := client.Get("https://example.com")
    if err != nil {
        fmt.Errorf("request return error: %v", err)
    }
    // read response body
} 
```

最小的库配置类似于：

```
config := GetConfigBuilder().Build() 
```

使用此配置，您将获得：

*   仅允许端口 80 和 443 的流量
*   允许使用 HTTP 或 HTTPS 协议的流量
*   阻止到私有 IP 地址的流量
*   阻止到任何地址的 IPv6 流量
*   缓解 DNS 重新绑定攻击

配置
--

用于`safeurl.Config`自定义`safeurl.Client`. 该配置可用于设置以下内容：

```
AllowedPorts            - list of ports the application can connect to
AllowedSchemes          - list of schemas the application can use
AllowedHosts            - list of hosts the application is allowed to communicate with
BlockedIPs              - list of IP addresses the application is not allowed to connect to
AllowedIPs              - list of IP addresses the application is allowed to connect to
AllowedCIDR             - list of CIDR range the application is allowed to connect to
BlockedCIDR             - list of CIDR range the application is not allowed to connect to
IsIPv6Enabled           - specifies whether communication through IPv6 is enabled
AllowSendingCredentials - specifies whether HTTP credentials should be sent
IsDebugLoggingEnabled   - enables debug logs 
```

作为 Go 原生的包装器`net/http.Client`，该库还允许您配置其他标准​​设置，例如 HTTP 重定向、cookie jar 设置和请求超时。有关生产环境的建议配置的更多信息，请参阅[官方文档。](https://pkg.go.dev/net/http#Client)

配置示例
----

为了展示`safeurl.Client`它的多功能性，让我们向您展示一些配置示例。

可以只允许一个**模式**：

```
GetConfigBuilder().
    SetAllowedSchemes("http").
    Build() 
```

或者配置一个或多个允许的**端口**：

```
// This enables only port 8080. All others are blocked (80, 443 are blocked too)
GetConfigBuilder().
    SetAllowedPorts(8080).
    Build()
// This enables only port 8080, 443, 80
GetConfigBuilder().
    SetAllowedPorts(8080, 80, 443). 
    Build()
// **Incorrect.** This configuration will allow traffic to the last allowed port (443), and overwrite any that was set before
GetConfigBuilder().
    SetAllowedPorts(8080).
    SetAllowedPorts(80).
    SetAllowedPorts(443).
    Build() 
```

此配置只允许流量流向一台主机，`example.com`在这种情况下：

```
GetConfigBuilder().
    SetAllowedHosts("example.com").
    Build() 
```

此外，您可以阻止特定的 IP（IPv4 或 IPv6）：

```
GetConfigBuilder().
    SetBlockedIPs("1.2.3.4").
    Build() 
```

请注意，对于之前的配置，`safeurl.Client`除了属于内部、专用或保留网络的所有 IP 之外，还将阻止 IP 1.2.3.4。

如果您希望允许流量到客户端默认阻止的 IP 地址，您可以使用以下配置：

```
GetConfigBuilder().
    SetAllowedIPs("10.10.100.101").
    Build() 
```

也可以允许或阻止完整的 CIDR 范围而不是单个 IP：

```
GetConfigBuilder().
    EnableIPv6(true).
    SetBlockedIPsCIDR("34.210.62.0/25", "216.239.34.0/25", "2001:4860:4860::8888/32").
    Build() 
```

DNS 重新绑定缓解
----------

由于两个（或多个）连续 HTTP 请求之间的 DNS 响应不匹配，可能会发生 DNS 重新绑定攻击。该漏洞是一个典型的TOCTOU问题。在检查时 (TOC)，IP 指向允许的目的地。但是，在使用时 (TOU)，它将指向一个完全不同的 IP 地址。

DNS 重新绑定保护是`safeurl`通过对将用于发出 HTTP 请求的实际 IP 地址执行允许/阻止列表验证来实现的。这是通过使用 Go 的`net/dialer`包和提供的`Control`钩子来实现的。如官方文档所述：

```
// If Control is not nil, it is called after creating the network
// connection but before actually dialing.
Control func(network, address string, c syscall.RawConn) error 
```

在我们的`safeurl`实现中，IP 验证发生在挂钩_内_`Control`。以下片段显示了正在执行的一些检查。如果全部通过，则发生 HTTP 拨号。如果检查失败，则丢弃 HTTP 请求。

```
func buildRunFunc(wc *WrappedClient) func(network, address string, c syscall.RawConn) error {
return func(network, address string, _ syscall.RawConn) error {
	// [...]
	if wc.config.AllowedIPs == nil && isIPBlocked(ip, wc.config.BlockedIPs) {
		wc.log(fmt.Sprintf("ip: %v found in blocklist", ip))
		return &AllowedIPError{ip: ip.String()}
	}
	if !isIPAllowed(ip, wc.config.AllowedIPs) && isIPBlocked(ip, wc.config.BlockedIPs) {
		wc.log(fmt.Sprintf("ip: %v not found in allowlist", ip))
		return &AllowedIPError{ip: ip.String()}
	}
	return nil
}
} 
```

我们在库开发过程中进行了广泛的测试。但是，我们很乐意让其他人选择我们的实施。

> “只要有足够的眼睛，所有的错误都是肤浅的”。希望。

连接到[http://164.92.85.153/](http://164.92.85.153/)并尝试捕获此内部（和未授权）URL 上托管的标志：`http://164.92.85.153/flag`

**挑战于 01/13/2023 关闭。** 您始终可以使用下面的代码片段在本地运行挑战。

这是挑战端点的源代码，具体`safeurl`配置如下：

```
func main() {
	cfg := safeurl.GetConfigBuilder().
		SetBlockedIPs("164.92.85.153").
		SetAllowedPorts(80, 443).
		Build()
	client := safeurl.Client(cfg)
	router := gin.Default()
	router.GET("/webhook", func(context *gin.Context) {
		urlFromUser := context.Query("url")
		if urlFromUser == "" {
			errorMessage := "Please provide an url. Example: /webhook?url=your-url.com\n"
			context.String(http.StatusBadRequest, errorMessage)
		} else {
			stringResponseMessage := "The server is checking the url: " + urlFromUser + "\n"
			resp, err := client.Get(urlFromUser)
			if err != nil {
				stringError := fmt.Errorf("request return error: %v", err)
				fmt.Print(stringError)
				context.String(http.StatusBadRequest, err.Error())
				return
			}
			defer resp.Body.Close()
			bodyString, err := io.ReadAll(resp.Body)
			if err != nil {
				context.String(http.StatusInternalServerError, err.Error())
				return
			}
			fmt.Print("Response from the server: " + stringResponseMessage)
			fmt.Print(resp)
			context.String(http.StatusOK, string(bodyString))
		}
	})
	router.GET("/flag", func(context *gin.Context) {
		ip := context.RemoteIP()
		nip := net.ParseIP(ip)
		if nip != nil {
			if nip.IsLoopback() {
				context.String(http.StatusOK, "You found the flag")
			} else {
				context.String(http.StatusForbidden, "")
			}
		} else {
			context.String(http.StatusInternalServerError, "")
		}
	})
	router.GET("/", func(context *gin.Context) {
		indexPage := "<!DOCTYPE html><html lang=\"en\"><head><title>SafeURL - challenge</title></head><body>...</body></html>"
		context.Writer.Header().Set("Content-Type", "text/html; charset=UTF-8")
		context.String(http.StatusOK, indexPage)
	})
	router.Run("127.0.0.1:8080")
} 
```

如果您能够绕过 强制执行的检查`safeurl.Client`，标志的内容将为您提供有关如何领取奖励的进一步说明。请注意，获取标志的意外方式（例如，不绕过`safeurl.Client`）被认为超出范围。

随意贡献[拉取请求](https://github.com/doyensec/safeurl/pulls)、[错误报告或增强](https://github.com/doyensec/safeurl/issues)想法。

由于在 Doyensec 投入[了 25% 的研究时间，](https://doyensec.com/careers.html)这个工具成为可能。再次收看新剧集。