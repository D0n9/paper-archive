# API安全接口安全设计
如何保证外网开放接口的安全性。

*   使用加签名方式，防止数据篡改
    
*   信息加密与密钥管理
    
*   搭建OAuth2.0认证授权
    
*   使用令牌方式
    
*   搭建网关实现黑名单和白名单
    

### 一、令牌方式搭建搭建API开放平台

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/4465861f-b0ab-444f-8d25-b2d498f8acde.png?raw=true)

**方案设计：** 

1.第三方机构申请一个appId,通过appId去获取accessToken,每次请求获取accessToken都要把老的accessToken删掉

2.第三方机构请求数据需要加上accessToken参数，每次业务处理中心执行业务前，先去dba持久层查看accessToken是否存在(可以把accessToken放到redis中，这样有个过期时间的效果)，存在就说明这个机构是合法，无需要登录就可以请求业务数据。不存在说明这个机构是非法的，不返回业务数据。

3.好处：无状态设计，每次请求保证都是在我们持久层保存的机构的请求，如果有人盗用我们accessToken，可以重新申请一个新的taken.

### 二、基于OAuth2.0协议方式

**原理**

第三方授权，原理和1的令牌方式一样

1.假设我是服务提供者A，我有开发接口，外部机构B请求A的接口必须申请自己的appid(B机构id)

2.当B要调用A接口查某个用户信息的时候，需要对应用户授权，告诉A，我愿同意把我的信息告诉B，A生产一个授权token给B。

3.B使用token获取某个用户的信息。

**联合微信登录总体处理流程**

1.  用户同意授权，获取code
    
2.  通过code换取网页授权access_token
    
3.  通过access_token获取用户openId
    
4.  通过openId获取用户信息
    

### 三、信息加密与密钥管理

*   单向散列加密
    
*   对称加密
    
*   非对称加密
    
*   安全密钥管理
    

#### 1.单向散列加密

散列是信息的提炼，通常其长度要比信息小得多，且为一个固定长度。加密性强的散列一定是不可逆的，这就意味着通过散列结果，无法推出任何部分的原始信息。任何输入信息的变化，哪怕仅一位，都将导致散列结果的明显变化，这称之为雪崩效应。

散列还应该是防冲突的，即找不出具有相同散列结果的两条信息。具有这些特性的散列结果就可以用于验证信息是否被修改。

单向散列函数一般用于产生消息摘要，密钥加密等，常见的有：

*   MD5（Message Digest Algorithm 5）：是RSA数据安全公司开发的一种单向散列算法，非可逆，相同的明文产生相同的密文。
    
*   SHA（Secure Hash Algorithm）：可以对任意长度的数据运算生成一个160位的数值；
    

**SHA-1与MD5的比较**

因为二者均由MD4导出，SHA-1和MD5彼此很相似。相应的，他们的强度和其他特性也是相似，但还有以下几点不同：

*   对强行供给的安全性：最显著和最重要的区别是SHA-1摘要比MD5摘要长32 位。使用强行技术，产生任何一个报文使其摘要等于给定报摘要的难度对MD5是2128数量级的操作，而对SHA-1则是2160数量级的操作。这样，SHA-1对强行攻击有更大的强度。
    
*   对密码分析的安全性：由于MD5的设计，易受密码分析的攻击，SHA-1显得不易受这样的攻击。
    
*   速度：在相同的硬件上，SHA-1的运行速度比MD5慢。
    

1、特征：雪崩效应、定长输出和不可逆。

2、作用是：确保数据的完整性。

3、加密算法：md5（标准密钥长度128位）、sha1（标准密钥长度160位）、md4、CRC-32

4、加密工具：md5sum、sha1sum、openssl dgst。

5、计算某个文件的hash值，例如：md5sum/shalsum FileName,openssl dgst –md5/-sha

#### 2.对称加密

秘钥：加密解密使用同一个密钥、数据的机密性双向保证、加密效率高、适合加密于大数据大文件、加密强度不高(相对于非对称加密)

**对称加密优缺点**

*   优点：与公钥加密相比运算速度快。
    
*   缺点：不能作为身份验证，密钥发放困难
    
    ![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/453f768e-5117-4d3c-be75-18647062e7b2.png?raw=true)
    

**DES是一种对称加密算法,加密和解密过程中，密钥长度都必须是8的倍数**

