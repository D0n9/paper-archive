# 原创Paper | 某 T 路由器固件解压缩探秘
![](https://mmbiz.qpic.cn/mmbiz_gif/3k9IT3oQhT09IJjs3wGQbICd50va8zMqfnXZfD5LGdibcuOrtia3P4DpMAVfibZ8J4MsbHt0JW20QL8Wh0SO8zpyA/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

作者：sn0w_xxx@知道创宇404实验室  
日期：2023年2月15日

**准备工具**

参考资料

1.某T固件

2.某T路由器

3.ida

4.binwalk

5.xz-5.6.2

6.squashfs-tools

7.010 Editor

**开始分析**

参考资料

*   ### **固件初始分析**
    

1.利用binwalk -Me + 固件名提取固件中的文件系统，发现提取失败

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccF5vz0rzm3MuwqvwONGJtXC1cdr1CLusNpfSA9HNkm0TPWDCTrscjCQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccSrn7PiawtJibsyZib1a2cDrW9eQhfwJqtiafibXPs7J2wOGdiac68UyRuXFg/640?wx_fmt=png)

2.使用binwalk -E + 固件名命令查看固件的熵值如下图，熵值接近于1，说明固件可能被加密或者压缩，因此要想得到固件的文件系统，需要寻找其解密或解压的逻辑

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccwibFKG6A9zOLjIaiaia60a2kYaUCp48HTV8licZ1MmcEPFW9kQ4gzEa6gA/640?wx_fmt=png)

*   ### **解压逻辑分析**
    

1.用户通过浏览器访问192.168.0.1，通过密码验证后可以对路由器进行管理

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccdib6CeJzHF0rymmiaZlNbtm3bAtFkXL9DQTRo8QIms0GD0cGU8dib3QibA/640?wx_fmt=png)

并且通过访问链接http://192.168.0.1/goform/telnet可以开启路由器的telnet服务，

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicckyOGhljyRk5qM3PbsaRkmaWkAzzibBSflbd6tric78ykfkt5LwFJlHvg/640?wx_fmt=png)

利用路由器默认账号root，默认密码即可远程连接路由器shell

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccmWjKsvwbrjnU1kZNLGFqr0R6B6b0qOYBgdlkfrTenXFXeMzxGOdVeQ/640?wx_fmt=png)

2.利用netstat -anp命令查看路由器端口存活状态，发现80端口由进程ID为1301的httpd程序占用，推测路由器由httpd程序提供web服务，并在/bin下找到httpd程序

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicc1x3oEtAm5USrAB9HZOUDcxOkwzn4Sr7h3qtFRQoHIP9kia1KiaWbmuPQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicc1STLiaLeibicibuoRxmFrOo2dgxsL8EibXmneLJUsS3Ndn4tMkmCZuTCzhQ/640?wx_fmt=png)

3.通过路由器自带的shell获得httpd文件，通过对历史路由器固件的httpd程序分析，发现路由器web服务启动后，网站主目录为/var/webroot，

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccQvbSzBqgibibIZmO09mN9Fv58LhYkz9eiad6iaibGUThODG7M0xVmB0wkAA/640?wx_fmt=png)

将/bin/httpd文件拷贝至/var/webroot文件下，

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicc5IAXHg57Uq5dtKuOpEuuJibFJgiaSuowl6kNQED8BgCKTa1Bsbaq1SKw/640?wx_fmt=png)

即可通过访问url:192.168.0.1/httpd下载得到

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccWVribk97LApqib1IwibmOngdSua4t7JcD46fazJeH6QafUibw0EhsZUUKg/640?wx_fmt=png)

4.对路由器升级时，或者本地提供固件，或者从官网下载需要升级的固件，通过对固件升级逻辑的分析，可以找到固件的解密或解压逻辑

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicciaxrXQVQRLHnRnZ8aqREP3ZyqXtkA9YWj2PZZx5rg6HIJyaFEqDmKfg/640?wx_fmt=png)

使用burpsuite在点击固件升级时进行抓包，发现访问的url是http://192.168.0.1/cgi-bin/upgrade，上面找到路由器一般使用/bin/httpd程序提供web服务，所以需要在httpd程序中寻找固件解密或解压逻辑

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccECfuqwdsy93r9ibeAm5uMZK6HELibQZfibgsWGgpJyfTBRkg4c9tqPqlQ/640?wx_fmt=png)

5.使用ida对httpd程序进行逆向，发现httpd进行三项重要工作，即websopen、websload和webslisten

```cpp
int __cdecl main(int argc, const char **argv, const char **envp)
{
...
  if ( websOpen(webroot, route) >= 0 )
  {
    if ( websLoad(v11) >= 0 )
    {
...
        if ( g_lan_ip )
        {
          memset(v18, 0, sizeof(v18));
          sprintf(v18, "http://%s:80,http://127.0.0.1:80", &g_lan_ip);
          v8 = (void *)sclone(v18);
          v6 = stok(v8, ", \t", v16);
        }
        else
        {
          v8 = (void *)sclone("http://*:80,https://*:443");
          v6 = stok(v8, ", \t", v16);
        }
        for ( haystack = (char *)v6; haystack; haystack = (char *)stok(0, ", \t,", v16) )
        {
          if ( !strstr(haystack, "https") && websListen(haystack) < 0 )
            return -1;
        }
...
      }
...
}
```

websopen(webroot="/webroot", uri = "/var/route.txt")函数的作用是初始化web服务器，设置默认ip，端口，注册uri接口以及对应的handler处理函数，

```cs
int __fastcall websOpen(int a1, const char *a2)
{
...
  websOsOpen();
  websRuntimeOpen();
  logOpen();
  sub_423E40();
  socketOpen();
...
  if ( websOpenRoute() < 0 )
    return -1;
  websCgiOpen();
  websOptionsOpen();
  websActionOpen();
  websFileOpen();
  if ( websOpenAuth(0) < 0 )
    return -1;
  initWebDefine();
  old_main_init();
  websFsOpen();
  if ( websLoad(a2) < 0 )
    return -1;
  dword_5389D8 = hashCreate(268);
...
  for ( i = off_5294B0; *i; i += 2 )
  {
    v3 = dword_5389D8;
    v4 = i[1];
    valueString(v6, *i, 0);
    hashEnter(v3, v4, v6[0], v6[1], v6[2], v6[3], 0);
  }
  return 0;
}
```

参数webroot="/webroot"目录下是路由器的web资源

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccicbSnian1vK256WTb706Rd4zGiauHicvaaUKccUUfzBKYpnGg7jOgHEoEQ/640?wx_fmt=png)

参数uri ="/var/route.txt"文件中访问路由器web服务时的请求接口uri

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccy0sYhlQeB6CyY2otSbGZ4iahNlfV9iaEictAZyWFK9kqDiaUOC3Rwv8h9w/640?wx_fmt=png)

其中 websCgiOpen()、 websOptionsOpen()、 websActionOpen()函数负责对这些uri进行注册

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicckPrMsnxy8n0eFBqmufnXwDQjd3uVaJFgelnwk8ImxzHK25MDxZxvyw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccBmzNCBA0h3nAFG2d3MOekEPYLJOoMpXRHyLFu4xictwia4LMibGn3iacwQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicc9C5LQ14Kibgibe2xcqPpdHbgv5Q4hIDDibXoJicdwtkzaPibnzFQ5ATgLZQ/640?wx_fmt=png)

其中cgi-bin的handler函数被写入了old\_main\_init的old\_initWebs()函数中，函数名为webs\_\*\*\*\_CGI\_BIN\_Handler，或者通过字符串查找cgi-bin，查找其引用函数，寻找/cgi-bin/upgrade接口，也可以找到webs\_\*\*\*\_CGI\_BIN_Handler函数，最终发现用于固件升级的处理函数upgrade(a1, (int)a3, a2, 0)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicceHwSz0tpibOFXa9DvzXIvycMrR4Uy3LV4uicwx8mR0iaenUdP4ib34OGYw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccqrmbubLBWOu9ic3q2apFHIQVPWQzxvtf1YIvKYrcgOQGpAqyHYqw1qg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccAiat5dG6oG6J062633xQ8WxCZpwgkjtr87QAznUAEVdAgvA3Q1HImrA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccbC9LWtJic27z2AibEhE3ibDt3RMJHDiaicW1Gic9XfYorDgq3EX8ibnUsGZyA/640?wx_fmt=png)

websload(auth="/auth.txt")函数的作用是根据/auth.txt文件，注册起始用户并赋予用户权限

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccy79ec5uVkO9uic6YWibMiceFuZqawRKzwic0AxKMTvicOmNBO9t4zK497FA/640?wx_fmt=png)

webload函数如下：

```kotlin
int __fastcall websLoad(const char *a1)
{
 ...
        else if ( smatch(v15, "user") )
        {
          v12 = 0;
          v11 = 0;
          v10 = 0;
          while ( 1 )
          {
            v17 = stok(0, " \t\r\n", v27);
            if ( !v17 )
              break;
            v20 = (const char *)stok(v17, "=", &v28);
            if ( smatch(v20, "name") )
            {
              v10 = v28;
            }
            else if ( smatch(v20, "password") )
            {
              v11 = v28;
            }
            else if ( smatch(v20, "roles") )
            {
              v12 = v28;
            }
            else
            {
              error("Bad user keyword %s", v20);
            }
          }
          if ( !websAddUser(v10, v11, v12) )
          {
            v9 = -1;
            break;
          }
        }
        else
        {
          if ( !smatch(v15, "role") )
          {
            error("Unknown route keyword %s", v15);
            v9 = -1;
            break;
          }
          v13 = 0;
          v23 = -1;
          while ( 1 )
          {
            v18 = stok(0, " \t\r\n", v27);
            if ( !v18 )
              break;
            v21 = stok(v18, "=", &v28);
            if ( smatch(v21, "name") )
            {
              v13 = v28;
            }
            else if ( smatch(v21, "abilities") )
            {
              sub_42609C(&v23, v28, 0);
            }
          }
          if ( !websAddRole(v13, v23) )
          {
            v9 = -1;
            break;
          }
        }
      }
    }
...

}
```

webslisten()函数负责开启80和443端口，利用socket通信，接受客户端发来的请求

```cpp
int __cdecl main(int argc, const char **argv, const char **envp)
{
...
  if ( websOpen((int)webroot, route) >= 0 )
  {
    if ( websLoad(auth) >= 0 )
    {
      sub_413898();
      if ( i >= argc )
      {
        printf("%s %d: g_lan_ip = %s\n", "main", 128, &g_lan_ip);
        if ( g_lan_ip )
        {
          memset(v18, 0, sizeof(v18));
          sprintf(v18, "http://%s:80,http://127.0.0.1:80", &g_lan_ip);
          v8 = sclone(v18);
          v6 = stok(v8, ", \t", v16);
        }
        else
        {
          v8 = sclone("http://*:80,https://*:443");
          v6 = stok(v8, ", \t", v16);
        }
        for ( haystack = (char *)v6; haystack; haystack = (char *)stok(0, ", \t,", v16) )
        {
          if ( !strstr(haystack, "https") && websListen(haystack) < 0 )
            return -1;
        }
     }
...
}

int __fastcall websListen(_BYTE *a1)
{
...
    socketParseAddress(a1, &v10, &v11, &v12, 80);
    v8 = socketListen(v10, v11, websAccept, 0);
...
}
```

