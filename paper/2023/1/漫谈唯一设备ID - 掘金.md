# 漫谈唯一设备ID - 掘金
一、前言
----

设备ID，简单来说就是一串符号（或者数字），映射现实中硬件设备。  
如果这些符号和设备是一一对应的，可称之为“唯一设备ID（Unique Device Identifier）”  

不幸的是，对于Android平台而言，没有稳定的API可以让开发者获取到这样的设备ID。  
开发者通常会遇到这样的困境：  
随着项目的演进， 越来越多的地方需要用到设备ID；  
然而随着Android版本的升级，获取设备ID却越来越难了。  
加上Android平台碎片化的问题，获取设备ID之路，可以说是步履维艰。

二、设备ID的作用
---------

关于设备ID的作用，大概可以分为下面几点：  

*   统计需求 统计需求是设备ID最常见的用途，包括DAU, MAU的统计，行为统计，广告激活的统计等。
    
*   业务需求 设备ID通常也用于业务中。  
    比如结合行为统计做用户画像，以为用户提供个性化的服务，大家感受比较明显的就是新闻类和电商类的APP了。  
    这类操作，有利有弊，仁者见仁智者见智。  
    又如，定向推送，不一定是广告推送，错误修复，内测推送等也会用到设备ID。  
    还有是一些和特定业务结合的用途，比如构造分布式ID等。
    
*   风控需求 设备ID还可用于防刷单，反作弊等。  
    当然，风控需求仅靠设备ID是无法完成的，通常需要建立一套反作弊系统。  
    关于这方面的内容，难以一言以蔽之，这里我们不多作展开。
    

三、获取设备ID的API
------------

获取设备标识的API屈指可数，而且都或多或少有一些问题。  
常规的API有以下这些：

#### IMEI

IMEI本该最理想的设备ID，具备唯一性，恢复出厂设置不会变化（真正的设备相关）。  
然而，获取IMEI需要 READ\_PHONE\_STATE 权限，估计大家也知道这个权限有多麻烦了。  
尤其是Android 6.0以后, 这类权限要动态申请，很多用户可能会选择拒绝授权。  
我们看到，有的APP不授权这个权限就无法使用， 这可能会降低用户对APP的好感度。  
而且，Android 10.0 将彻底禁止第三方应用获取设备的IMEI, 即使申请了 READ\_PHONE\_STATE 权限。  
所以，如果是新APP，不建议用IMEI作为设备标识；  
如果已经用IMEI作为标识，要赶紧做兼容工作了，尤其是做新设备标识和IMEI的映射。

#### 设备序列号