```
   
public class DES {  
 public DES() {  
 }

  // 测试  
 public static void main(String args[]) throws Exception {  
  // 待加密内容  
  String str = "123456";  
  // 密码，长度要是8的倍数 密钥随意定  
  String password = "12345678";  
  byte[] encrypt = encrypt(str.getBytes(), password);  
  System.out.println("加密前:" +str);  
  System.out.println("加密后:" + new String(encrypt));  
  // 解密  
  byte[] decrypt = decrypt(encrypt, password);  
  System.out.println("解密后:" + new String(decrypt));  
 }

  /**  
  * 加密  
  *   
  * @param datasource  
  *            byte[]  
  * @param password  
  *            String  
  * @return byte[]  
  */  
 public static byte[] encrypt(byte[] datasource, String password) {  
  try {  
   SecureRandom random = new SecureRandom();  
   DESKeySpec desKey = new DESKeySpec(password.getBytes());  
   // 创建一个密匙工厂，然后用它把DESKeySpec转换成  
   SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("DES");  
   SecretKey securekey = keyFactory.generateSecret(desKey);  
   // Cipher对象实际完成加密操作  
   Cipher cipher = Cipher.getInstance("DES");  
   // 用密匙初始化Cipher对象,ENCRYPT_MODE用于将 Cipher 初始化为加密模式的常量  
   cipher.init(Cipher.ENCRYPT_MODE, securekey, random);  
   // 现在，获取数据并加密  
   // 正式执行加密操作  
   return cipher.doFinal(datasource); // 按单部分操作加密或解密数据，或者结束一个多部分操作  
  } catch (Throwable e) {  
   e.printStackTrace();  
  }  
  return null;  
 }

  /**  
  * 解密  
  *   
  * @param src  
  *            byte[]  
  * @param password  
  *            String  
  * @return byte[]  
  * @throws Exception  
  */  
 public static byte[] decrypt(byte[] src, String password) throws Exception {  
  // DES算法要求有一个可信任的随机数源  
  SecureRandom random = new SecureRandom();  
  // 创建一个DESKeySpec对象  
  DESKeySpec desKey = new DESKeySpec(password.getBytes());  
  // 创建一个密匙工厂  
  SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("DES");// 返回实现指定转换的  
                   // Cipher  
                   // 对象  
  // 将DESKeySpec对象转换成SecretKey对象  
  SecretKey securekey = keyFactory.generateSecret(desKey);  
  // Cipher对象实际完成解密操作  
  Cipher cipher = Cipher.getInstance("DES");  
  // 用密匙初始化Cipher对象  
  cipher.init(Cipher.DECRYPT_MODE, securekey, random);  
  // 真正开始解密操作  
  return cipher.doFinal(src);  
 }  
}

 输出

 加密前:123456  
加密后:>p.72|  
解密后:123456


```

#### 3.非对称加密

非对称加密算法需要两个密钥：公开密钥（publickey:简称公钥）和私有密钥（privatekey:简称私钥）。

**公钥与私钥是一对**

*   公钥对数据进行加密，只有用对应的私钥才能解密
    
*   私钥对数据进行加密，只有用对应的公钥才能解密
    

**过程：** 

*   甲方生成一对密钥，并将公钥公开，乙方使用该甲方的公钥对机密信息进行加密后再发送给甲方；
    
*   甲方用自己私钥对加密后的信息进行解密。
    
*   甲方想要回复乙方时，使用乙方的公钥对数据进行加密
    
*   乙方使用自己的私钥来进行解密。
    
*   甲方只能用其私钥解密由其公钥加密后的任何信息。
    

**特点：** 

*   算法强度复杂
    
*   保密性比较好
    
*   加密解密速度没有对称加密解密的速度快。
    
*   对称密码体制中只有一种密钥，并且是非公开的，如果要解密就得让对方知道密钥。所以保证其安全性就是保证密钥的安全，而非对称密钥体制有两种密钥，其中一个是公开的，这样就可以不需要像对称密码那样传输对方的密钥了。这样安全性就大了很多
    
*   适用于：金融，支付领域
    

**RSA加密是一种非对称加密**