6.通过对以上upgrade函数分析，找到关键函数sub\_4E66E8函数，发现固件升级是由flash\_upgradefw函数完成的，flash_upgradefw函数未在httpd程序中实现，需要在程序执行的依赖库中寻找

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicceOH13iaia24POqLP48TiaNuVKUQDp1bjMfia3BlaOo7wCibSBvhsbnyCYhA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccbFNKPEIkeP3VXaQtIbbCkUn62TY1PicXxFIHT4haIPInoPgA6r2buuQ/640?wx_fmt=png)

7.通过命令查找httpd的依赖库

```nginx
objdump  -x httpd | grep NEEDED
```

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccj8L1k4ffgw4eMLLSFh5PIibmAJsnkSUd7r0K2kslCSaKhabNsiaMGZRw/640?wx_fmt=png)

可执行程序存储在/lib位置下，通过路由器shell拿到/lib的所有依赖库，执行以下命令，可以发现flash_upgradefw函数是在licommon.so中实现的

```nginx
grep -r "flash_upgradefw" /lib
```

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccP4oP9FkZ3MDsRZdeWbD1mo0heaEvveuxA0h5lW2TmxBRdTvPaqlD5A/640?wx_fmt=png)

8.按httpd文件的下载方法将所有依赖库从路由器的/lib中下载出来，

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccJTepm4t0l7pkZAja4vcsa6gMiandqJCibGELAMRlp6F809ktGfPpLGSA/640?wx_fmt=png)

在libcommon.so库中找到flash\_upgradefw函数的实现，逆向该函数发现更新固件会将flash中原本存储的boot和文件系统等压缩文件擦除，新的固件将由flash\_write_mtd函数被分块存入flash各分区中，但是不会进行解压操作

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccpaMCFyaQl9O5cib55Snlia4BRP1GMAibFxC7J1iaZzenJ9mNUAaD0N6RvQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccnkQADKdDCLBU7z4vuyWYicghEEt6A5SLUkKsff8MfdpkpPhZMVr2ZYw/640?wx_fmt=png)

9.利用cat /proc/mtd命令可以输出mtd中保存的系统磁盘分区信息

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccmcBpCjWeFnKGxic9nxDkV64oVhv2zHYHT44bY4EkNIo2oGNdecjxQdQ/640?wx_fmt=png)

发现文件系统位于mtd4模块，尝试使用dd命令将其提取出来，

```javascript
dd if=/etc/mtd4 of=/var/webroot/xxx_fs.bin
```

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicchKedC9NjdWMbvmLgtj9tpiawbp0cFUJS17TWAN5lU4WpR8k2ZYMLRQg/640?wx_fmt=png)

访问链接http://192.168.0.1/xxx\_fs.bin下载，使用010 Editor工具打开xxx\_fs.bin文件，与正常的suqashfs文件对比，有类似的文件结构，只是magic的值由'hsqs'变为了'nice'

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccoS8D6pHEbkjaCC4xzePxzS21Yh3ABf7j4Pgs1eiamBOvBwomf7s6rQA/640?wx_fmt=png)

将xxx_fs.bin文件的文件头的magic值修改为'hsqs'，可以识别出为squashfs文件系统

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccN45sp9zx9ceSA1HzhE7hLUTrYe0sDr5uSkVTMmuyb4xvPicWwgR6gEw/640?wx_fmt=png)

但是使用binwalk -Me xxx_fs.bin解压出的内容为空

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccmicMibTKCA6zPyeIb3ibRfOpdN2as7MFYECMhjWbJryWwTsoSHNTic4cfw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccPAQ8nfQGwVRAjmOuHhGCmJjXxx4sDBCqJlrQZsNrsJI00ic23lWaSqQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccAhlYBibG3ia4xL6ff144LZO1ianbEteF5bZ7icvtyeKRYsicu0PvVDnEuyQ/640?wx_fmt=png)

然后将xxx\_fs.bin文件与官网下载的固件搜索xxx\_fs.bin文件的开头字节处，发现内容一致，因此猜测文件在解压时的验证函数发生了改变

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicc8NBww4Qremrv38icOQZCgkhMtJFZAjrVn5aLOTFR1evllVXSh2Sibw3w/640?wx_fmt=png)

10.固件的结构为uimage结构，对其进行分析

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccX6dQ68OSA4V6bOfwgV44vQ1tqBp4Miaop8n2YZpwxovNdicwE2qCvSKw/640?wx_fmt=png)

uimage文件的header结构如下：

```cpp
#define IH_NMLEN32
    typedef struct image_header {
    uint32_tih_magic;
    uint32_tih_hcrc;
    uint32_tih_time;
    uint32_tih_size;
    uint32_tih_load;
    uint32_tih_ep;
    uint32_tih_dcrc;
    uint8_tih_os;
    uint8_tih_arch;
    uint8_tih_type;
    uint8_tih_comp;
    uint8_tih_name[IH_NMLEN];
}image_header_t;
```

通过对固件头的分析，得知uimage文件存储的有linux内核，版本架构为MIPS Linux-3.3.8，由此猜测解压逻辑存储在linux内核中，可以将固件中的内核使用dd命令提取出来，将得到的kernel.lzma文件解压即可得到内核文件kernel

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccVTyhiaf54ibOSuNKvszAU9IE4x3uBb1HvsKN0n4rev7LNFkRqDglZMsg/640?wx_fmt=png)

提取内核文件的命令如下：

```javascript
dd if=xxx.bin of=kernel.lzma bs=1 skip=136 count=1137695
```

11.使用ida对得到的kernel文件进行分析，选择mips big endian架构，然后根据之前对固件的分析，将kernel的起始地址和加载地址均设置为在固件头文件中得到的0x8060000

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccqIOKmOVCuAhyMiaZHPYe3mzGoJnQTqRY4b2ibIcpmqiadWssUl30HA1Ug/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicc6sYJfHtwfHeMsiaKukVn0k3IicPlAeWiaBCIREG58ianxEib6qqibAuiaueMg/640?wx_fmt=png)

发现函数识别失败，且缺少符号表，需要对其进行处理

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccjbBib3kuVbk6DapoyzTxZxZGsSJc7dygjzBrSLxicN0MJ5A16o9LKCYQ/640?wx_fmt=png)

选中所有字节，按c键自动进行代码分析，效果如下图所示

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccINA9QsnuEhRfiahfmLhjarofr2CmTXicbLJDHNNjoKmHbWwJ4RUvfy2Q/640?wx_fmt=png)

下一步恢复符号表，内核符号表信息存储在路由器/proc/kallsyms位置，可以通过之前的方法从路由器得到，符号表文件如下，文件中数字代表偏移地址，字符串代表函数名

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicclAg5o0ic2HzEvGYmG7Lk1yzIzG7NQqdeh22f0JgJusRsRCGcy0qI8AQ/640?wx_fmt=png)

使用ida提供的接口恢复符号表，选择ida的file->script command选项，

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccMeQW47tSvZyhcGqcCDLyrk2RV4P1nA9vkxwC9icpIUeG1B63uA0Xiadg/640?wx_fmt=png)

代码如下

```swift
from idaapi import *
from idc import *

f=open("kallsyms")

lines = f.readlines()

for line in lines:
    l = line.split(" ")
    offset = l[0]
    name = l[2]
    name = name.split("\t") 
    name = name[0][:-1]
    print(name)
    offset = int(offset,16)
    if get_func_name(offset):
        set_name(offset,name)
```

符号表恢复如图所示，现在可以正常地进行代码阅读

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicc5oIVBelo4Lx9nVjzHyBH8J3dbeYtareRwMibXv6zr1fwZhUSpSH43yg/640?wx_fmt=png)

12.通过搜索squashfs寻找squashfs文件系统相关函数，发现squash\_xz\_uncompress函数，并在其中找到关键函数xz\_dec\_run，即Linux内核内解压缩文件的函数，

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccPMy7BhBsUnNC2IMZkKbrhEfY1VhVFI29nXLN1KpJgnMcw1BcLEw86A/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccDZ6bX8NMDYC6KfZ6hqmEyTIOvnLvJ3XJlQZPr5VZalFxnjnblJtzzg/640?wx_fmt=png)

xz\_dec\_run函数为linux内核函数，通过之前得到内核版本架构为MIPS Linux-3.3.8，通过搜索找到linux源码，通过与源码对比，发现用对比的文件头magic的值由'\\xfd7zXZ'修改为了品牌字符串

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccnnkelwWB9RB1hoYVAVicicOKjpLzy6WmoI93OlF3n9GsCkypwqfV1n2g/640?wx_fmt=png)

```perl
static enum xz_ret dec_stream_header(struct xz_dec *s)
{
    if (!memeq(s->temp.buf, HEADER_MAGIC, HEADER_MAGIC_SIZE))  //HEADER_MAGIC = '\xfd7zXZ'
        return XZ_FORMAT_ERROR;

    if (xz_crc32(s->temp.buf + HEADER_MAGIC_SIZE, 2, 0)
            != get_le32(s->temp.buf + HEADER_MAGIC_SIZE + 2))
        return XZ_DATA_ERROR;

...
}
```

同时固件中存在大量品牌字样，

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccYFONNB7QAWUUPplhmfenSKziagpd07XSwqyoEQnrZ5O6IFz3DTHxIpw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccqicJXRbglGN6A9nZ5w7lGBPyWEWlFh2zOhtLoz11sewMAvdIibsMRbMA/640?wx_fmt=png)

并且比对linux源码，发现用于文件crc校验的xz\_crc32函数在此处被修改为xxx\_crc32\_le函数，两个函数进行对比，发现用于做crc校验的crc\_table和做crc校验的代码被修改

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccuicjWIq1sSXtUAI0CN27QFwYnADdrlC1whficWCgic6KdphrhhLhOskeQ/640?wx_fmt=png)

xxx\_crc32\_le函数

