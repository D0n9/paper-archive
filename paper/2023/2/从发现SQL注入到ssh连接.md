# 从发现SQL注入到ssh连接
​​**声明：本文仅限于技术讨论与分享，严禁用于非法途径。若读者因此作出任何危害网络安全行为后果自负，与本号及原作者无关。** 

**前言：** 

某天，同事扔了一个教育站点过来，里面的url看起来像有sql注入。正好最近手痒痒，就直接开始。

**一、发现时间盲注和源码**

后面发现他发的url是不存在SQL注入的，但是我在其他地方发现了SQL盲注。然后改站点本身也可以下载试用源代码，和该站点是同一套系统：

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt2r4t84j20u00tdwfl.jpg)

一开始的思路是直接用时间盲注写马，然后遇到的问题就是如何获取站点的绝对路径。通过sqlmap自带的字典去爆破，发现都失效了。（但是其实只是没写成功，不代表路径是不对。）那么接下来的思路就在源码上了。从源码里面没有找到啥可以直接未授权getshell的点。后面在本地搭建这套系统时，发现了其配置信息都在网站目录下的configure.php，后面就是尝试使用sqlmap读取文件。通过猜测，发现了站点的路径为/var/www/html/{站点域名}下面。然后再回头尝试写马，还是失败。但是可以读取文件。然后写了个脚本去跑，成功获取数据库账号密码：

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt2s2ampj20u005i3z9.jpg)

Nmap一试，3306开放，心中窃喜。使用mysql连接的时候，发现root登录被做了限制，只能使用localhost进行登录。然后也通过sqlmap获取到其他账号，有的可以登录，但是都因为权限小，无法写马。

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt2tfp1cj20hs06x42e.jpg)

**二、惊现上传漏洞**

写马失败后，想着查询下数据库里面的管理员密码，登录后台看看有没有可利用的点。后面又回过头来看源码了。一边放着dump数据，一边又发现了新东西，这站点存在ckfinder和ckeditor编辑器，但是一个无法访问，一个无法上传木马。

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt2v81cij20u00gbgmi.jpg)

就在我想破脑袋也没想到还有啥办法之时，我同事那边来了个好消息。他从旁站获取到了测试账号密码：

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt2whiuqj20rx0bkdgh.jpg)

然后他在个人资料处发现了一些功能点，发现了一堆xss和csrf、会话固定后，最后测了一下上传点

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt2y7zqzj20hs0eu7c3.jpg)

这个上传点如果你直接上传php是可以上传成功的，但是路径找不到。很奇怪。

不过如果你先上传一个jpg文件，就会发现图片路径为upload/fileimages/ew00000000040/user_photoa009.jpg

然后再通过bp修改文件扩展名为php，重新上传，就可以成功在前端看到php的路径：

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt30dcn0j20u00gsjsn.jpg)

通过抓包分析，我们发现他存在一个http\_user字段可控，并且只在前端校验文件类型得到重命名组合为user\_photo\[http_user\]\[传入的文件后缀(.php)\]

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt312m7mj20u00ebwft.jpg)
![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt325npaj20u00l60w1.jpg)

直接写入phpinfo()，发现解析了，上蚁剑：

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt33g2ssj20u00htwez.jpg)

成功getshell。

**三、脏牛提权**

虽然成功获取权限，但是这权限很低，有执行权限，但是很多操作都被限制。前面有获取数据库账号密码，在获取webshell后，可直接连接mysql数据库：

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt342pm0j20u00en3zd.jpg)

这时候可以考虑udf提权，但是尝试发现没有/usr/lib64/mysql/plugin/路径的上传权限。那么久只能通过常规的提权了。使用工具linux-exploit-suggester：https://github.com/mzet-/linux-exploit-suggester

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt356n7ej20u00l3q7c.jpg)

发现很多种方式可以提权，但是我用kali编译完的程序上传到目标机上，发现运行不了。后面直接在目标机编译，也出现确实一些库文件，好像因为目标机版本太低了。后面参考了这篇文章，成功进行提权。

https://www.jianshu.com/p/df72d1ee1e3e

Exp：https://github.com/FireFart/dirtycow

**四、SSH连接**

这个提权会删除root用户，新建一个用户firefart。本来还在考虑使用内网穿透把22端口代理出来，然后直接ssh连接。但是渗透步骤不规范就会导致我这样的结果：他的ssh并不是22端口，而是999端口。我信息收集的时候没有发现到位。当时一开始看没有22端口，所以才顺势觉得要穿透进去。但是其实人家999端口就是ssh。接下来就是成功使用ssh连接。

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt373tsuj20hs07pq3j.jpg)

但是有个问题又出现了：如果我想连接ssh，那么久只能使用这个账户登录，因为我不知道root密码。但是这样的话，人家登录不了root就会发现异常。但是如果我把root恢复了，我就没有root权限了。

诶，后面我在想，如果我把原始的passwd文件恢复，然后不断开ssh连接是不是我还能有权限操作呢？说干就干，使用firefart执行mv /tmp/passwd.bak /etc/passwd恢复原本的账户。然后ssh不断开，我发现我还是root权限。这就好办了。useradd新建账号edu，然后把新建的账户加入管理员组。具体操作可以参考：https://blog.csdn.net/llm_hao/article/details/118031154

使用新账号edu进行登录，发现为root权限，成功！

![](https://wx1.sinaimg.cn/mw1024/eab03d57ly4hatt38r9ccj20u00evt99.jpg)

这时候才把原本firefart账号的窗口关闭。重新再使用firefart账户登录，发现已经无法登录了。看来这应该是系统的一种机制吧，哈哈哈。

**结尾**

这次渗透其实走了很多弯路，到最后都没用上数据库。很多时候一个点打不进去的时候，适当的放弃，去打新的点，不要太头铁，特别是攻防的时候。

总结一下：发现盲注，源码到跑取站点账号密码（时间盲注效率低到我现在还没跑出后台管理账户密码），无果。到从旁站上传木马，获取网站服务器权限，权限较低，使用脏牛提权，到后面的恢复原本的账户，并新建一个管理员。其实这个站点是还有内网在，貌似是教育局办公内网，但目前还在尝试，后续会随缘更新。​​​​