# 【Web逆向】阿里云盘签名算法探究及可用 PoC
**作者论坛账号：t00t00**

**前言**

先放文件

https://github.com/kazutoiris/ali_ecc  
通过 Python 实现的模拟阿里云盘签名算法。可以通过云盘\`x-device-id\` 和 \`x-signature\` 校验，妈妈再也不用担心  \`invalid X-Device-Id\` 了。  
欢迎 Star、Issue、Pull Requests！！！

（施工完毕，请放心阅读）

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/823f1ea4-3fe6-44ff-985e-3f2b0bc4eac1.gif?raw=true)

**背景**

自从阿里云盘2023年2月13日一波大更新，直接把原来调网页版接口的第三方网盘客户端给干爆了。像小白羊、Clouddrive、AList都出现了\`invalid X-Device-Id\`。其中，Clouddrive通过官方申请API招安的方式勉强能用，其他俩都不能下载文件了。

这使得复习高数的我直接傻眼了，第三方客户端只能上传不能下载，而网页版的在线视频播放功能一坨，只好全部缓存下来再看。。。

这波操作属实有点逆天，晚饭吃撑了研究一下。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/fcce2490-ded5-4271-9c1e-72dcccfda680.png?raw=true)

**分析**

STEP1

通过报错可以发现，阿里云盘请求时候主要使用 \`x-device-id\` 和 \`x-signature\` 来进行校验。像列目录之类的可以不用校验，但是取下载链接一定要校验。  
（P.S. 这里发现一个小小的漏洞，在列文件目录的时候会携带下载链接，这似乎是油猴脚本至今可用的原因）

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/b813e42c-2bfa-4009-a324-094f113737d7.png?raw=true)

如果不带的话就只能收到 \`invalid X-Device-Id\` 的错误提示了。

STEP2

貌似在不同浏览器上登录，这个 \`x-device-id\` 会不同。暂时不清楚是怎样随机生成的（准确来说是懒得调了，又不是不能用）。

(( 补充一下：84楼大佬已经分析了一下 x-device-id 的生成过程，只是普通的 UUID 随机字符串。))  
这样， x-device-id 就可以指定随机数种子 userId，随机且唯一地生成一个 UUID 字符串，避免风控。  
privateKey 则可以对 userId 做 SHA256，所得值用于私钥生成，同理可以保证随机且唯一。  
  
综上所述，唯一要生成的就是 \`x-signature\`。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/35a487f2-c6d1-459c-a892-d3ad80a14c75.png?raw=true)

**开导逆**

STEP1

首先就是要确定 \`x-signature\` 啥时候生成的。因为在前几条请求中都没有涉及到 \`x-signature\`，只有在请求目录后，所有的请求都带上了 \`x-signature\`。  
（这里发现一个小小的细节：用开发者工具的”编辑并重新发送“，可以发现，即使是删掉了 \`x-signature\`，发送时也会自动带上。）

STEP2

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/466a8ab0-6163-4d12-a4cb-f30918c07565.png?raw=true)

随机挑一个请求，查看发起的程序。这个一目了然，发送逻辑必定在 \`bundle.js\` 中！

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/4885600b-6a1c-4c9c-a528-c57ac45eb758.png?raw=true)

一个搜索，一个回车，直达要害！这证明了之前的猜想没错。

STEP3

看了一下请求记录，没有 WASM，那估计不是 WASM，是纯 JS 加密算法。

悲，纯 JS 没得下硬件访问断点，WASM 调试起来可方便多了。。

不管了，直接一个断，一个刷新，欸，直接就断下了。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/b6b9c519-e28d-4c77-b9ea-2a1c6c323779.png?raw=true)

这不比调 WASM 爽，环境不用补，连反调试都没有。

仔细观察，createSession 中 \`x-signature\` 是通过外部调用传参进来的，并不是这个函数内部产生的。  
所以，这就需要根据调用堆栈来找是谁生成了这个值，是哪个函数传递进来的。

但是看了一圈，似乎在调用前就有了签名值。这就意味着必须要换一种方法了。

STEP4

直接开搜 \`createSession\`，很快啊，就找到了引用位置。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/b898b9cd-c4b2-4da4-99b4-03753ee5c558.png?raw=true)

看了一下，整体代码似乎是状态机结构，总的来说还是通俗易懂的，比起恶心的 ollvm 确实正常了不少。

整体代码是自上而下运行的，是一种流水线式的运行方式。打个比方，A切肉，B把切完的肉拿去腌，C把腌完的肉拿去烤，D把烤完的拿去出餐。

这个函数先生成签名，然后生成会话。生成会话过程的签名就是上一级产生的。  
当然，这里有很多其他函数，用以处理不同流水线段之间的数据传递，感兴趣的可以研究下，不感兴趣的倒也没啥关系，毕竟不影响抽象实现。  
所以这个函数两个部分既可以分成多段运行，也可以直接合并起来。

而这种流水线的方式会导致前面找堆栈的时候，没有办法找到原始签名字符串是啥时候传进来的，就像是写在了全局变量一样（这也是没办法下硬件写入断点的槽点，不然早找到了）。

STEP5

找准了函数，下一步就简单了。创建签名是在 \`genSignature\` 中，直接一个 Ctrl+F 直达。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/41f6c767-4e3b-479a-91f5-d5c0b7b1a0ca.png?raw=true)

整体就很清晰了：

*   先生成字符串 appId:deviceId:userId:nonce，然后 sha256 编码一下。
    
*   交付给下一段流水线，用 privateKey 去签名。
    
*   签名值后面加上 01。（这里的 1 是代码中 concat(u)，因为 u 来自 recovered 的值，也就是图上蓝色的下一行。这个 1 的来源后续慢慢讲）  
    

STEP6

签名是找到了，但是是拿啥算法签名的呢？

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/67026c73-16fc-4ff8-ac90-3e6b3eae214d.png?raw=true)

可以看到，阿里云盘有请求一个 \`get\_public\_key\` 的地址，而且返回了一个 RSA 密钥。前面代码又有公私钥加密的，这不就是 RSA 签名嘛！

哎哎哎，可别高兴太早，RSA 签名有个显著的特点，就是 e 基本都是 65537。但是调了这么久，似乎从来也没看到过 65537？  
而且，根据上面的代码，这个私钥可以直接转换成公钥？RSA 双向转换都是不可以的。就算是调用私钥，最起码也得带上俩个数吧。

```css
this.secp.utils.bytesToHex(this.secp.getPublicKey(this.dbMemData.privateKey))
```

  
这波属实是半场开香槟——给爷整笑了。

STEP7

兄弟们，干就完事了。

现在有两种可能，一是魔改的 RSA 算法，存在一个常数，这样就只需要提供另一个数就行了；二是，这压根不是 RSA 算法。

不过这两种路子殊途同归，直接一个断点，打到 \`getPublicKey\` 里面。如果是魔改，必定有常数，如果不是，那也有提示。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/f4218ca5-c12d-44b7-a66b-10a2810f11b9.png?raw=true)
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/6fd323a0-3e46-4c42-aace-32bbd6840e07.png?raw=true)

可以看到这是一个乘法运算。RSA 中我怎么记得是没有乘法的呢？

放狗搜一下常数，哦哦哦哦哦哦.jpg

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/5d5bd9e6-979f-4f82-a7a6-0516b66a213b.png?raw=true)

来了兄弟们，ecc-secp256k1 算法，也是非对称算法。这是用椭圆曲线做的，难怪怎么只需要一个常数就可以了。

一把梭，直接开搞。

```makefile
private_key = [175, 87, 171, 214, 222, 196, 127, 36, 25, 50, 237, 179, 71, 81, 49, 196,
               250, 103, 115, 203, 138, 179, 192, 182, 43, 175, 233, 72, 200, 14, 64, 254]