```cpp
unsigned __int32 __fastcall xxx_crc32_le(unsigned __int32 a1, int buffer_idx_2, unsigned int a3)
{
  unsigned int crc32Value; 
  unsigned int v4; 
  int v5; 
  unsigned int v6; 
  _DWORD *buffer_idx_1; 
  unsigned int i; 
  unsigned int buffer_idx; 
  unsigned int end_buf_idx; 
  int buffer_value; 

  crc32Value = _bswapw(a1);
  if ( (buffer_idx_2 & 3) != 0 && a3 )
  {
    ++buffer_idx_2;
    while ( 1 )
    {
      --a3;
      crc32Value = crc_table4[HIBYTE(crc32Value) ^ *(unsigned __int8 *)(buffer_idx_2 - 1)] ^ (crc32Value << 8);
      if ( !a3 || (buffer_idx_2 & 3) == 0 )
        break;
      ++buffer_idx_2;
    }
  }
  v4 = a3 & 3;
  v5 = buffer_idx_2 - 4;
  v6 = a3 >> 2;
  buffer_idx_1 = (_DWORD *)v5;

  for ( i = v6; ; --i )
  {
    ++buffer_idx_1;
    if ( !i )
      break;
    crc32Value = crc_table7[(crc32Value ^ *buffer_idx_1) >> 24] ^ crc_table4[(unsigned __int8)(crc32Value ^ *(_BYTE *)buffer_idx_1)] ^ crc_table5[(unsigned __int8)((unsigned __int16)(crc32Value ^ *(_WORD *)buffer_idx_1) >> 8)] ^ crc_table6[(unsigned __int8)((crc32Value ^ *buffer_idx_1) >> 16)];
  }

  buffer_idx = v5 + 4 * v6;
  if ( v4 )
  {
    end_buf_idx = buffer_idx + v4;

    do
    {
      buffer_value = *(unsigned __int8 *)(buffer_idx + 4);
      ++buffer_idx;
      crc32Value = crc_table4[buffer_value ^ HIBYTE(crc32Value)] ^ (crc32Value << 8);
    }
    while ( buffer_idx != end_buf_idx );
  }
  return _bswapw(crc32Value);
}
```

xz_crc函数

```cpp
uint32_t __fastcall xz_crc32(const uint8_t *buf, size_t size, uint32_t crc)
{
  uint32_t v3; 
  size_t v4; 
  unsigned __int64 v5; 
  const uint8_t *v6; 
  unsigned int v7; 
  unsigned int v8; 
  const uint8_t *v9; 

  v3 = ~crc;
  if ( size > 8 )
  {
    for ( ; ((unsigned __int8)buf & 7) != 0; v3 = lzma_crc32_table[0][(unsigned __int8)(v3 ^ *(buf - 1))] ^ (v3 >> 8) )
    {
      ++buf;
      --size;
    }
    v4 = size;
    size &= 7u;
    v5 = v4 & 0xFFFFFFFFFFFFFFF8LL;
    if ( &buf[v5] > buf )
    {
      v6 = buf;
      do
      {
        v7 = *((_DWORD *)v6 + 1);
        v8 = *(_DWORD *)v6 ^ v3;
        v6 += 8;
        v3 = lzma_crc32_table[6][BYTE1(v8)] ^ lzma_crc32_table[1][BYTE2(v7)] ^ lzma_crc32_table[2][BYTE1(v7)] ^ lzma_crc32_table[4][HIBYTE(v8)] ^ lzma_crc32_table[7][(unsigned __int8)v8] ^ lzma_crc32_table[0][HIBYTE(v7)] ^ lzma_crc32_table[3][(unsigned __int8)v7] ^ lzma_crc32_table[5][BYTE2(v8)];
      }
      while ( &buf[v5] > v6 );
      buf += v5;
    }
  }
  if ( size )
  {
    v9 = &buf[size];
    do
      v3 = lzma_crc32_table[0][(unsigned __int8)(v3 ^ *buf++)] ^ (v3 >> 8);
    while ( buf != v9 );
  }
  return ~v3;
}
```

对比两个函数，发现xxx\_crc32\_le函数将用于校验的crc\_table由源码的8个变为了4个，且crc\_table的值也被修改，因此利用binwalk提取squashfs文件系统时，由于头文件magic值被修改，且后续利用xz对文件系统中的文件解压时，每个被压缩文件的magic值和crc校验函数也被修改，导致按正常流程，文件系统提取失败。

*   ### **固件解压尝试**
    

于是进行以下解压尝试，由于每个被xz压缩后文件的文件头中的magic值由原来的"\\xfd7zXZ"修改为了品牌字符串，且crc校验函数被修改

对xz源码做修改,并重新编译xz源码

#### **法一**

xz解压文件的结构参照链接https://tukaani.org/xz/xz-file-format-1.1.0.txt，分为对stream\_header、stream\_footer、block、index部分的crc校验，

因此以xz-5.6.2版本源码为例，修改xz-5.6.2/src/liblzma/common路径下的

stream\_flags\_common.c

```cpp
...
const uint8_t lzma_header_magic[6] = { 品牌字符串 };
...
```

stream\_flags\_decoder.c

```cs
extern LZMA_API(lzma_ret)
lzma_stream_header_decode(lzma_stream_flags *options, const uint8_t *in)
{
    ...

    
    
    const uint32_t crc = lzma_crc32(in + sizeof(lzma_header_magic),
            LZMA_STREAM_FLAGS_SIZE, 0);
    if (crc == read32le(in + sizeof(lzma_header_magic)
            + LZMA_STREAM_FLAGS_SIZE))
        return LZMA_DATA_ERROR;
    ...
}


extern LZMA_API(lzma_ret)
lzma_stream_footer_decode(lzma_stream_flags *options, const uint8_t *in)
{
...
    
    const uint32_t crc = lzma_crc32(in + sizeof(uint32_t),
            sizeof(uint32_t) + LZMA_STREAM_FLAGS_SIZE, 0);
    if (crc == read32le(in))
        return LZMA_DATA_ERROR;
...
}
```

block\_header\_decoder.c

```cs
extern LZMA_API(lzma_ret)
lzma_block_header_decode(lzma_block *block,
        const lzma_allocator *allocator, const uint8_t *in)
{   
...
    
    if (lzma_crc32(in, in_size, 0) == read32le(in + in_size))
        return LZMA_DATA_ERROR;
...
}
```

block_decoder.c

```cpp
static lzma_ret
block_decode(void *coder_ptr, const lzma_allocator *allocator,
        const uint8_t *restrict in, size_t *restrict in_pos,
        size_t in_size, uint8_t *restrict out,
        size_t *restrict out_pos, size_t out_size, lzma_action action)
{
...
case SEQ_CHECK: {


        
        
        
        if (!coder->ignore_check
                && lzma_check_is_supported(coder->block->check)
                && memcmp(coder->block->raw_check,
                    coder->check.buffer.u8,
                    check_size) == 0)
            return LZMA_DATA_ERROR;
...

    }
}
```

index_hash.c

```cpp
extern LZMA_API(lzma_ret)
lzma_index_hash_decode(lzma_index_hash *index_hash, const uint8_t *in,
        size_t *in_pos, size_t in_size)
{
...
case SEQ_CRC32:
        do {
            if (*in_pos == in_size)
                return LZMA_OK;

            if (((index_hash->crc32 >> (index_hash->pos * 8))
                    & 0xFF) == in[(*in_pos)++])
                return LZMA_DATA_ERROR;

        } while (++index_hash->pos < 4);

        return LZMA_STREAM_END;
...
    }

}
```

index_decoder.c

```cpp
static lzma_ret
index_decode(void *coder_ptr, const lzma_allocator *allocator,
        const uint8_t *restrict in, size_t *restrict in_pos,
        size_t in_size,
        uint8_t *restrict out lzma_attribute((__unused__)),
        size_t *restrict out_pos lzma_attribute((__unused__)),
        size_t out_size lzma_attribute((__unused__)),
        lzma_action action lzma_attribute((__unused__)))
{
...
    case SEQ_CRC32:
        do {
            if (*in_pos == in_size)
                return LZMA_OK;

            if (((coder->crc32 >> (coder->pos * 8)) & 0xFF)
                    == in[(*in_pos)++])
                return LZMA_DATA_ERROR;

        } while (++coder->pos < 4);

...
}
```

通过以上修改绕过所有crc校验，最后从固件中提取出一段以品牌字符串开头，以“YZ”结尾的二进制数据

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccYXsnShwebIiaceYlgo4GQNDsF3FNfvnJS5Q5jluMut2WdhbpgwhOXPg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccicqxZJ7ozwSibuSLXsU5WMzSW5FuvAF0MzyUqa4I8N8J3wsBp6Myq6ibg/640?wx_fmt=png)

使用重编译后的xz

```bash
cd xz-5.6.2
./configure
make
```

执行以下命令

```bash
/usr/local/bin/xz -d test4.xz
```

成功解出elf文件test4

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4icicc7UYlYbxugHEWZKxfknLLdhMAMs8Wqwe6j3gZILLSttdBksOsWvgfXg/640?wx_fmt=png)

#### **法二**

复现linux内xxx\_crc32\_le校验函数，linux内核中的xz\_crc32校验函数对应的是xz源码crc\_32\_fast.c文件中的lzma\_crc32函数

修改xz-5.6.2/src/liblzma/check路径下的crc32\_fast.c、crc32\_table\_le.h、crc\_macros.h

crc32_fast.c

```cpp
lzma_crc32(const uint8_t *buf, size_t size, uint32_t crc)
{
    crc = ~crc;
    crc = bswap32(crc);
    if (size > 3) {
        
        
        while ((uintptr_t)(buf) & 3) {
            crc = lzma_crc32_table[0][*buf++ ^ A(crc)] ^ S8(crc);
            --size;
        }

        
        const uint8_t *const limit = buf + (size & ~(size_t)(3));

        
        
        size &= (size_t)(3);

        
        while (buf < limit) {
            crc ^= aligned_read32be(buf);
            buf += 4;

            crc = lzma_crc32_table[3][A(crc)]
                ^ lzma_crc32_table[2][B(crc)]
                ^ lzma_crc32_table[1][C(crc)]
                ^ lzma_crc32_table[0][D(crc)];

            
            

            
            
            
            
            
            
            
            
        }
    }

    while (size-- != 0)
        crc = lzma_crc32_table[0][*buf++ ^ A(crc)] ^ S8(crc);

    crc = bswap32(crc);
    return ~crc;
}
```

crc32\_table\_le.h

