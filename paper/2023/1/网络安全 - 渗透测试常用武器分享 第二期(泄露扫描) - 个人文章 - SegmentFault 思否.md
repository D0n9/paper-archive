# 网络安全 - 渗透测试常用武器分享 第二期(泄露扫描) - 个人文章 - SegmentFault 思否
本期是针对渗透中的网站泄露扫描。泄露扫描分为针对网站的泄露扫描、针对第三方平台的泄露扫描。  
Packer Fuzzer  
开源 | 信息收集 | Python | 1.3k Star

![](https://segmentfault.com/img/bVcY7lZ)

简介: 一款针对Webpack等前端打包工具所构造的网站进行快速、高效安全检测的扫描工具。本工具支持自动模糊提取对应目标站点的API以及API对应的参数内容，并支持对：未授权访问、敏感信息泄露、CORS、SQL注入、水平越权、弱口令、任意文件上传七大漏洞进行模糊高效的快速检测。在扫描结束之后，本工具还支持自动生成扫描报告，您可以选择便于分析的HTML版本以及较为正规的doc、pdf、txt版本

点评: 如果你遇到VUE的站点，这个工具可能会给你带来惊喜

地址:[https://github.com/rtcatc/Pac...](https://link.segmentfault.com/?enc=Ut9IUrJPx6cpF8U4VdnvwA%3D%3D.mRR%2FnYL5bAKw0qmXMMElTstkmUVsPTW2fWE%2BqKcgcNlrWorHX25MlLd7Qg9Eu4rX)

JSFinder  
开源 | 信息收集 | Python | 1.5k Star

![](https://segmentfault.com/img/bVcY7l2)

简介: JSFinder是一款用作快速在网站的js文件中提取URL，子域名的工具。提取URL的正则部分使用的是LinkFinder

点评: 速度快，信息收集时候用起来还是很不错的！

地址: [https://github.com/Threezh1/J...](https://link.segmentfault.com/?enc=NDEXIMDxMHjUmazeSs0PnQ%3D%3D.qOJ3MMsUZxM%2FsMybqKl1BuMyZhc3u0RDSCN10Qe43fv9aDPztxOyurqYzASG4Shy)

HostCollision  
开源 | 信息收集 | Java | 296 Star

简介: 用于host碰撞而生的小工具,专门检测渗透中需要绑定hosts才能访问的主机或内部系统

点评: 在内网渗透中，有时候利用成功了。就离拿下不远了

地址: [https://github.com/pmiaowu/Ho...](https://link.segmentfault.com/?enc=FGp3jyWyVW6gCELaHlMZZQ%3D%3D.Vn0jE%2BHDTiJwU9h2OmbWwWYLucjb9SjSyRp%2FiQgASGTAR1FCaOiRtDsmax9%2FeAcq)

dirsearch  
开源 | 目录扫描 | Python | 7.8k Star

![](https://segmentfault.com/img/bVcY7l3)

简介: 网站路径扫描。使用字典破解网站目录和文件。而且支持递归破解

点评: 界面好看又快，作者还一直持续更新

地址:[https://github.com/maurosoria...](https://link.segmentfault.com/?enc=ma%2BXVE9UUYduhAzD%2Bzx3gQ%3D%3D.0%2BS0Js9VR57vHXDur5zgshu%2BJlZdqIOPS7J8cbHWgYa0oXzPsfcYYfXOHKj88vhk)

gobuster  
开源 | 目录扫描 | Golang | 5.8k Star

![](https://segmentfault.com/img/bVcY7l4)

简介: 跟dirsearch一样，可以爆破网站目录。功能还丰富点

dir：传统的目录爆破模式  
dns：DNS子域名爆破模式  
vhost：虚拟主机爆破模式  
点评: Golang版本，学习go的同学可以看看源码。能学习不少

地址:[https://github.com/OJ/gobuster](https://link.segmentfault.com/?enc=NS540Spm4kUGUiHQyDOPNg%3D%3D.a80DWaKU8P20P1fecCQ1k89JG%2FHhAerFU3ke%2FWiKqXs%3D)

SecretFinder  
开源 | 信1息收集 | Python | 7.8k Star  
![](https://segmentfault.com/img/bVcY7l5)

简介: SecretFinder是一个基于LinkFinder的python脚本, 用来发现JavaScript中的敏感数据，如apikeys, accesstoken，未授权，jwt等

点评: 重点是规则，可以参考下他的规则

地址:[https://github.com/m4ll0k/Sec...](https://link.segmentfault.com/?enc=gumXCs7Bd5%2Fl4I539rs46w%3D%3D.gcPp4%2Bry8n%2F%2BTzSusaTRd70jlIjGxv3PJ8WIcLG5al53D4W4uDjH7DXYDbK48erw)