通过[android.os.Build.SERIAL](https://link.juejin.cn/?target=http%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroid%2Fos%2FBuild.html%23SERIAL "http://developer.android.com/reference/android/os/Build.html#SERIAL")获得，由厂商提供。  
如果厂商比较规范的话，设备序列号+Build.MANUFACTURER应该能唯一标识设备。 但现实是并非所有厂商都按规范来，尤其是早期的设备。  
最致命的是，Android 8.0 以上，android.os.Build.SERIAL 总返回 “unknown”；  
若要获取序列号，可调用Build.getSerial() ，但是需要申请 READ\_PHONE\_STATE 权限。  
到了Android 10.0以上，则和IMEI一样，也被禁止获取了。  
总体来说，设备序列号有点鸡肋：食之无味，弃之可惜。

#### MAC地址

获取MAC地址也是越来越困难了，  
Android 6.0以后通过 WifiManager 获取到的mac将是固定的：02:00:00:00:00:00 ，  
再后来连读取 /sys/class/net/wlan0/address 也获取不到了。  
如今只剩下面这种方法可以获取（没有开启wifi也可以获取到）：

```java
public static String getWifiMac() {
    try {
        Enumeration<NetworkInterface> enumeration = NetworkInterface.getNetworkInterfaces();
        if (enumeration == null) {
            return "";
        }
        while (enumeration.hasMoreElements()) {
            NetworkInterface netInterface = enumeration.nextElement();
            if (netInterface.getName().equals("wlan0")) {
                return formatMac(netInterface.getHardwareAddress());
            }
        }
    } catch (Exception e) {
        Log.e("tag", e.getMessage(), e);
    }
    return "";
}

```

再往后说不准这种方法也行不通了，且用且珍惜~  
(更新：目前某些系统支持动态MAC，接入不同的wifi设备，获取到的MAC不相同）。

#### ANDROID_ID

Android ID 是获取门槛最低的，不需要任何权限，64bit 的取值范围，唯一性算是很好的了。  
但是不足之处也很明显：  
1、刷机、root、恢复出厂设置等会使得 Android ID 改变；  
2、Android 8.0之后，Android ID的规则发生了[变化](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fabout%2Fversions%2Foreo%2Fandroid-8.0-changes%3Fhl%3Dzh-cn "https://developer.android.com/about/versions/oreo/android-8.0-changes?hl=zh-cn")：

*   对于升级到8.0之前安装的应用，`ANDROID_ID`会保持不变。如果卸载后重新安装的话，`ANDROID_ID`将会改变。
*   对于安装在8.0系统的应用来说，`ANDROID_ID`根据应用签名和用户的不同而不同。`ANDROID_ID`的唯一决定于应用签名、用户和设备三者的组合。

两个规则导致的结果就是：  
第一，如果用户安装APP设备是8.0以下，后来卸载了，升级到8.0之后又重装了应用，Android ID不一样；  
第二，不同签名的APP，获取到的Android ID不一样。  
其中第二点可能对于广告联盟之类的有所影响。

四、 设备ID的特性分析
------------

笔者之前写过一篇文章[《Android设备唯一标识的获取和构造》](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F9d828d259270 "https://www.jianshu.com/p/9d828d259270"), 文中提到设备ID的两个概念：唯一性和稳定性。  

### 4.1 唯一性

唯一性： 两台不同的设备获取到的设备ID不相同。  

分析唯一性，我们可以从ID的分配来入手：

*   1、按规则构造 比如自增ID（包括分步自增），分段构造的ID（如snowflake算法）等，此类ID能保证唯一性。  
    设备ID中的IMEI，设备序列号，MAC等，都是按照规则构造的，理论上能保证唯一性。  
    设备序列号是对厂商本身唯一，全局唯一需要在加上 **Build.MANUFACTURER**。  
    不过，设备序列号和MAC的唯一要打个问号，因为要看厂商是否遵守规则。  
    但随着手机产业的日渐成熟，传统意义上的山寨设备已越来越少，所以大多数情况下还是唯一的。
    
*   2、随机生成 比如UUID和Android ID，这类ID有一定的概率会重复，关键是看ID的长度（有多少bit）。  
    有人做了这样一张随机数的冲突概率表：
    

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/91ccf066-24c3-4abd-abb3-fd10b0ad3184.webp?raw=true)

左边第一栏是bit数量，第二栏是对应的取值范围，再后面是元素个数以及对应的冲突概率。  
例如，假如有50000个32bit的随机数，则这些随机数中，至少有两个相同数字的概率为25%；  
换一种说法，就是假如有四组数，每组都有50000个随机数，则大约其中一组会有重复的数字。  
32bit有43亿的取值范围，怎么50000个随机数就有这么高的出现重复的概率？  
如果对此感到困惑，可以了解一下[生日悖论](https://link.juejin.cn/?target=https%3A%2F%2Fwiki.mbalib.com%2Fwiki%2F%25E7%2594%259F%25E6%2597%25A5%25E6%2582%2596%25E8%25AE%25BA "https://wiki.mbalib.com/wiki/%E7%94%9F%E6%97%A5%E6%82%96%E8%AE%BA")。

Android ID是长度为16的十六进制字符串，其实就是64bit，我们来分析一下其重复的概率：  
假如APP累计激活量达到50亿的APP，则每两个这样的APP就大约有一个会有重复的Android ID。  
不过这里的“重复”不是大量的重复，而是“至少有两个相同”，也就是，如果设备激活量有50亿（很多APP达不到-_-），那么有可能会有少量的重复的Android ID。  
总体而言, Android ID的唯一性还是不错的。  
JDK的randomUUID，大致可以认为是128bit的随机数（其中有6bit是固定的），即使到达200亿的数量，有重复的概率也仅仅是10的负18次方，微乎其微。

### 4.2 稳定性

稳定性：同一台设备在不同的时间, 获取到设备ID相同。  

稳定性有两个层面：

*   1、ID的生命周期 IMEI，序列号，MAC等都是硬件相关，即使刷机也不会改变；  
    Android ID则稳定性较弱，恢复出厂设置和刷机都会改变Android ID。  
    
*   2、受版本的变化的影响 随着Android版本的提升，Google对权限是越收越紧了。  
    获取设备ID的API，要么收起不给用（IMEI）, 要么获取变得困难（SERIAL  
    ），要么不同签名的APP获取的值不一样（Android ID）。  
    同时，Android 10中存储权限也收缩了，之前的那种生成唯一ID写到SD卡的某个角落的，以求卸载重装后读之前的ID等方法也不奏效了。  
    加强隐私方面的权限，对用户而言是好事，但对开发者而言就比较难受了。  
    尤其是有的API本来可以用，升级后就获取不到了，这种断崖式的变化，可能会对数据统计造成影响。

五、设备ID的构造
---------

无论是统计需求，还是业务需求，都要求设备ID是唯一的，稳定的。  
如果设备ID有重复，则活跃统计，用户画像，定向推送等统统都不准确了；  
其中，影响最深是定向推送，送错快递还有可能追回，推送错了就不好说了，如果推送内容又比较重要，后果不堪设想。  
如果设备ID不稳定（ID变化），会影响到活跃统计（会认为是新用户），对用户画像也有较大影响（之前的ID关联的行为数据无法跟踪了）。  

为此，有必要设计一套方案，提供相对定稳定的，唯一的设备ID。  
首先要明确两个前提条件：  
前面分析的设备ID中，在可用的前提下，出现重复的概率较小；  
如果一定的频率去观察，比如说每天，总体而言，观察到和昨天不一样的概率也是较小的。  
如何在本来就较小的概率的前提下，继续降低概率呢？

### 5.1 方案分析

一种方案是组合设备ID(直接拼接，或者拼接后计算摘要)。  
举个例子，假如出现重复的概率和发生变化的的概率都是千分之一，  
则对于两台不同设备，两个设备ID同时重复的概率是百万分之一，两个设备ID至少有一个发生变化约为千分之二。  
也就是，拼接ID的效果是大大提高唯一性，但是一定程度上降低稳定性（只要其中一个要素变化，拼接的ID就变了）。  
但事实上，如今能拿到的设备ID，最突出的矛盾是不稳定，所以，我们不能为了提高唯一性而牺牲稳定性。

要提高稳定性，可以引入容错方案。  
容错方案有很多，比如网络传输，用checksum去校验报文，如果出错了则重发；  
再如磁盘阵列，数据写入两个磁盘，只有当两个磁盘同时出错时才会丢失数据，从而大大降低丢失数据的概率。  
但是对于设备ID，以上两种方案都不合适，因为上面的方案需要通过checksum来确认原信息是否被修改，设备ID没有这样的条件。

所以，可以引入类似虚拟货币用到的"拜占庭容错"方案。  
简单地说，就是要采集三个设备ID到云端，如果有两个（包括两个以上）的设备ID和之前的记录相同，则认为是同一台设备。  
同样假设出现重复的概率和发生变化的的概率都是千分之一，则：  
同一台设备的两次采集，认不出是同一台设备的条件为“至少两个设备ID都和上次不一样”，概率约为百万分之三。  
两台不同的设备，认为是同一台的条件是为“三个设备ID中，至少有两个设备ID和另一台设备相同”，概率同样约为百万分之三。  
所以，用此方案，唯一性和稳定性都能得到提高。

### 5.2 具体实现

基本思想是：服务端有一张设备 ID 的表，核心的属性（Column）有：  
id | did\_1 | did\_2 | did_3  
客户请求时，上传三个设备 ID，服务端检索:  

```sql
SELECT * from t_device_id WHERE did_1=? or did_2=? or did_3=?

```

如果检索到记录，其中至少两个did和上传的相同，则返回 id;  
否则，插入上传的三个设备 ID，并将新插入记录的 id 返回。

然后就是，需要三个设备ID……

*   Android ID 和 MAC地址都还可以取到，再加一个，实施方案的条件就凑齐了。
*   IMEI 需要 READ\_PHONE\_STATE 权限，所以如果不想申请READ\_PHONE\_STATE 权限，可以不采集IMEI了；  
    而且，即使申请了 READ\_PHONE\_STATE，Android 10.0以后也获取不到了。
*   但是设备序列号只有在Android 8.0之前才可以免权限获取；  
    在8.0之后，10.0之前，需READ\_PHONE\_STATE 权限；  
    10.0之后， 有READ\_PHONE\_STATE权限也获取不到了。

那么，如果在没有 READ\_PHONE\_STATE 权限的情况下，以及Android 10.0之后，如何处理？  
首先，设备序列号还是要采集的，毕竟还有部分旧版本的设备可以获取到，能区分一点是一点；  
然后，采集一些设备相关的信息，机型，硬件信息等（相同的机型，可能有多种配置，所以同时也采集一下硬件信息）。  
最终匹配规则如下：

```kotlin
private fun matchDeviceId(deviceIdList: List<DeviceId>, r: DeviceId): DeviceId? {
	if (deviceIdList.isEmpty()) {
		return null
	}
	var maxPriorityDid : DeviceId? = null
	var priority = 0
	deviceIdList.forEach { did ->
		val s = idMatch(did.serial_no, r.serial_no)
		val a = idMatch(did.android_id, r.android_id)
		val m = idMatch(did.mac, r.mac)
		if (s && m && a) {
			return did
		}

		if(priority == 3) return@forEach
		if ((s && (a || m)) || (a && m)) {
			priority = 3
			maxPriorityDid = did
		}

		if(priority >= 2) return@forEach
		val p = idMatch(did.physics_info, r.physics_info)
				|| idMatch(did.dark_physics_info, r.dark_physics_info)
		if (p && a) {
			priority = 2
			maxPriorityDid = did
		}

		if(priority >= 1) return@forEach
		if (p && m) {
			priority = 1
			maxPriorityDid = did
		}
	}
	return maxPriorityDid
}

```

*   如果设备序列号、Android ID、MAC全都不等，则前面的SQL查询不会返回记录（也就是没有匹配的设备）。
*   如果设备序列号，Android ID 和 MAC 全部相同，直接返回。
*   否则，遍历列表，取优先级最高的deviceId返回。
*   如果只有Android ID 或 MAC 之一相等，但是设备信息都匹配不上的话，也认为不是同一个设备。  
    

如果没有匹配的设备，则认为是新设备；  
此时，生成新的udid返回，同时插入新设备的相关信息（设备ID，硬件信息）。

关于硬件信息，需满足一个要求：在设备重启、恢复出厂设置等操作之后，不会变化。  
常规信息有CPU核心数，RAM/ROM大小（以Gb为单位采集，而不是精确到比特，否则容易变化），屏幕分辨率和dpi等，结合机型，保守估计有上千甚至上万种可能性，相对Android ID 的 2^64 当然相差很远了，但是仍可作为辅助的参考信息。  
试想在设备序列号获取不到，Android ID 和 MAC 地址其中一个发生变化时，检索到的都是只有Android ID 或者 MAC 其中一个匹配的记录，茫茫机海，说不准就有一两台的Android ID 或 MAC是相同的。  
这时候选哪一个呢？ 再加上设备信息，或许就区分开了。  
常规的设备信息容易遭到篡改，所以，在常规信息之外，我们可以挖掘一些冷门的设备特征，比如 NetworkInterface 和 传感器 的相关信息。  
当常规信息被篡改时，如果冷门的设备信息还没变，仍可识别出是同一台设备。  
至于如何挖掘，那就各显神通了，通常做手机硬件或者ROM的朋友可能会知道更多的API。  
为了方便检索，我们可以用[MurmurHash](https://link.juejin.cn/?target=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FMurmurHash "https://en.wikipedia.org/wiki/MurmurHash")将信息压缩到64bit(Long的长度）。  

再者，在获取到udid之后，可以定时（比如每隔两天）就上传udid和设备信息给云端，云端比较一下存储的信息和上传的信息，不相同则更新，这样可以提高udid的稳定性。  
比方说，用户在设备是Android 7.0 的时候卸载了APP，在Android 8.0之后安装回来，这时候Android ID 是变化了的，但是凭着MAC和设备信息我们可以认出这台设备，同时更新其 Android ID;  
如果哪一天轮到MAC获取不到了，这时候我们仍可以根据 Android ID和设备信息识别出这台设备。

关于UDID的构造：  
通常情况下，服务端表的主键为自增序列（为了确保插入的有序性），  
所以我们不能直接返回表的主键，否则容易被他人推测其他的设备 ID，以及知晓用户数量。  
因此，在主键 ID 之外，我们需要另外一个唯一 ID。  
有两种思路：

*   随机化，比如用randomUUID 这种方案优点是具有隐蔽性，好处是UUID完全不可能得知主键ID，  
    缺点是占空间，检索效率一般，输入不友好（很多时候我们需要输入设备ID去查询一些数据）。
*   根据主键 id 加密（混淆）出另一个Long类型的id（推荐）。  
    此方案优点是节省空间，检索快，但是要求和主键ID一一映射，以确保不会重复。  
    具体方法可参考[《如何加密Long类型数值》](https://juejin.im/post/6844904078561001486 "https://juejin.im/post/6844904078561001486")。

六、总结
----

本文介绍了设备ID的用途，现状，并分析了现有设备ID的特性，最后提出了一套设备ID的构造方案。  
按照这几年的趋势，各种设备ID的API或许还会越收越紧，仅从客户端去构造可靠的设备ID是比较困难的，而基于信息采集和云端综合计算则相对容易。  

更新：  
随着Android新版本的占用率越来越多，设备序列号获取率越来越低，厂商逐渐支持动态MAC，  
要维持UDID的可靠性，需更多地挖掘设备特征，以及优化匹配算法。

七、Demo
------

具体实现，笔者编写了一个Demo，已发布的到github，谨供参考。  
项目地址：[github.com/BillyWei001…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FBillyWei001%2FUdid "https://github.com/BillyWei001/Udid")  
Demo项目分为两部分UdidClient和UdidServer。  
UdidClient直接用AndroidStudio打开运行即可；  
UdidServer用IDEA打开，打开之后如果不能直接运行，需要手动配置一下Application以及依赖。

**参考资料**  
[www.jianshu.com/p/9d828d259…](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F9d828d259270 "https://www.jianshu.com/p/9d828d259270")  
[juejin.cn/post/684490…](https://juejin.cn/post/6844903605523185671 "https://juejin.cn/post/6844903605523185671")  
[blog.csdn.net/andoop/arti…](https://link.juejin.cn/?target=https%3A%2F%2Fblog.csdn.net%2Fandoop%2Farticle%2Fdetails%2F54633077 "https://blog.csdn.net/andoop/article/details/54633077")  
[blog.csdn.net/renlonggg/a…](https://link.juejin.cn/?target=https%3A%2F%2Fblog.csdn.net%2Frenlonggg%2Farticle%2Fdetails%2F78435986 "https://blog.csdn.net/renlonggg/article/details/78435986")  
[developer.android.com/about/versi…](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fabout%2Fversions%2Foreo%2Fandroid-8.0-changes%3Fhl%3Dzh-cn "https://developer.android.com/about/versions/oreo/android-8.0-changes?hl=zh-cn")  
[en.wikipedia.org/wiki/Birthd…](https://link.juejin.cn/?target=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FBirthday_attack "https://en.wikipedia.org/wiki/Birthday_attack")