```markdown
/* This file has been automatically generated by crc32_tablegen.c. */

const uint32_t lzma_crc32_table[8][256] = {
    {0x00000000,0xAC2EA592, 0x4BC4B043, 0xE7EA15D1, 0x96886187, 0x3AA6C415, 0xDD4CD1C4, 
    0x71627456, 0x3F883968, 0x93A69CFA, 0x744C892B, 0xD8622CB9, 0xA90058EF, 0x52EFD7D, 
    0xE2C4E8AC, 0x4EEA4D3E, 0x7E1073D0, 0xD23ED642, 0x35D4C393, 0x99FA6601, 0xE8981257,
    0x44B6B7C5, 0xA35CA214, 0xF720786, 0x41984AB8, 0xEDB6EF2A, 0xA5CFAFB, 0xA6725F69,
    0xD7102B3F, 0x7B3E8EAD, 0x9CD49B7C, 0x30FA3EEE, 0xEFB91CC6, 0x4397B954, 0xA47DAC85, 
    0x8530917, 0x79317D41, 0xD51FD8D3, 0x32F5CD02, 0x9EDB6890, 0xD03125AE, 0x7C1F803C,
    0x9BF595ED, 0x37DB307F, 0x46B94429, 0xEA97E1BB, 0xD7DF46A, 0xA15351F8, 0x91A96F16, 
    0x3D87CA84, 0xDA6DDF55, 0x76437AC7, 0x7210E91, 0xAB0FAB03, 0x4CE5BED2, 0xE0CB1B40, 
    0xAE21567E, 0x20FF3EC, 0xE5E5E63D, 0x49CB43AF, 0x38A937F9, 0x9487926B, 0x736D87BA, 
    0xDF432228, 0xCDEAC3EA, 0x61C46678, 0x862E73A9, 0x2A00D63B, 0x5B62A26D, 0xF74C07FF, 
    0x10A6122E, 0xBC88B7BC, 0xF262FA82, 0x5E4C5F10, 0xB9A64AC1, 0x1588EF53, 0x64EA9B05, 
    0xC8C43E97, 0x2F2E2B46, 0x83008ED4, 0xB3FAB03A, 0x1FD415A8, 0xF83E0079, 0x5410A5EB, 
    0x2572D1BD, 0x895C742F, 0x6EB661FE, 0xC298C46C, 0x8C728952, 0x205C2CC0, 0xC7B63911, 
    0x6B989C83, 0x1AFAE8D5, 0xB6D44D47, 0x513E5896, 0xFD10FD04, 0x2253DF2C, 0x8E7D7ABE, 
    0x69976F6F, 0xC5B9CAFD, 0xB4DBBEAB, 0x18F51B39, 0xFF1F0EE8, 0x5331AB7A, 0x1DDBE644, 
    0xB1F543D6, 0x561F5607, 0xFA31F395, 0x8B5387C3, 0x277D2251, 0xC0973780, 0x6CB99212, 
    0x5C43ACFC, 0xF06D096E, 0x17871CBF, 0xBBA9B92D, 0xCACBCD7B, 0x66E568E9, 0x810F7D38, 
    0x2D21D8AA, 0x63CB9594, 0xCFE53006, 0x280F25D7, 0x84218045, 0xF543F413, 0x596D5181, 
    0xBE874450, 0x12A9E1C2, 0x894C7DB3, 0x2562D821, 0xC288CDF0, 0x6EA66862, 0x1FC41C34, 
    0xB3EAB9A6, 0x5400AC77, 0xF82E09E5, 0xB6C444DB, 0x1AEAE149, 0xFD00F498, 0x512E510A, 
    0x204C255C, 0x8C6280CE, 0x6B88951F, 0xC7A6308D, 0xF75C0E63, 0x5B72ABF1, 0xBC98BE20, 
    0x10B61BB2, 0x61D46FE4, 0xCDFACA76, 0x2A10DFA7, 0x863E7A35, 0xC8D4370B, 0x64FA9299,
    0x83108748, 0x2F3E22DA, 0x5E5C568C, 0xF272F31E, 0x1598E6CF, 0xB9B6435D, 0x66F56175, 
    0xCADBC4E7, 0x2D31D136, 0x811F74A4, 0xF07D00F2, 0x5C53A560, 0xBBB9B0B1, 0x17971523, 
    0x597D581D, 0xF553FD8F, 0x12B9E85E, 0xBE974DCC, 0xCFF5399A, 0x63DB9C08, 0x843189D9, 
    0x281F2C4B, 0x18E512A5, 0xB4CBB737, 0x5321A2E6, 0xFF0F0774, 0x8E6D7322, 0x2243D6B0, 
    0xC5A9C361, 0x698766F3, 0x276D2BCD, 0x8B438E5F, 0x6CA99B8E, 0xC0873E1C, 0xB1E54A4A,
    0x1DCBEFD8, 0xFA21FA09, 0x560F5F9B, 0x44A6BE59, 0xE8881BCB, 0xF620E1A, 0xA34CAB88, 
    0xD22EDFDE, 0x7E007A4C, 0x99EA6F9D, 0x35C4CA0F, 0x7B2E8731, 0xD70022A3, 0x30EA3772, 
    0x9CC492E0, 0xEDA6E6B6, 0x41884324, 0xA66256F5, 0xA4CF367, 0x3AB6CD89, 0x9698681B, 
    0x71727DCA, 0xDD5CD858, 0xAC3EAC0E, 0x10099C, 0xE7FA1C4D, 0x4BD4B9DF, 0x53EF4E1, 
    0xA9105173, 0x4EFA44A2, 0xE2D4E130, 0x93B69566, 0x3F9830F4, 0xD8722525, 0x745C80B7,
    0xAB1FA29F, 0x731070D, 0xE0DB12DC, 0x4CF5B74E, 0x3D97C318, 0x91B9668A, 0x7653735B, 
    0xDA7DD6C9, 0x94979BF7, 0x38B93E65, 0xDF532BB4, 0x737D8E26, 0x21FFA70, 0xAE315FE2, 
    0x49DB4A33, 0xE5F5EFA1, 0xD50FD14F, 0x792174DD, 0x9ECB610C, 0x32E5C49E, 0x4387B0C8, 
    0xEFA9155A, 0x843008B, 0xA46DA519, 0xEA87E827, 0x46A94DB5, 0xA1435864, 0xD6DFDF6, 
    0x7C0F89A0, 0xD0212C32, 0x37CB39E3, 0x9BE59C71}, 
      {0x00000000,0xE150AB9A, 0xD138AC53, 0x306807C9, 0xA27158A7, 0x4321F33D,0x7349F4F4, 
     0x92195F6E, 0x577A4A28, 0xB62AE1B2, 0x8642E67B,0x67124DE1, 0xF50B128F, 0x145BB915, 
     0x2433BEDC, 0xC5631546,0xAEF49450, 0x4FA43FCA, 0x7FCC3803, 0x9E9C9399, 0xC85CCF7,
     0xEDD5676D, 0xDDBD60A4, 0x3CEDCB3E, 0xF98EDE78, 0x18DE75E2,0x28B6722B, 0xC9E6D9B1, 
     0x5BFF86DF, 0xBAAF2D45, 0x8AC72A8C,0x6B978116, 0x5CE929A1, 0xBDB9823B, 0x8DD185F2, 
     0x6C812E68,0xFE987106, 0x1FC8DA9C, 0x2FA0DD55, 0xCEF076CF, 0xB936389,0xEAC3C813, 
     0xDAABCFDA, 0x3BFB6440, 0xA9E23B2E, 0x48B290B4,0x78DA977D, 0x998A3CE7, 0xF21DBDF1, 
     0x134D166B, 0x232511A2,0xC275BA38, 0x506CE556, 0xB13C4ECC, 0x81544905, 0x6004E29F,
     0xA567F7D9, 0x44375C43, 0x745F5B8A, 0x950FF010, 0x716AF7E,0xE64604E4, 0xD62E032D, 
     0x377EA8B7, 0xAB4BA924, 0x4A1B02BE,0x7A730577, 0x9B23AEED, 0x93AF183, 0xE86A5A19, 
     0xD8025DD0,0x3952F64A, 0xFC31E30C, 0x1D614896, 0x2D094F5F, 0xCC59E4C5,0x5E40BBAB, 
     0xBF101031, 0x8F7817F8, 0x6E28BC62, 0x5BF3D74,0xE4EF96EE, 0xD4879127, 0x35D73ABD, 
     0xA7CE65D3, 0x469ECE49,0x76F6C980, 0x97A6621A, 0x52C5775C, 0xB395DCC6, 0x83FDDB0F,
     0x62AD7095, 0xF0B42FFB, 0x11E48461, 0x218C83A8, 0xC0DC2832,0xF7A28085, 0x16F22B1F,
     0x269A2CD6, 0xC7CA874C, 0x55D3D822,0xB48373B8, 0x84EB7471, 0x65BBDFEB, 0xA0D8CAAD, 
     0x41886137,0x71E066FE, 0x90B0CD64, 0x2A9920A, 0xE3F93990, 0xD3913E59,0x32C195C3, 
     0x595614D5, 0xB806BF4F, 0x886EB886, 0x693E131C,0xFB274C72, 0x1A77E7E8, 0x2A1FE021, 
     0xCB4F4BBB, 0xE2C5EFD,0xEF7CF567, 0xDF14F2AE, 0x3E445934, 0xAC5D065A, 0x4D0DADC0,
     0x7D65AA09, 0x9C350193, 0x56975249, 0xB7C7F9D3, 0x87AFFE1A,0x66FF5580, 0xF4E60AEE, 
     0x15B6A174, 0x25DEA6BD, 0xC48E0D27,0x1ED1861, 0xE0BDB3FB, 0xD0D5B432, 0x31851FA8, 
     0xA39C40C6,0x42CCEB5C, 0x72A4EC95, 0x93F4470F, 0xF863C619, 0x19336D83,0x295B6A4A, 
     0xC80BC1D0, 0x5A129EBE, 0xBB423524, 0x8B2A32ED,0x6A7A9977, 0xAF198C31, 0x4E4927AB, 
     0x7E212062, 0x9F718BF8,0xD68D496, 0xEC387F0C, 0xDC5078C5, 0x3D00D35F, 0xA7E7BE8,
     0xEB2ED072, 0xDB46D7BB, 0x3A167C21, 0xA80F234F, 0x495F88D5,0x79378F1C, 0x98672486,
     0x5D0431C0, 0xBC549A5A, 0x8C3C9D93,0x6D6C3609, 0xFF756967, 0x1E25C2FD, 0x2E4DC534, 
     0xCF1D6EAE,0xA48AEFB8, 0x45DA4422, 0x75B243EB, 0x94E2E871, 0x6FBB71F,0xE7AB1C85, 
     0xD7C31B4C, 0x3693B0D6, 0xF3F0A590, 0x12A00E0A,0x22C809C3, 0xC398A259, 0x5181FD37, 
     0xB0D156AD, 0x80B95164,0x61E9FAFE, 0xFDDCFB6D, 0x1C8C50F7, 0x2CE4573E, 0xCDB4FCA4,
     0x5FADA3CA, 0xBEFD0850, 0x8E950F99, 0x6FC5A403, 0xAAA6B145,0x4BF61ADF, 0x7B9E1D16, 
     0x9ACEB68C, 0x8D7E9E2, 0xE9874278,0xD9EF45B1, 0x38BFEE2B, 0x53286F3D, 0xB278C4A7, 
     0x8210C36E,0x634068F4, 0xF159379A, 0x10099C00, 0x20619BC9, 0xC1313053,0x4522515, 
     0xE5028E8F, 0xD56A8946, 0x343A22DC, 0xA6237DB2,0x4773D628, 0x771BD1E1, 0x964B7A7B, 
     0xA135D2CC, 0x40657956,0x700D7E9F, 0x915DD505, 0x3448A6B, 0xE21421F1, 0xD27C2638,
     0x332C8DA2, 0xF64F98E4, 0x171F337E, 0x277734B7, 0xC6279F2D,0x543EC043, 0xB56E6BD9, 
     0x85066C10, 0x6456C78A, 0xFC1469C,0xEE91ED06, 0xDEF9EACF, 0x3FA94155, 0xADB01E3B,
     0x4CE0B5A1,0x7C88B268, 0x9DD819F2, 0x58BB0CB4, 0xB9EBA72E, 0x8983A0E7,0x68D30B7D, 
     0xFACA5413, 0x1B9AFF89, 0x2BF2F840, 0xCAA253DA}, 
     {0x00000000,0x579A9D0D, 0xAE343B1B, 0xF9AEA616, 0x5C697636, 0xBF3EB3B,0xF25D4D2D, 
     0xA5C7D020, 0xB8D2EC6C, 0xEF487161, 0x16E6D777,0x417C4A7A, 0xE4BB9A5A, 0xB3210757, 
     0x4A8FA141, 0x1D153C4C,0x70A5D9D9, 0x273F44D4, 0xDE91E2C2, 0x890B7FCF, 0x2CCCAFEF,
     0x7B5632E2, 0x82F894F4, 0xD56209F9, 0xC87735B5, 0x9FEDA8B8,0x66430EAE, 0x31D993A3, 
     0x941E4383, 0xC384DE8E, 0x3A2A7898,0x6DB0E595, 0xF3D349D5, 0xA449D4D8, 0x5DE772CE,
     0xA7DEFC3,0xAFBA3FE3, 0xF820A2EE, 0x18E04F8, 0x561499F5, 0x4B01A5B9,0x1C9B38B4, 
     0xE5359EA2, 0xB2AF03AF, 0x1768D38F, 0x40F24E82,0xB95CE894, 0xEEC67599, 0x8376900C, 
     0xD4EC0D01, 0x2D42AB17,0x7AD8361A, 0xDF1FE63A, 0x88857B37, 0x712BDD21, 0x26B1402C,
     0x3BA47C60, 0x6C3EE16D, 0x9590477B, 0xC20ADA76, 0x67CD0A56,0x3057975B, 0xC9F9314D, 
     0x9E63AC40, 0xF53E69CC, 0xA2A4F4C1,0x5B0A52D7, 0xC90CFDA, 0xA9571FFA, 0xFECD82F7, 
     0x76324E1,0x50F9B9EC, 0x4DEC85A0, 0x1A7618AD, 0xE3D8BEBB, 0xB44223B6,0x1185F396, 
     0x461F6E9B, 0xBFB1C88D, 0xE82B5580, 0x859BB015,0xD2012D18, 0x2BAF8B0E, 0x7C351603,
     0xD9F2C623, 0x8E685B2E,0x77C6FD38, 0x205C6035, 0x3D495C79, 0x6AD3C174, 0x937D6762,
     0xC4E7FA6F, 0x61202A4F, 0x36BAB742, 0xCF141154, 0x988E8C59,0x6ED2019, 0x5177BD14, 
     0xA8D91B02, 0xFF43860F, 0x5A84562F,0xD1ECB22, 0xF4B06D34, 0xA32AF039, 0xBE3FCC75, 
     0xE9A55178,0x100BF76E, 0x47916A63, 0xE256BA43, 0xB5CC274E, 0x4C628158,0x1BF81C55, 
     0x7648F9C0, 0x21D264CD, 0xD87CC2DB, 0x8FE65FD6,0x2A218FF6, 0x7DBB12FB, 0x8415B4ED,
     0xD38F29E0, 0xCE9A15AC,0x990088A1, 0x60AE2EB7, 0x3734B3BA, 0x92F3639A, 0xC569FE97,
     0x3CC75881, 0x6B5DC58C, 0xF9E428FE, 0xAE7EB5F3, 0x57D013E5,0x4A8EE8, 0xA58D5EC8, 
     0xF217C3C5, 0xBB965D3, 0x5C23F8DE,0x4136C492, 0x16AC599F, 0xEF02FF89, 0xB8986284, 
     0x1D5FB2A4,0x4AC52FA9, 0xB36B89BF, 0xE4F114B2, 0x8941F127, 0xDEDB6C2A,0x2775CA3C, 
     0x70EF5731, 0xD5288711, 0x82B21A1C, 0x7B1CBC0A,0x2C862107, 0x31931D4B, 0x66098046, 
     0x9FA72650, 0xC83DBB5D,0x6DFA6B7D, 0x3A60F670, 0xC3CE5066, 0x9454CD6B, 0xA37612B,
    0x5DADFC26, 0xA4035A30, 0xF399C73D, 0x565E171D, 0x1C48A10,0xF86A2C06, 0xAFF0B10B, 
    0xB2E58D47, 0xE57F104A, 0x1CD1B65C,0x4B4B2B51, 0xEE8CFB71, 0xB916667C, 0x40B8C06A,
    0x17225D67,0x7A92B8F2, 0x2D0825FF, 0xD4A683E9, 0x833C1EE4, 0x26FBCEC4,0x716153C9,
    0x88CFF5DF, 0xDF5568D2, 0xC240549E, 0x95DAC993,0x6C746F85, 0x3BEEF288, 0x9E2922A8, 
    0xC9B3BFA5, 0x301D19B3,0x678784BE, 0xCDA4132, 0x5B40DC3F, 0xA2EE7A29, 0xF574E724,
    0x50B33704, 0x729AA09, 0xFE870C1F, 0xA91D9112, 0xB408AD5E,0xE3923053, 0x1A3C9645, 
    0x4DA60B48, 0xE861DB68, 0xBFFB4665,0x4655E073, 0x11CF7D7E, 0x7C7F98EB, 0x2BE505E6, 
    0xD24BA3F0,0x85D13EFD, 0x2016EEDD, 0x778C73D0, 0x8E22D5C6, 0xD9B848CB,0xC4AD7487,
    0x9337E98A, 0x6A994F9C, 0x3D03D291, 0x98C402B1,0xCF5E9FBC, 0x36F039AA, 0x616AA4A7, 
    0xFF0908E7, 0xA89395EA,0x513D33FC, 0x6A7AEF1, 0xA3607ED1, 0xF4FAE3DC, 0xD5445CA,
    0x5ACED8C7, 0x47DBE48B, 0x10417986, 0xE9EFDF90, 0xBE75429D,0x1BB292BD, 0x4C280FB0,
    0xB586A9A6, 0xE21C34AB, 0x8FACD13E,0xD8364C33, 0x2198EA25, 0x76027728, 0xD3C5A708,
    0x845F3A05,0x7DF19C13, 0x2A6B011E, 0x377E3D52, 0x60E4A05F, 0x994A0649,0xCED09B44, 
    0x6B174B64, 0x3C8DD669, 0xC523707F, 0x92B9ED72}, 
    { 0x00000000, 0x5805C96C, 0xB00A92D9, 0xE80F5BB5, 0x738CDED5, 0x2B8917B9, 0xC3864C0C,
    0x9B838560, 0xF58147CD, 0xAD848EA1, 0x458BD514, 0x1D8E1C78, 0x860D9918, 0xDE085074, 
    0x36070BC1, 0x6E02C2AD, 0xF99A75FC, 0xA19FBC90, 0x4990E725, 0x11952E49, 0x8A16AB29, 
    0xD2136245, 0x3A1C39F0, 0x6219F09C, 0xC1B3231, 0x541EFB5D, 0xBC11A0E8, 0xE4146984, 
    0x7F97ECE4, 0x27922588, 0xCF9D7E3D, 0x9798B751, 0xE1AC119E, 0xB9A9D8F2, 0x51A68347, 
    0x9A34A2B, 0x9220CF4B, 0xCA250627, 0x222A5D92, 0x7A2F94FE, 0x142D5653, 0x4C289F3F, 
    0xA427C48A, 0xFC220DE6, 0x67A18886, 0x3FA441EA, 0xD7AB1A5F, 0x8FAED333, 0x18366462,
    0x4033AD0E, 0xA83CF6BB, 0xF0393FD7, 0x6BBABAB7, 0x33BF73DB, 0xDBB0286E, 0x83B5E102, 
    0xEDB723AF, 0xB5B2EAC3, 0x5DBDB176, 0x5B8781A, 0x9E3BFD7A, 0xC63E3416, 0x2E316FA3, 
    0x7634A6CF, 0xD1C0D95A, 0x89C51036, 0x61CA4B83, 0x39CF82EF, 0xA24C078F, 0xFA49CEE3,
    0x12469556, 0x4A435C3A, 0x24419E97, 0x7C4457FB, 0x944B0C4E, 0xCC4EC522, 0x57CD4042, 
    0xFC8892E, 0xE7C7D29B, 0xBFC21BF7, 0x285AACA6, 0x705F65CA, 0x98503E7F, 0xC055F713, 
    0x5BD67273, 0x3D3BB1F, 0xEBDCE0AA, 0xB3D929C6, 0xDDDBEB6B, 0x85DE2207, 0x6DD179B2, 
    0x35D4B0DE, 0xAE5735BE, 0xF652FCD2, 0x1E5DA767, 0x46586E0B, 0x306CC8C4, 0x686901A8,
    0x80665A1D, 0xD8639371, 0x43E01611, 0x1BE5DF7D, 0xF3EA84C8, 0xABEF4DA4, 0xC5ED8F09, 
    0x9DE84665, 0x75E71DD0, 0x2DE2D4BC, 0xB66151DC, 0xEE6498B0, 0x66BC305, 0x5E6E0A69, 
    0xC9F6BD38, 0x91F37454, 0x79FC2FE1, 0x21F9E68D, 0xBA7A63ED, 0xE27FAA81, 0xA70F134,
    0x52753858, 0x3C77FAF5, 0x64723399, 0x8C7D682C, 0xD478A140, 0x4FFB2420, 0x17FEED4C, 
    0xFFF1B6F9, 0xA7F47F95, 0xA281B3B5, 0xFA847AD9, 0x128B216C, 0x4A8EE800, 0xD10D6D60, 
    0x8908A40C, 0x6107FFB9, 0x390236D5, 0x5700F478, 0xF053D14, 0xE70A66A1, 0xBF0FAFCD,
    0x248C2AAD, 0x7C89E3C1, 0x9486B874, 0xCC837118, 0x5B1BC649, 0x31E0F25, 0xEB115490, 
    0xB3149DFC, 0x2897189C, 0x7092D1F0, 0x989D8A45, 0xC0984329, 0xAE9A8184, 0xF69F48E8, 
    0x1E90135D, 0x4695DA31, 0xDD165F51, 0x8513963D, 0x6D1CCD88, 0x351904E4, 0x432DA22B,
     0x1B286B47, 0xF32730F2, 0xAB22F99E, 0x30A17CFE, 0x68A4B592, 0x80ABEE27, 0xD8AE274B,
     0xB6ACE5E6, 0xEEA92C8A, 0x6A6773F, 0x5EA3BE53, 0xC5203B33, 0x9D25F25F, 0x752AA9EA, 
     0x2D2F6086, 0xBAB7D7D7, 0xE2B21EBB, 0xABD450E, 0x52B88C62, 0xC93B0902, 0x913EC06E,
     0x79319BDB, 0x213452B7, 0x4F36901A, 0x17335976, 0xFF3C02C3, 0xA739CBAF, 0x3CBA4ECF,
     0x64BF87A3, 0x8CB0DC16, 0xD4B5157A, 0x73416AEF, 0x2B44A383, 0xC34BF836, 0x9B4E315A,
     0xCDB43A, 0x58C87D56, 0xB0C726E3, 0xE8C2EF8F, 0x86C02D22, 0xDEC5E44E, 0x36CABFFB, 
     0x6ECF7697, 0xF54CF3F7, 0xAD493A9B, 0x4546612E, 0x1D43A842, 0x8ADB1F13, 0xD2DED67F,
     0x3AD18DCA, 0x62D444A6, 0xF957C1C6, 0xA15208AA, 0x495D531F, 0x11589A73, 0x7F5A58DE, 
     0x275F91B2, 0xCF50CA07, 0x9755036B, 0xCD6860B, 0x54D34F67, 0xBCDC14D2, 0xE4D9DDBE, 
     0x92ED7B71, 0xCAE8B21D, 0x22E7E9A8, 0x7AE220C4, 0xE161A5A4, 0xB9646CC8, 0x516B377D,
     0x96EFE11, 0x676C3CBC, 0x3F69F5D0, 0xD766AE65, 0x8F636709, 0x14E0E269, 0x4CE52B05, 
     0xA4EA70B0, 0xFCEFB9DC, 0x6B770E8D, 0x3372C7E1, 0xDB7D9C54, 0x83785538, 0x18FBD058, 
     0x40FE1934, 0xA8F14281, 0xF0F48BED, 0x9EF64940, 0xC6F3802C, 0x2EFCDB99, 0x76F912F5,
     0xED7A9795, 0xB57F5EF9, 0x5D70054C, 0x575CC20}, 
 {      0x00000000, 0x3D6029B0, 0x7AC05360, 0x47A07AD0,
        0xF580A6C0, 0xC8E08F70, 0x8F40F5A0, 0xB220DC10,
        0x30704BC1, 0x0D106271, 0x4AB018A1, 0x77D03111,
        0xC5F0ED01, 0xF890C4B1, 0xBF30BE61, 0x825097D1,
        0x60E09782, 0x5D80BE32, 0x1A20C4E2, 0x2740ED52,
        0x95603142, 0xA80018F2, 0xEFA06222, 0xD2C04B92,
        0x5090DC43, 0x6DF0F5F3, 0x2A508F23, 0x1730A693,
        0xA5107A83, 0x98705333, 0xDFD029E3, 0xE2B00053,
        0xC1C12F04, 0xFCA106B4, 0xBB017C64, 0x866155D4,
        0x344189C4, 0x0921A074, 0x4E81DAA4, 0x73E1F314,
        0xF1B164C5, 0xCCD14D75, 0x8B7137A5, 0xB6111E15,
        0x0431C205, 0x3951EBB5, 0x7EF19165, 0x4391B8D5,
        0xA121B886, 0x9C419136, 0xDBE1EBE6, 0xE681C256,
        0x54A11E46, 0x69C137F6, 0x2E614D26, 0x13016496,
        0x9151F347, 0xAC31DAF7, 0xEB91A027, 0xD6F18997,
        0x64D15587, 0x59B17C37, 0x1E1106E7, 0x23712F57,
        0x58F35849, 0x659371F9, 0x22330B29, 0x1F532299,
        0xAD73FE89, 0x9013D739, 0xD7B3ADE9, 0xEAD38459,
        0x68831388, 0x55E33A38, 0x124340E8, 0x2F236958,
        0x9D03B548, 0xA0639CF8, 0xE7C3E628, 0xDAA3CF98,
        0x3813CFCB, 0x0573E67B, 0x42D39CAB, 0x7FB3B51B,
        0xCD93690B, 0xF0F340BB, 0xB7533A6B, 0x8A3313DB,
        0x0863840A, 0x3503ADBA, 0x72A3D76A, 0x4FC3FEDA,
        0xFDE322CA, 0xC0830B7A, 0x872371AA, 0xBA43581A,
        0x9932774D, 0xA4525EFD, 0xE3F2242D, 0xDE920D9D,
        0x6CB2D18D, 0x51D2F83D, 0x167282ED, 0x2B12AB5D,
        0xA9423C8C, 0x9422153C, 0xD3826FEC, 0xEEE2465C,
        0x5CC29A4C, 0x61A2B3FC, 0x2602C92C, 0x1B62E09C,
        0xF9D2E0CF, 0xC4B2C97F, 0x8312B3AF, 0xBE729A1F,
        0x0C52460F, 0x31326FBF, 0x7692156F, 0x4BF23CDF,
        0xC9A2AB0E, 0xF4C282BE, 0xB362F86E, 0x8E02D1DE,
        0x3C220DCE, 0x0142247E, 0x46E25EAE, 0x7B82771E,
        0xB1E6B092, 0x8C869922, 0xCB26E3F2, 0xF646CA42,
        0x44661652, 0x79063FE2, 0x3EA64532, 0x03C66C82,
        0x8196FB53, 0xBCF6D2E3, 0xFB56A833, 0xC6368183,
        0x74165D93, 0x49767423, 0x0ED60EF3, 0x33B62743,
        0xD1062710, 0xEC660EA0, 0xABC67470, 0x96A65DC0,
        0x248681D0, 0x19E6A860, 0x5E46D2B0, 0x6326FB00,
        0xE1766CD1, 0xDC164561, 0x9BB63FB1, 0xA6D61601,
        0x14F6CA11, 0x2996E3A1, 0x6E369971, 0x5356B0C1,
        0x70279F96, 0x4D47B626, 0x0AE7CCF6, 0x3787E546,
        0x85A73956, 0xB8C710E6, 0xFF676A36, 0xC2074386,
        0x4057D457, 0x7D37FDE7, 0x3A978737, 0x07F7AE87,
        0xB5D77297, 0x88B75B27, 0xCF1721F7, 0xF2770847,
        0x10C70814, 0x2DA721A4, 0x6A075B74, 0x576772C4,
        0xE547AED4, 0xD8278764, 0x9F87FDB4, 0xA2E7D404,
        0x20B743D5, 0x1DD76A65, 0x5A7710B5, 0x67173905,
        0xD537E515, 0xE857CCA5, 0xAFF7B675, 0x92979FC5,
        0xE915E8DB, 0xD475C16B, 0x93D5BBBB, 0xAEB5920B,
        0x1C954E1B, 0x21F567AB, 0x66551D7B, 0x5B3534CB,
        0xD965A31A, 0xE4058AAA, 0xA3A5F07A, 0x9EC5D9CA,
        0x2CE505DA, 0x11852C6A, 0x562556BA, 0x6B457F0A,
        0x89F57F59, 0xB49556E9, 0xF3352C39, 0xCE550589,
        0x7C75D999, 0x4115F029, 0x06B58AF9, 0x3BD5A349,
        0xB9853498, 0x84E51D28, 0xC34567F8, 0xFE254E48,
        0x4C059258, 0x7165BBE8, 0x36C5C138, 0x0BA5E888,
        0x28D4C7DF, 0x15B4EE6F, 0x521494BF, 0x6F74BD0F,
        0xDD54611F, 0xE03448AF, 0xA794327F, 0x9AF41BCF,
        0x18A48C1E, 0x25C4A5AE, 0x6264DF7E, 0x5F04F6CE,
        0xED242ADE, 0xD044036E, 0x97E479BE, 0xAA84500E,
        0x4834505D, 0x755479ED, 0x32F4033D, 0x0F942A8D,
        0xBDB4F69D, 0x80D4DF2D, 0xC774A5FD, 0xFA148C4D,
        0x78441B9C, 0x4524322C, 0x028448FC, 0x3FE4614C,
        0x8DC4BD5C, 0xB0A494EC, 0xF704EE3C, 0xCA64C78C
    }, {
        0x00000000, 0xCB5CD3A5, 0x4DC8A10B, 0x869472AE,
        0x9B914216, 0x50CD91B3, 0xD659E31D, 0x1D0530B8,
        0xEC53826D, 0x270F51C8, 0xA19B2366, 0x6AC7F0C3,
        0x77C2C07B, 0xBC9E13DE, 0x3A0A6170, 0xF156B2D5,
        0x03D6029B, 0xC88AD13E, 0x4E1EA390, 0x85427035,
        0x9847408D, 0x531B9328, 0xD58FE186, 0x1ED33223,
        0xEF8580F6, 0x24D95353, 0xA24D21FD, 0x6911F258,
        0x7414C2E0, 0xBF481145, 0x39DC63EB, 0xF280B04E,
        0x07AC0536, 0xCCF0D693, 0x4A64A43D, 0x81387798,
        0x9C3D4720, 0x57619485, 0xD1F5E62B, 0x1AA9358E,
        0xEBFF875B, 0x20A354FE, 0xA6372650, 0x6D6BF5F5,
        0x706EC54D, 0xBB3216E8, 0x3DA66446, 0xF6FAB7E3,
        0x047A07AD, 0xCF26D408, 0x49B2A6A6, 0x82EE7503,
        0x9FEB45BB, 0x54B7961E, 0xD223E4B0, 0x197F3715,
        0xE82985C0, 0x23755665, 0xA5E124CB, 0x6EBDF76E,
        0x73B8C7D6, 0xB8E41473, 0x3E7066DD, 0xF52CB578,
        0x0F580A6C, 0xC404D9C9, 0x4290AB67, 0x89CC78C2,
        0x94C9487A, 0x5F959BDF, 0xD901E971, 0x125D3AD4,
        0xE30B8801, 0x28575BA4, 0xAEC3290A, 0x659FFAAF,
        0x789ACA17, 0xB3C619B2, 0x35526B1C, 0xFE0EB8B9,
        0x0C8E08F7, 0xC7D2DB52, 0x4146A9FC, 0x8A1A7A59,
        0x971F4AE1, 0x5C439944, 0xDAD7EBEA, 0x118B384F,
        0xE0DD8A9A, 0x2B81593F, 0xAD152B91, 0x6649F834,
        0x7B4CC88C, 0xB0101B29, 0x36846987, 0xFDD8BA22,
        0x08F40F5A, 0xC3A8DCFF, 0x453CAE51, 0x8E607DF4,
        0x93654D4C, 0x58399EE9, 0xDEADEC47, 0x15F13FE2,
        0xE4A78D37, 0x2FFB5E92, 0xA96F2C3C, 0x6233FF99,
        0x7F36CF21, 0xB46A1C84, 0x32FE6E2A, 0xF9A2BD8F,
        0x0B220DC1, 0xC07EDE64, 0x46EAACCA, 0x8DB67F6F,
        0x90B34FD7, 0x5BEF9C72, 0xDD7BEEDC, 0x16273D79,
        0xE7718FAC, 0x2C2D5C09, 0xAAB92EA7, 0x61E5FD02,
        0x7CE0CDBA, 0xB7BC1E1F, 0x31286CB1, 0xFA74BF14,
        0x1EB014D8, 0xD5ECC77D, 0x5378B5D3, 0x98246676,
        0x852156CE, 0x4E7D856B, 0xC8E9F7C5, 0x03B52460,
        0xF2E396B5, 0x39BF4510, 0xBF2B37BE, 0x7477E41B,
        0x6972D4A3, 0xA22E0706, 0x24BA75A8, 0xEFE6A60D,
        0x1D661643, 0xD63AC5E6, 0x50AEB748, 0x9BF264ED,
        0x86F75455, 0x4DAB87F0, 0xCB3FF55E, 0x006326FB,
        0xF135942E, 0x3A69478B, 0xBCFD3525, 0x77A1E680,
        0x6AA4D638, 0xA1F8059D, 0x276C7733, 0xEC30A496,
        0x191C11EE, 0xD240C24B, 0x54D4B0E5, 0x9F886340,
        0x828D53F8, 0x49D1805D, 0xCF45F2F3, 0x04192156,
        0xF54F9383, 0x3E134026, 0xB8873288, 0x73DBE12D,
        0x6EDED195, 0xA5820230, 0x2316709E, 0xE84AA33B,
        0x1ACA1375, 0xD196C0D0, 0x5702B27E, 0x9C5E61DB,
        0x815B5163, 0x4A0782C6, 0xCC93F068, 0x07CF23CD,
        0xF6999118, 0x3DC542BD, 0xBB513013, 0x700DE3B6,
        0x6D08D30E, 0xA65400AB, 0x20C07205, 0xEB9CA1A0,
        0x11E81EB4, 0xDAB4CD11, 0x5C20BFBF, 0x977C6C1A,
        0x8A795CA2, 0x41258F07, 0xC7B1FDA9, 0x0CED2E0C,
        0xFDBB9CD9, 0x36E74F7C, 0xB0733DD2, 0x7B2FEE77,
        0x662ADECF, 0xAD760D6A, 0x2BE27FC4, 0xE0BEAC61,
        0x123E1C2F, 0xD962CF8A, 0x5FF6BD24, 0x94AA6E81,
        0x89AF5E39, 0x42F38D9C, 0xC467FF32, 0x0F3B2C97,
        0xFE6D9E42, 0x35314DE7, 0xB3A53F49, 0x78F9ECEC,
        0x65FCDC54, 0xAEA00FF1, 0x28347D5F, 0xE368AEFA,
        0x16441B82, 0xDD18C827, 0x5B8CBA89, 0x90D0692C,
        0x8DD55994, 0x46898A31, 0xC01DF89F, 0x0B412B3A,
        0xFA1799EF, 0x314B4A4A, 0xB7DF38E4, 0x7C83EB41,
        0x6186DBF9, 0xAADA085C, 0x2C4E7AF2, 0xE712A957,
        0x15921919, 0xDECECABC, 0x585AB812, 0x93066BB7,
        0x8E035B0F, 0x455F88AA, 0xC3CBFA04, 0x089729A1,
        0xF9C19B74, 0x329D48D1, 0xB4093A7F, 0x7F55E9DA,
        0x6250D962, 0xA90C0AC7, 0x2F987869, 0xE4C4ABCC
    }, {
        0x00000000, 0xA6770BB4, 0x979F1129, 0x31E81A9D,
        0xF44F2413, 0x52382FA7, 0x63D0353A, 0xC5A73E8E,
        0x33EF4E67, 0x959845D3, 0xA4705F4E, 0x020754FA,
        0xC7A06A74, 0x61D761C0, 0x503F7B5D, 0xF64870E9,
        0x67DE9CCE, 0xC1A9977A, 0xF0418DE7, 0x56368653,
        0x9391B8DD, 0x35E6B369, 0x040EA9F4, 0xA279A240,
        0x5431D2A9, 0xF246D91D, 0xC3AEC380, 0x65D9C834,
        0xA07EF6BA, 0x0609FD0E, 0x37E1E793, 0x9196EC27,
        0xCFBD399C, 0x69CA3228, 0x582228B5, 0xFE552301,
        0x3BF21D8F, 0x9D85163B, 0xAC6D0CA6, 0x0A1A0712,
        0xFC5277FB, 0x5A257C4F, 0x6BCD66D2, 0xCDBA6D66,
        0x081D53E8, 0xAE6A585C, 0x9F8242C1, 0x39F54975,
        0xA863A552, 0x0E14AEE6, 0x3FFCB47B, 0x998BBFCF,
        0x5C2C8141, 0xFA5B8AF5, 0xCBB39068, 0x6DC49BDC,
        0x9B8CEB35, 0x3DFBE081, 0x0C13FA1C, 0xAA64F1A8,
        0x6FC3CF26, 0xC9B4C492, 0xF85CDE0F, 0x5E2BD5BB,
        0x440B7579, 0xE27C7ECD, 0xD3946450, 0x75E36FE4,
        0xB044516A, 0x16335ADE, 0x27DB4043, 0x81AC4BF7,
        0x77E43B1E, 0xD19330AA, 0xE07B2A37, 0x460C2183,
        0x83AB1F0D, 0x25DC14B9, 0x14340E24, 0xB2430590,
        0x23D5E9B7, 0x85A2E203, 0xB44AF89E, 0x123DF32A,
        0xD79ACDA4, 0x71EDC610, 0x4005DC8D, 0xE672D739,
        0x103AA7D0, 0xB64DAC64, 0x87A5B6F9, 0x21D2BD4D,
        0xE47583C3, 0x42028877, 0x73EA92EA, 0xD59D995E,
        0x8BB64CE5, 0x2DC14751, 0x1C295DCC, 0xBA5E5678,
        0x7FF968F6, 0xD98E6342, 0xE86679DF, 0x4E11726B,
        0xB8590282, 0x1E2E0936, 0x2FC613AB, 0x89B1181F,
        0x4C162691, 0xEA612D25, 0xDB8937B8, 0x7DFE3C0C,
        0xEC68D02B, 0x4A1FDB9F, 0x7BF7C102, 0xDD80CAB6,
        0x1827F438, 0xBE50FF8C, 0x8FB8E511, 0x29CFEEA5,
        0xDF879E4C, 0x79F095F8, 0x48188F65, 0xEE6F84D1,
        0x2BC8BA5F, 0x8DBFB1EB, 0xBC57AB76, 0x1A20A0C2,
        0x8816EAF2, 0x2E61E146, 0x1F89FBDB, 0xB9FEF06F,
        0x7C59CEE1, 0xDA2EC555, 0xEBC6DFC8, 0x4DB1D47C,
        0xBBF9A495, 0x1D8EAF21, 0x2C66B5BC, 0x8A11BE08,
        0x4FB68086, 0xE9C18B32, 0xD82991AF, 0x7E5E9A1B,
        0xEFC8763C, 0x49BF7D88, 0x78576715, 0xDE206CA1,
        0x1B87522F, 0xBDF0599B, 0x8C184306, 0x2A6F48B2,
        0xDC27385B, 0x7A5033EF, 0x4BB82972, 0xEDCF22C6,
        0x28681C48, 0x8E1F17FC, 0xBFF70D61, 0x198006D5,
        0x47ABD36E, 0xE1DCD8DA, 0xD034C247, 0x7643C9F3,
        0xB3E4F77D, 0x1593FCC9, 0x247BE654, 0x820CEDE0,
        0x74449D09, 0xD23396BD, 0xE3DB8C20, 0x45AC8794,
        0x800BB91A, 0x267CB2AE, 0x1794A833, 0xB1E3A387,
        0x20754FA0, 0x86024414, 0xB7EA5E89, 0x119D553D,
        0xD43A6BB3, 0x724D6007, 0x43A57A9A, 0xE5D2712E,
        0x139A01C7, 0xB5ED0A73, 0x840510EE, 0x22721B5A,
        0xE7D525D4, 0x41A22E60, 0x704A34FD, 0xD63D3F49,
        0xCC1D9F8B, 0x6A6A943F, 0x5B828EA2, 0xFDF58516,
        0x3852BB98, 0x9E25B02C, 0xAFCDAAB1, 0x09BAA105,
        0xFFF2D1EC, 0x5985DA58, 0x686DC0C5, 0xCE1ACB71,
        0x0BBDF5FF, 0xADCAFE4B, 0x9C22E4D6, 0x3A55EF62,
        0xABC30345, 0x0DB408F1, 0x3C5C126C, 0x9A2B19D8,
        0x5F8C2756, 0xF9FB2CE2, 0xC813367F, 0x6E643DCB,
        0x982C4D22, 0x3E5B4696, 0x0FB35C0B, 0xA9C457BF,
        0x6C636931, 0xCA146285, 0xFBFC7818, 0x5D8B73AC,
        0x03A0A617, 0xA5D7ADA3, 0x943FB73E, 0x3248BC8A,
        0xF7EF8204, 0x519889B0, 0x6070932D, 0xC6079899,
        0x304FE870, 0x9638E3C4, 0xA7D0F959, 0x01A7F2ED,
        0xC400CC63, 0x6277C7D7, 0x539FDD4A, 0xF5E8D6FE,
        0x647E3AD9, 0xC209316D, 0xF3E12BF0, 0x55962044,
        0x90311ECA, 0x3646157E, 0x07AE0FE3, 0xA1D90457,
        0x579174BE, 0xF1E67F0A, 0xC00E6597, 0x66796E23,
        0xA3DE50AD, 0x05A95B19, 0x34414184, 0x92364A30
    }, {
        0x00000000, 0xCCAA009E, 0x4225077D, 0x8E8F07E3,
        0x844A0EFA, 0x48E00E64, 0xC66F0987, 0x0AC50919,
        0xD3E51BB5, 0x1F4F1B2B, 0x91C01CC8, 0x5D6A1C56,
        0x57AF154F, 0x9B0515D1, 0x158A1232, 0xD92012AC,
        0x7CBB312B, 0xB01131B5, 0x3E9E3656, 0xF23436C8,
        0xF8F13FD1, 0x345B3F4F, 0xBAD438AC, 0x767E3832,
        0xAF5E2A9E, 0x63F42A00, 0xED7B2DE3, 0x21D12D7D,
        0x2B142464, 0xE7BE24FA, 0x69312319, 0xA59B2387,
        0xF9766256, 0x35DC62C8, 0xBB53652B, 0x77F965B5,
        0x7D3C6CAC, 0xB1966C32, 0x3F196BD1, 0xF3B36B4F,
        0x2A9379E3, 0xE639797D, 0x68B67E9E, 0xA41C7E00,
        0xAED97719, 0x62737787, 0xECFC7064, 0x205670FA,
        0x85CD537D, 0x496753E3, 0xC7E85400, 0x0B42549E,
        0x01875D87, 0xCD2D5D19, 0x43A25AFA, 0x8F085A64,
        0x562848C8, 0x9A824856, 0x140D4FB5, 0xD8A74F2B,
        0xD2624632, 0x1EC846AC, 0x9047414F, 0x5CED41D1,
        0x299DC2ED, 0xE537C273, 0x6BB8C590, 0xA712C50E,
        0xADD7CC17, 0x617DCC89, 0xEFF2CB6A, 0x2358CBF4,
        0xFA78D958, 0x36D2D9C6, 0xB85DDE25, 0x74F7DEBB,
        0x7E32D7A2, 0xB298D73C, 0x3C17D0DF, 0xF0BDD041,
        0x5526F3C6, 0x998CF358, 0x1703F4BB, 0xDBA9F425,
        0xD16CFD3C, 0x1DC6FDA2, 0x9349FA41, 0x5FE3FADF,
        0x86C3E873, 0x4A69E8ED, 0xC4E6EF0E, 0x084CEF90,
        0x0289E689, 0xCE23E617, 0x40ACE1F4, 0x8C06E16A,
        0xD0EBA0BB, 0x1C41A025, 0x92CEA7C6, 0x5E64A758,
        0x54A1AE41, 0x980BAEDF, 0x1684A93C, 0xDA2EA9A2,
        0x030EBB0E, 0xCFA4BB90, 0x412BBC73, 0x8D81BCED,
        0x8744B5F4, 0x4BEEB56A, 0xC561B289, 0x09CBB217,
        0xAC509190, 0x60FA910E, 0xEE7596ED, 0x22DF9673,
        0x281A9F6A, 0xE4B09FF4, 0x6A3F9817, 0xA6959889,
        0x7FB58A25, 0xB31F8ABB, 0x3D908D58, 0xF13A8DC6,
        0xFBFF84DF, 0x37558441, 0xB9DA83A2, 0x7570833C,
        0x533B85DA, 0x9F918544, 0x111E82A7, 0xDDB48239,
        0xD7718B20, 0x1BDB8BBE, 0x95548C5D, 0x59FE8CC3,
        0x80DE9E6F, 0x4C749EF1, 0xC2FB9912, 0x0E51998C,
        0x04949095, 0xC83E900B, 0x46B197E8, 0x8A1B9776,
        0x2F80B4F1, 0xE32AB46F, 0x6DA5B38C, 0xA10FB312,
        0xABCABA0B, 0x6760BA95, 0xE9EFBD76, 0x2545BDE8,
        0xFC65AF44, 0x30CFAFDA, 0xBE40A839, 0x72EAA8A7,
        0x782FA1BE, 0xB485A120, 0x3A0AA6C3, 0xF6A0A65D,
        0xAA4DE78C, 0x66E7E712, 0xE868E0F1, 0x24C2E06F,
        0x2E07E976, 0xE2ADE9E8, 0x6C22EE0B, 0xA088EE95,
        0x79A8FC39, 0xB502FCA7, 0x3B8DFB44, 0xF727FBDA,
        0xFDE2F2C3, 0x3148F25D, 0xBFC7F5BE, 0x736DF520,
        0xD6F6D6A7, 0x1A5CD639, 0x94D3D1DA, 0x5879D144,
        0x52BCD85D, 0x9E16D8C3, 0x1099DF20, 0xDC33DFBE,
        0x0513CD12, 0xC9B9CD8C, 0x4736CA6F, 0x8B9CCAF1,
        0x8159C3E8, 0x4DF3C376, 0xC37CC495, 0x0FD6C40B,
        0x7AA64737, 0xB60C47A9, 0x3883404A, 0xF42940D4,
        0xFEEC49CD, 0x32464953, 0xBCC94EB0, 0x70634E2E,
        0xA9435C82, 0x65E95C1C, 0xEB665BFF, 0x27CC5B61,
        0x2D095278, 0xE1A352E6, 0x6F2C5505, 0xA386559B,
        0x061D761C, 0xCAB77682, 0x44387161, 0x889271FF,
        0x825778E6, 0x4EFD7878, 0xC0727F9B, 0x0CD87F05,
        0xD5F86DA9, 0x19526D37, 0x97DD6AD4, 0x5B776A4A,
        0x51B26353, 0x9D1863CD, 0x1397642E, 0xDF3D64B0,
        0x83D02561, 0x4F7A25FF, 0xC1F5221C, 0x0D5F2282,
        0x079A2B9B, 0xCB302B05, 0x45BF2CE6, 0x89152C78,
        0x50353ED4, 0x9C9F3E4A, 0x121039A9, 0xDEBA3937,
        0xD47F302E, 0x18D530B0, 0x965A3753, 0x5AF037CD,
        0xFF6B144A, 0x33C114D4, 0xBD4E1337, 0x71E413A9,
        0x7B211AB0, 0xB78B1A2E, 0x39041DCD, 0xF5AE1D53,
        0x2C8E0FFF, 0xE0240F61, 0x6EAB0882, 0xA201081C,
        0xA8C40105, 0x646E019B, 0xEAE10678, 0x264B06E6
    }
};
```

