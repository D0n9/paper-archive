# webpack解析之详细过程 - FreeBuf网络安全行业门户
简介
--

webpack是一个JavaScript应用程序的静态资源打包器（module bundler）。它会递归构建一个依赖关系图（dependency graph）,其中包含应用程序需要的每个模块，然后将所有这些模块打包成一个或多个bundle。

可以直接使用浏览器的调试模式进行查看，对泄露的各种信息如API、加密算法、管理员邮箱、内部功能等等。  
使用wappalyzer发现该网站使用webpack打包器。  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/caf6d240-5cf4-4793-80af-65d75d412858.jpeg?raw=true)

查看源码方法
------

在源码中可能存在敏感的API接口，可以直接看Js文件，也可以还原出来在本地查看。

方法一：  
使用谷歌插件可以直接下载代码：  
[https://github.com/SunHuawei/SourceDetector](https://github.com/SunHuawei/SourceDetector)

方法二：

#安装 reverse-sourcemap

npm install --global reverse-sourcemap

#下载 *.js.map (右键查看源代码，找到 js ，在 js 后面加上 .map)

curl -O https://IP/*.js.map

#使用 reverse-sourcemap

reverse-sourcemap --output-dir ./JS xxx.js.map

#得到 JS

本地还原前端代码过程
----------

在这里使用npm安装,安装npm需要先安装homebrew包管理器，在利用sourcemap将前端代码还原到本地。

### 安装HomeBrew

Homebrew 是一个包管理器，用来在 macOS 安装 Linux 工具包。  
使用以下命令，非root权限下运行。

```
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"

```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/2e3f0f75-5897-4b8f-8bce-99244bd6d74b.jpeg?raw=true)

查看版本信息  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/258c8890-6cae-4d4f-ab68-6b4483774130.jpeg?raw=true)

### 安装npm

使用如下命令直接安装

```
brew install node

```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/e24633bf-0e3a-4032-a328-7637bb9d8ca5.jpeg?raw=true)

### 安装reverse-sourcemap

使用以下命令

```
npm install --global reverse-sourcemap

```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/03e457d6-cd78-4d0d-856c-9d3d32dda79b.jpeg?raw=true)

webpack解析js map文件
-----------------

1、找到要下载的js文件。  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/f4dca8e6-7d5c-4ec4-9a07-979dfde1f4d6.jpeg?raw=true)

2、通常我们要找到的SourceMap 映射文件都在这些文件的最下面有个注释的地方。

```
//#sourceMappingURL=pages.sessions.new.7ef701e7.chunk.js.map

```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/1482f4d1-4f87-4868-b92e-938bebcf7529.jpeg?raw=true)
  
3、使用`curl -O #把输出写到该文件中，保留远程文件的文件名`命令。

```
sudo curl -O http://gitlab.neoroll.xin/assets/webpack/pages.sessions.new.7ef701e7.chunk.js.map

```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/396cd5e3-39ca-485e-bc44-136b778e11a1.jpeg?raw=true)

文件保存到自己当前目录下  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/19188a80-a430-411a-a8d0-c9441727dc17.jpeg?raw=true)
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/36065f45-33fa-47a4-beb7-f3e5e3f1ab66.jpeg?raw=true)
使用另外一种方法进行URL拼接保存文件试了下发现下载的文件乱码。  
js添加.map就可以直接进行下载。  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/828b9d11-32cd-42d1-8fcb-b5c3581c4bf8.jpeg?raw=true)

我这里文件打开乱码，无法进行编译。  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/0fd69d73-5bc1-4a74-a847-01399ae2c1b0.jpeg?raw=true)
4、reverse-sourcemap --output-dir XXX(导出路径) app.XXX.js.map(打包文件中的.js.map文件)  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/509b6908-b934-47d8-bd4c-756c7b711768.jpeg?raw=true)
成功编译成功,打开就可以查看源码了。  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/c1f9ed1c-fe34-4cec-86d3-3c52f8f60143.jpeg?raw=true)