private_key = int.from_bytes(private_key, byteorder='big')
ecc_pri = ecdsa.SigningKey.from_secret_exponent(
    private_key, curve=ecdsa.SECP256k1)
ecc_pub = ecc_pri.get_verifying_key()
print(ecc_pub.to_string().hex())
```

  
STEP8

别高兴太早。

出来的是 3e2a3bbd2dfe8675bc40b0b0af6a558421b827b5ad022c8d305eb861b9bfdcc48aea1243df77a6298dfd5cf692a407852b3fd996ea5a13ae213c4380bb7b8d0d，  
而发送的却是 043e2a3bbd2dfe8675bc40b0b0af6a558421b827b5ad022c8d305eb861b9bfdcc48aea1243df77a6298dfd5cf692a407852b3fd996ea5a13ae213c4380bb7b8d0d

奇怪，算法搞错了？

别急，这不还没调完嘛。。。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/e082212e-401b-4c91-afae-c62e2e910fcf.png?raw=true)

```javascript
t ? `${this.y & o ? "03" : "02"}${e}` : `04${e}${A(this.y)}`
```

  
是吧，04 是后续转 HEX 的时候手动加上的。这证明算法已经掌握了。

(( 经坛友提醒，04 指的是未压缩 ECC 公钥，细节可以看这篇文章 https://asecuritysite.com/ecc/js_ethereum2 ))  
  
STEP9

算法搞完了，回过头看看公私钥对是咋生成的，这玩意不会有啥稀奇古怪的限制吧。

现在有两点已经掌握：

*   只需要提供私钥，就可以求解得到公钥。
    
*   私钥每次登陆时都会变。  
    

大致可以分析出来，只需要生成私钥就可以了，而且是在初始化的时候。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/ad0ff20a-a02a-465c-99d2-bd1909d361b4.png?raw=true)

没有限制，随机生成，完美收官。

STEP10

搞完了生成、算法，但是签名呢？哎，没想到吧，前面已经提到了哦！

*   先生成字符串 appId:deviceId:userId:nonce，然后 sha256 编码一下。
    
*   交付给下一段流水线，用 privateKey 去签名。
    
*   签名值后面加上 01。（这里的 1 是代码中 concat(u)，因为 u 来自 recovered 的值，也就是图上蓝色的下一行。这个 1 的来源后续慢慢讲）  
    

那你肯定要问了，楼主楼主，nonce 一开始是 0，后面会不会变捏？  
  
简单，友谊手套，Ctrl + F 一搜 nonce 赋值，很快奥，就定位到了。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/1539e255-0cc0-4285-a72b-e60ef468dc0d.png?raw=true)

目测是超时后生成新的 nonce。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/74606fd6-79ff-411f-907d-c0ddda911d76.png?raw=true)

更新时间随机的。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/6cdacf8d-bebf-4f64-b974-5bdef3dfbb11.png?raw=true)

这里 nonce 只起限位作用，每次更新 +1，初始值为零，更新时间随机。

那你肯定又要问了，楼主楼主，appId、deviceId、userId 捏？  
自己去 GitHub 项目看一下捏，这个抓包都有的，这里就不多做解释了，写文章太累了。。。

**开积写**

把前面导完的东西积一下，首先生成密钥对，发送公钥至阿里云盘，后续则使用这个密钥对进行签名。

STEP1 生成密钥对

```makefile
private_key = random.randint(1, 2**256-1)
ecc_pri = ecdsa.SigningKey.from_secret_exponent(
    private_key, curve=ecdsa.SECP256k1)