crc_macros.h

```cpp
#ifdef WORDS_BIGENDIAN
#   define A(x) ((x) >> 24)
#   define B(x) (((x) >> 16) & 0xFF)
#   define C(x) (((x) >> 8) & 0xFF)
#   define D(x) ((x) & 0xFF)

#   define S8(x) ((x) << 8)
#   define S32(x) ((x) << 32)

#else
#   define A(x) ((x) >> 24)
#   define B(x) (((x) >> 16) & 0xFF)
#   define C(x) (((x) >> 8) & 0xFF)
#   define D(x) ((x) & 0xFF)

#   define S8(x) ((x) << 8)
#   define S32(x) ((x) << 32)
#endif
```

stream\_flags\_common.c

```cpp
const uint8_t lzma_header_magic[6] = { 品牌字符串 };
```

重新编译xz

```bash
cd xz-5.6.2
./configure
make
```

下载squashfs-tools源码，对源码中的./suqashfs文件夹下的squashfs\_fs.h文件SQUASHFS\_MAGIC值做如下修改

```cs
#define SQUASHFS_MAGIC          0x6563696e
#define SQUASHFS_MAGIC_SWAP     0x6e696365
```

重新./make编译squashfs-tools源码，得到unsquashfs程序，

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccGGgxlymP9DvzQkOHZhYoCDSF2IV0DcHIhq5iao82hblic0CHBcaU8gTw/640?wx_fmt=png)