```
import javax.crypto.Cipher;  
import java.security.*;  
import java.security.interfaces.RSAPrivateKey;  
import java.security.interfaces.RSAPublicKey;  
import java.security.spec.PKCS8EncodedKeySpec;  
import java.security.spec.X509EncodedKeySpec;  
import java.security.KeyFactory;  
import java.security.KeyPair;  
import java.security.KeyPairGenerator;  
import java.security.PrivateKey;  
import java.security.PublicKey;  
import org.apache.commons.codec.binary.Base64;

  /**  
 * RSA加解密工具类  
 *  
 *   
 */  
public class RSAUtil {

  public static String publicKey; // 公钥  
 public static String privateKey; // 私钥

  /**  
  * 生成公钥和私钥  
  */  
 public static void generateKey() {  
  // 1.初始化秘钥  
  KeyPairGenerator keyPairGenerator;  
  try {  
   keyPairGenerator = KeyPairGenerator.getInstance("RSA");  
   SecureRandom sr = new SecureRandom(); // 随机数生成器  
   keyPairGenerator.initialize(512, sr); // 设置512位长的秘钥  
   KeyPair keyPair = keyPairGenerator.generateKeyPair(); // 开始创建  
   RSAPublicKey rsaPublicKey = (RSAPublicKey) keyPair.getPublic();  
   RSAPrivateKey rsaPrivateKey = (RSAPrivateKey) keyPair.getPrivate();  
   // 进行转码  
   publicKey = Base64.encodeBase64String(rsaPublicKey.getEncoded());  
   // 进行转码  
   privateKey = Base64.encodeBase64String(rsaPrivateKey.getEncoded());  
  } catch (NoSuchAlgorithmException e) {  
   // TODO Auto-generated catch block  
   e.printStackTrace();  
  }  
 }

  /**  
  * 私钥匙加密或解密  
  *   
  * @param content  
  * @param privateKeyStr  
  * @return  
  */  
 public static String encryptByprivateKey(String content, String privateKeyStr, int opmode) {  
  // 私钥要用PKCS8进行处理  
  PKCS8EncodedKeySpec pkcs8EncodedKeySpec = new PKCS8EncodedKeySpec(Base64.decodeBase64(privateKeyStr));  
  KeyFactory keyFactory;  
  PrivateKey privateKey;  
  Cipher cipher;  
  byte[] result;  
  String text = null;  
  try {  
   keyFactory = KeyFactory.getInstance("RSA");  
   // 还原Key对象  
   privateKey = keyFactory.generatePrivate(pkcs8EncodedKeySpec);  
   cipher = Cipher.getInstance("RSA");  
   cipher.init(opmode, privateKey);  
   if (opmode == Cipher.ENCRYPT_MODE) { // 加密  
    result = cipher.doFinal(content.getBytes());  
    text = Base64.encodeBase64String(result);  
   } else if (opmode == Cipher.DECRYPT_MODE) { // 解密  
    result = cipher.doFinal(Base64.decodeBase64(content));  
    text = new String(result, "UTF-8");  
   }

   } catch (Exception e) {  
   // TODO Auto-generated catch block  
   e.printStackTrace();  
  }  
  return text;  
 }

  /**  
  * 公钥匙加密或解密  
  *   
  * @param content  
  * @param privateKeyStr  
  * @return  
  */  
 public static String encryptByPublicKey(String content, String publicKeyStr, int opmode) {  
  // 公钥要用X509进行处理  
  X509EncodedKeySpec x509EncodedKeySpec = new X509EncodedKeySpec(Base64.decodeBase64(publicKeyStr));  
  KeyFactory keyFactory;  
  PublicKey publicKey;  
  Cipher cipher;  
  byte[] result;  
  String text = null;  
  try {  
   keyFactory = KeyFactory.getInstance("RSA");  
   // 还原Key对象  
   publicKey = keyFactory.generatePublic(x509EncodedKeySpec);  
   cipher = Cipher.getInstance("RSA");  
   cipher.init(opmode, publicKey);  
   if (opmode == Cipher.ENCRYPT_MODE) { // 加密  
    result = cipher.doFinal(content.getBytes());  
    text = Base64.encodeBase64String(result);  
   } else if (opmode == Cipher.DECRYPT_MODE) { // 解密  
    result = cipher.doFinal(Base64.decodeBase64(content));  
    text = new String(result, "UTF-8");  
   }  
  } catch (Exception e) {  
   // TODO Auto-generated catch block  
   e.printStackTrace();  
  }  
  return text;  
 }

  // 测试方法  
 public static void main(String[] args) {  
  /**  
   * 注意： 私钥加密必须公钥解密 公钥加密必须私钥解密  
   *  // 正常在开发中的时候,后端开发人员生成好密钥对，服务器端保存私钥 客户端保存公钥  
   */  
  System.out.println("-------------生成两对秘钥，分别发送方和接收方保管-------------");  
  RSAUtil.generateKey();  
  System.out.println("公钥:" + RSAUtil.publicKey);  
  System.out.println("私钥:" + RSAUtil.privateKey);

   System.out.println("-------------私钥加密公钥解密-------------");  
   String textsr = "11111111";  
   // 私钥加密  
   String cipherText = RSAUtil.encryptByprivateKey(textsr,  
   RSAUtil.privateKey, Cipher.ENCRYPT_MODE);  
   System.out.println("私钥加密后：" + cipherText);  
   // 公钥解密  
   String text = RSAUtil.encryptByPublicKey(cipherText,  
   RSAUtil.publicKey, Cipher.DECRYPT_MODE);  
   System.out.println("公钥解密后：" + text);

   System.out.println("-------------公钥加密私钥解密-------------");  
  // 公钥加密  
  String textsr2 = "222222";

   String cipherText2 = RSAUtil.encryptByPublicKey(textsr2, RSAUtil.publicKey, Cipher.ENCRYPT_MODE);  
  System.out.println("公钥加密后：" + cipherText2);  
  // 私钥解密  
  String text2 = RSAUtil.encryptByprivateKey(cipherText2, RSAUtil.privateKey, Cipher.DECRYPT_MODE);  
  System.out.print("私钥解密后：" + text2 );  
 }

 }


```

