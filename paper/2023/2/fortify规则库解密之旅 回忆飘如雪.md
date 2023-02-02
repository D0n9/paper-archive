# fortify规则库解密之旅 | 回忆飘如雪
前段时间在学习fortify的规则编写，想充分利用其污点回溯功能来扫描出当下比较新的漏洞，比如fastjson反序列化漏洞。网上有比较好的资料是《fortify安全代码规则编写指南》，但是很缺例子。于是想参考下官方的规则库，但是是加密的，万般无奈只能踏上解密之旅。

[](#0x01-解密思路 "0x01 解密思路")0x01 解密思路
-----------------------------------

猜测fortify会和AWVS一样，会将规则库加载到内存当中进行解密，然后再使用其进行代码扫描。基于这个想法，它必然存在一个解密方法，而这个方法肯定在某个jar当中。锁定负责解密的jar之后，就可以审计jar的所有方法。然后通过调试来理清解密流程，最后我们就可以写代码来模拟这个过程，来解密规则库。

[](#0x02-定位解密jar "0x02 定位解密jar")0x02 定位解密jar
--------------------------------------------

通过反编译发现fortify依赖的jar基本都没有混淆，说明我们可以通过`jar名`和`类名`来初步锁定加密方法所在jar。类名搜索工具使用的是我在[《如何快速找到POC/EXP依赖的jar？》](https://gv7.me/articles/2019/quickly-find-jars-that-depend-on-poc-exp/)一文中开发的`SearchClassInJar.jar`。在分别尝试`encrypt`,`decrypt`,`crypto`,`rule`,`fortify`等关键字后,最终搜索到两个可疑jar。

1.  fortify-common-17.10.0.0156.jar
2.  fortify-crypto-1.0.jar

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/d34756e9-250d-445a-b1e5-e193a9ec51a2.png?raw=true)
](https://gv7.me/articles/2019/fortify-rule-library-decryption-process/F42189E8-11C8-4825-A49B-58FD79640C35.png)搜索解密jar

[](#0x03-定位解密方法 "0x03 定位解密方法")0x03 定位解密方法
-----------------------------------------

#### [](#3-1-通过调试定位 "3.1 通过调试定位")3.1 通过调试定位

定位解密方法最好的方法就是调试。打开fortify的`\Core\private-bin\awb\productlaunch.cmd`脚本，在最后一行如下图位置粘贴调试配置，就可以以调试模式启动fortify。然后配置IDEA连接5005端口即可进行调试。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/cde3c32e-b95f-49cd-8072-4dfab5e1952e.png?raw=true)
](https://gv7.me/articles/2019/fortify-rule-library-decryption-process/ECBDE745-99FE-40AC-8C13-D1267B9CA5BB.png)让fortify开启调试模式

通过审计这两个jar代码，基本确定`fortify-crypto-1.0.jar`就是加解密方法所在。通过函数名，参数类型，代码逻辑确定了如下涉及解密的可疑方法，并给它们都打上断点。

1.  void `decrypt`(long\[\] v, long\[\] k)
2.  void `dec`(InputStream source, OutputStream dest, long\[\] usrKey)
3.  InputStream `decryptCompressedAfterHeaders`(InputStream encrypted, String keyString)
4.  InputStream `decryptAfterHeaders`(InputStream encrypted, String keyString, boolean compressed)
5.  InputStream `decryptCompressed`(InputStream encrypted, String keyString)
6.  void `encryptAfterHeaders`(InputStream stream, OutputStream ciphertext, String keyString, boolean compress)

接着运行fortify扫描一个`java web demo`，最终漏洞是扫描出来了，但是没有一个可疑方法被调用，甚是奇怪。于是我将所有方法都打上断点，发现扫描期间只有`readHeaders(InputStream encrypted)`被调用了。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/c39b616b-97b0-47b9-9c12-d162648f1af8.png?raw=true)
](https://gv7.me/articles/2019/fortify-rule-library-decryption-process/8128736C-CBCB-4521-9E67-E33D900E0756.png)扫描期间只有readHeaders方法被调用

难道fortify并没有在扫描时对规则进行解密，可以直接读取规则内容？后面通过调用栈上下文也没发现解密操作。

#### [](#3-2-通过编码调用定位 "3.2 通过编码调用定位")3.2 通过编码调用定位

这时一个朋友突然叫去包饺子，我才记起今天是冬至。为了速战速决，我决定 通过写代码直接将规则库传入到可疑方法中进行解密，然后看返回的解密结果是否是有意义的明文来判断是否是我们要找的解密方法。 于是将CryptoUtil类中的所有代码审计一遍之后，发现decryptCompressed()可以解密压缩一个文件，感觉看到来希望。​

下面我们来看看该方法的运行流程。该方法最终会调用decryptAfterHeaders()，它负责控制解密解压整个流程。可以看到如果key没设置会被设置为默认值。接着会调用doBlockCipher()来解密，使用uncompressString来解压。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/33b1aa31-ad47-4056-8e29-ebbf8d933b10.png?raw=true)
](https://gv7.me/articles/2019/fortify-rule-library-decryption-process/0B404A33-CFBB-49CF-BE75-FB2364DEA968.png)解密压缩方法decryptAfterHeaders()

我们再来看看`doBlockCipher()`方法,它可以进行加密和解密。传入的是`false`所以是解密。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/5d4d70c6-f2fe-484e-b1be-3de621d77f6c.png?raw=true)
](https://gv7.me/articles/2019/fortify-rule-library-decryption-process/DC72BDA4-2492-405B-AC9A-3815A386311A.png)doBlockCipher()方法调用dec对文件进行解密

而最终文件内容会被传入`dec()`方法解密。

```java
private static void dec(InputStream source, OutputStream dest, long[] usrKey) throws IOException {
    long[] k = (long[])((long[])usrKey.clone());
    byte[] byteBuf = new byte[8];
    byte[] byteBufDelay = null;
    long[] unsigned32Buf = new long[2];
    long top = 4294967295L;

    int bytesRead;
    while((bytesRead = source.read(byteBuf)) != -1) {
        if (bytesRead < 8) {
            throw new IOException("invalid encrypted stream");
        }

        byteArrayToUnsigned32(byteBuf, unsigned32Buf);
        decrypt(unsigned32Buf, k);
        k[0] = k[0] + 17L & top;
        k[1] = k[1] + 17L & top;
        k[2] = k[2] + 17L & top;
        k[3] = k[3] + 17L & top;
        unsigned32ToByteArray(unsigned32Buf, byteBuf);
        if (source.available() == 0) {
            int bytesToWrite = byteBuf[7];
            if (bytesToWrite > 8 || bytesToWrite < 0 || byteBufDelay == null) {
                throw new IOException("invalid encrypted stream");
            }

            dest.write(byteBufDelay, 0, bytesToWrite);
        }

        if (byteBufDelay != null) {
            dest.write(byteBufDelay, 0, 8);
            byte[] t = byteBufDelay;
            byteBufDelay = byteBuf;
            byteBuf = t;
        } else {
            byteBufDelay = byteBuf;
            byteBuf = new byte[8];
        }
    }

}
```

至此我们确定decryptCompressed()可以解密解压一个文件，至于是否可以是规则库文件，我们可以写如下代码来测试。

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/5ca54786-751a-4cde-a188-759f7233ff5a.png?raw=true)
](https://gv7.me/articles/2019/fortify-rule-library-decryption-process/117337DD-962F-4B64-90AF-AC4C98D92F47.png)decryptCompressed()方法可以完美解密规则库文件

发现解密结果是有意义的xml文件内容，完美解密！

[](#0x04-编写解密程序 "0x04 编写解密程序")0x04 编写解密程序
-----------------------------------------

理清整个过程后，解密就很简单了。说白了就是批量调用fortify自带的`fortify-crypto-1.0.jar`中的`com.fortify.util.CryptoUtil.decryptCompressed()`方法进行解密。最后附上解密程序。

```java
import java.io.*;
import static com.fortify.util.CryptoUtil.decryptCompressed;

public class FortifyRuleDecrypter {
    private String ruleDir;
    private String saveDir;

    FortifyRuleDecrypter(String ruleDir,String saveDir){
        this.ruleDir = ruleDir;
        this.saveDir = saveDir;
    }

    public  void doDecrypt(){
        File encryptRule = new File(ruleDir);
        
        if(encryptRule.isFile()) {
            if(encryptRule.getName().endsWith(".bin")) {
                decryptRule(encryptRule, new File(saveDir + File.separator + encryptRule.getName() + ".xml"));
            }else{
                System.out.println("[-] The rule file suffix is.bin!");
                System.exit(0);
            }
        }

        
        if (encryptRule.isDirectory()) {
            File[] listFile = encryptRule.listFiles();
            for(File file:listFile){
                if(file.getName().endsWith(".bin")){
                    File saveName = new File(saveDir + File.separator + file.getName().replace(".bin","") + ".xml");
                    decryptRule(file,saveName);
                }
            }
        }

    }

    public  void decryptRule(File encFile, File decFile){
        try {
        	
            InputStream ruleStream = decryptCompressed(new FileInputStream(encFile), null);
            OutputStream outputStream = new FileOutputStream(decFile);
            byte[] b = new byte[1024];
            while ((ruleStream.read(b)) != -1) {
                outputStream.write(b);
            }
            ruleStream.close();
            outputStream.close();
            System.out.println(String.format("[+] success %s -> %s",encFile.getName(),decFile.getAbsolutePath()));
        }catch (Exception e){
            System.out.println(String.format("[-] fail %s -> %s",encFile.getName(),decFile.getAbsolutePath()));
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        if(args.length != 2){
            System.out.println("Usage: java -jar FortifyRuleDecrypter.jar [rule_dir|rule_file] <save_dir>");
            System.exit(0);
        }
        FortifyRuleDecrypter decrypter = new FortifyRuleDecrypter(args[0],args[1]);
        decrypter.doDecrypt();
    }
}
```

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/0d2f79df-0a56-43cb-9412-fd4e3f874f23.png?raw=true)
](https://gv7.me/articles/2019/fortify-rule-library-decryption-process/726FEDA7-ABD4-4EED-9431-B87C034A5F5C.png)解密效果

[](#0x05-最后的话 "0x05 最后的话")0x05 最后的话
-----------------------------------

最终为了快速解决问题，通过编码调用锁定解密方法，确实有运气的成分。​最终虽然解决了问题，但依然存在如下疑问，只能等有空再研究。先赶时间去朋友那撸猫包饺子去了！

1.  fortify在扫描时没有调用解密方法，难道是加密的规则库可以直接用于扫描？
2.  如果扫描无需解密规则库，那为何fortify又要在jar中提供解密方法？
3.  到底解密方法在哪里被调用？

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/1ca850e3-347a-4b59-9f81-519e378cb20b.jpeg?raw=true)
](https://gv7.me/articles/2019/fortify-rule-library-decryption-process/dumplings-and-cat.jpeg)冬至的夜晚