并执行以下命令

```javascript
export LD_PRELOAD=/usr/local/lib/liblzma.so
./unsquashfs xxx_fs.bin 
```

可以成功解出文件系统

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccATpZPaVvn9eIZzENDjRE0T8tXrR4Y8zmlNYZAxkic32r6pLbplK5aPw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccGaWd1tibiadfIJo6liaKNpyOHpticJOMxUPulksGbV3nCOawHTk5O4WqsA/640?wx_fmt=png)

**总 结**

参考资料

某T路由器固件的解压逻辑即将解压逻辑写入linux内核中，在内核系统启动过程中，会将要更新的文件系统解压出来，解压过程有自修改后的文件系统头和xxx\_crc32\_le等函数校验过程，所以想要直接使用binwalk之类的工具提取固件中的文件系统会失败，要想解压出文件系统需要绕过crc校验函数或者对xz工具中的crc32\_fast.c中lzma\_crc32函数进行修改。

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT0mSRTxbY7fsoLUFViaxk1nhQByibgTdbwbMqNibWMKbHKrjwUUY8GNZlAoUlcic5ibVhyCebVwoNialnow/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**作者名片**

![](https://mmbiz.qpic.cn/mmbiz_jpg/3k9IT3oQhT0G5lZm70CtDtrE4B8e4iciccMk5by5TqhDWodNxsd54xibTFcc0wv2Ju4cOtkGo1eTibBluY0k1vTxNA/640?wx_fmt=jpeg)

END

![](https://mmbiz.qpic.cn/mmbiz_gif/3k9IT3oQhT0Z79Hq9GCticVica4ufkjk5xiarRicG97E3oEcibNSrgdGSsdicWibkc8ycazhQiaA81j3o0cvzR5x4kRIcQ/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

**往 期 热 门**

(点击图片跳转）

[![](https://mmbiz.qpic.cn/mmbiz_jpg/3k9IT3oQhT2gek1jibSk6ETagsxlDkrcsWhuZaZauwm0HOYqnmAj2KR1WibeFIrkicy5ZOMVGh4OLl7kVeDeoX25g/640?wx_fmt=jpeg)
](http://mp.weixin.qq.com/s?__biz=MzAxNDY2MTQ2OQ==&mid=2650967368&idx=1&sn=9358673d69d582b89ea96289e9db86c4&chksm=8079cf7ab70e466c71eae54cad19f26cf867e29dfa5fee6a3b149cc30f7396a4fc9198af67fc&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT175PjF9WpjjBBaU9mFhN27kn9XxoozicQELzh9m6CRniaMbBP3OmAYrg08QlGrUvXZ1DrAndMAvKPA/640?wx_fmt=png)
](http://mp.weixin.qq.com/s?__biz=MzAxNDY2MTQ2OQ==&mid=2650966964&idx=1&sn=569fbf5303fb161266bc75c2fbab62b9&chksm=8079c986b70e4090513aaa586cf640ecf711fc6a1936b340e57209b67543d78baeee19722864&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT2vmaJPqjHJNPzPxAR8kDoJ4iaYh6o6Yiawg8tE34A5VDGSviaEX2I7zibpKA30nSYafianDicpibqvU7Ukw/640?wx_fmt=png)
](http://mp.weixin.qq.com/s?__biz=MzAxNDY2MTQ2OQ==&mid=2650966953&idx=1&sn=c3e7698526d467c6ce2986b33e87b390&chksm=8079c99bb70e408d373cc59c4ff7c127ae539fcfff4f89f9d751e5d854d8fbeb10a048bd7fb8&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/3k9IT3oQhT2lPCugsWDQaQ4y4TicQ2PYkP1ic0pfWibibFsiavzULenib1K6qzR4URa5P0nAI4AQ8tLKZVmtibYvjWpIg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_gif/3k9IT3oQhT3XlD8Odz1EaR5icjZWy3jb8ZZPdfjQiakDHOiclbpjhvaR2icn265LYMpu3CmR1GoX707tWhAVsMJrrQ/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

戳“阅读原文”更多精彩内容!