### 四、使用加签名方式，防止数据篡改

客户端：请求的数据分为2部分（业务参数，签名参数），签名参数=md5（业务参数）

服务端：验证md5(业务参数)是否与签名参数相同

```javascript
作者:单身贵族男
原文:https:
```

**侵权请私聊公众号删文**

 **热文推荐**

*   [蓝队应急响应姿势之Linux](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247523380&idx=1&sn=27acf248b4bbce96e2e40e193b32f0c9&chksm=f9e3f36fce947a79b416e30442009c3de226d98422bd0fb8cbcc54a66c303ab99b4d3f9bbb05&scene=21#wechat_redirect)  
    
*   [通过DNSLOG回显验证漏洞](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247523485&idx=1&sn=2825827e55c1c9264041744a00688caf&chksm=f9e3f3c6ce947ad0c129566e5952ac23c990cf0428704df1a51526d8db6adbc47f998ee96eb4&scene=21#wechat_redirect)  
    
*   [记一次服务器被种挖矿溯源](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247523441&idx=2&sn=94c6fae1f131c991d82263cb6a8c820b&chksm=f9e3f32ace947a3cdae52cf4cdfc9169ecf2b801f6b0fc2312801d73846d28b36d4ba47cb671&scene=21#wechat_redirect)  
    
*   [内网渗透初探 | 小白简单学习内网渗透](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247523346&idx=1&sn=4bf01626aa7457c9f9255dc088a738b4&chksm=f9e3f349ce947a5f934329a78177b9ce85e625a36039008eead2fe35cbad5e96a991569d0b80&scene=21#wechat_redirect)  
    
*   [实战|通过恶意 pdf 执行 xss 漏洞](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247523274&idx=1&sn=89290e2b7a8e408ff62a657ef71c8594&chksm=f9e3f491ce947d8702eda190e8d4f7ea2e3721549c27a2f768c3256de170f1fd0c99e817e0fb&scene=21#wechat_redirect)  
    
*   [免杀技术有一套（免杀方法大集结）(Anti-AntiVirus)](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247523189&idx=1&sn=44ea2c9a59a07847e1efb1da01583883&chksm=f9e3f42ece947d3890eb74e4d5fc60364710b83bd4669344a74c630ac78f689b1248a2208082&scene=21#wechat_redirect)  
    
*   [内网渗透之内网信息查看常用命令](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247522979&idx=1&sn=894ac98a85ae7e23312b0188b8784278&chksm=f9e3f5f8ce947cee823a62ae4db34270510cc64772ed8314febf177a7660de08c36bedab6267&scene=21#wechat_redirect)  
    
*   [关于漏洞的基础知识](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247523083&idx=2&sn=0b162aba30063a4073bad24269a8dc0e&chksm=f9e3f450ce947d4699dfebf0a60a2dade481d8baf5f782350c2125ad6a320f91a2854d027e85&scene=21#wechat_redirect)  
    
*   [任意账号密码重置的6种方法](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247522927&idx=1&sn=075ccdb91ae67b7ad2a771aa1d6b43f3&chksm=f9e3f534ce947c220664a938bc42926bee3ca8d07c6e3129795d7c8977948f060b08c0f89739&scene=21#wechat_redirect)  
    
*   [干货 | 横向移动与域控权限维持方法总汇](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247522810&idx=2&sn=ed65a8c60c45f9af598178ed20c89896&chksm=f9e3f6a1ce947fb710ff77d8fbd721220b16673953b30eba6b10ad6e86924f6b4b9b2a983e74&scene=21#wechat_redirect)  
    
*   [手把手教你Linux提权](http://mp.weixin.qq.com/s?__biz=MzUyMTA0MjQ4NA==&mid=2247522500&idx=2&sn=ec74a21ef0a872f7486ccac6772e0b9a&chksm=f9e3f79fce947e89eac9d9077eee8ce74f3ab35a345b1c2194d11b77d5b522be3b269b326ebf&scene=21#wechat_redirect)
    

****欢迎关注LemonSec****

**觉得不错点个**“赞”**、“在看”哦**