```

  
STEP2 生成并处理公钥

```ini
ecc_pub = ecc_pri.get_verifying_key()
public_key = "04"+ecc_pub.to_string().hex()
```

STEP3 签名

```python
def sign(appId, deviceId, userId, nonce) -> str:
    sign_dat = ecc_pri.sign(r(appId, deviceId, userId, nonce).encode('utf-8'), entropy=None,
                            hashfunc=hashlib.sha256)
    return sign_dat.hex()+"01"
```

  
至于发送这种小事，完整代码直接见仓库吧，贴完文章太长没眼看了。

这里有个坑点，非对称签名每次签同样的内容产生的签名是不一样的，但是似乎阿里云盘直接简单处理了。  
create_session 时候用的 \`x-signature\` 必须原封不动的传递到其他 API 的调用上，否则就算是密钥对一致、内容一致，也只会报无效。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/a5f48791-0da6-47c3-bc19-4ca71480881a.png?raw=true)

**后记**

*   似乎油猴脚本下载不受影响。
    
*   AList通过分享创建的也不受影响。
    
*   盲猜一波断更小白羊估计这次应该是彻底淘汰了，作者不管，第三方作者也不太好做，不知道有没有接盘侠接个盘。
    
*   有啥问题建议直接 GitHub 项目下提交 Issue，秒级回复。论坛估计是得按天起步了。。。
    
*   Star、Follow、CB 任一大于 1145，胱速整个有意思小活。
    

****-官方论坛****

www.52pojie.cn

**--推荐给朋友**

公众微信号：吾爱破解论坛

或搜微信号：pojie_52