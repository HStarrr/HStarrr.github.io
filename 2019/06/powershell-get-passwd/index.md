# 记一次利用SQL调用powershell命令获取管理员密码



<!--more-->

---

## 1. 前言

这篇文章是把以前写在word文档中的内容搬过来了，主要内容就是记录了一次提权过程，通过`MSSQL`权限调用`PowerShell`并执行命令，最终获取了管理员的密码。虽然最后总结的流程很简单，但中间踩过的坑、遇到的问题可是不计其数(自己比较菜)，所以将此过程做了一个记录。

---

## 2. 初试提权

* 首先，已经得到了某个服务器的`IP`和`MSSQL`的`SA`账户和密码，很高兴的连上去，心想`SA`账户都有了，提个权还不是分分钟的事
![](/images/记一次利用SQL调用powershell命令获取管理员密码/f86ee672c3681c0a1f1e909fab7ca02f39f881c9fcee7cf61873e39fefde9ea4.png " ")

* 登路上去之后马上尝试调用`xp_cmdshell`，看管理员是否禁用了`xp_cmdshell`
![](/images/记一次利用SQL调用powershell命令获取管理员密码/0475e8ed90f078e72380652bfb781cc72ec3e692f179eba7107c62bc3a900c8c.png " ")

* 虽然报了一个错误(不清楚是什么错误)，但是还好没有被禁用，而且还是`system权限`，看来提权触手可得，马上使用`net user`创建用户
![](/images/记一次利用SQL调用powershell命令获取管理员密码/b033bdc4c3fd8219bcebef3aa836f93ff3d87e256a08232810ce80b15b0ea341.png " ")

* what xxxx! 什么情况，居然没执行成功，怀疑管理员把`xp_cmdshell`做了什么手脚，于是去百度了许多不利用`xp_cmdshell`来执行命令的`sql`语句
![](/images/记一次利用SQL调用powershell命令获取管理员密码/ec600b20fc2aedf75f6c880da0d943e9a7c4465b957dcd1cef577312b32188be.png " ")

* 但是并没有什么卵用，无一例外都失败了，只能另想办法了，然后发现可以执行`systeminfo`命令，显示是`win2008`
![](/images/记一次利用SQL调用powershell命令获取管理员密码/3834773d7e463419faed8f7f3480994ca6c6d3bd9ba65972983497b7f65e5103.png " ")

* `Nmap`扫描了一下端口，`445`是开启的，刚好这几天学会了`ms17-010`的漏洞，就打算用`msf`直接获取到`shell`，但是想象都是美好的，这服务器好像打了补丁，而且自己没有公网地址，导致反弹不回来`shell`
![](/images/记一次利用SQL调用powershell命令获取管理员密码/691a33361b7abd35a3240ad37b7cc4a77ce60069460529f2b070c7fa2e58c6ed.png " ")

---

## 3. 山穷水尽疑无路

经过上面的测试，我意识到事情并没有那么简单。这台服务器不知道做了啥手脚，`CMD命令`只能执行一部分没有啥危害的命令，执行创建用户这种命令就直接报错。需要换一换思路搞搞。

* 刚才使用`Nmap`扫描了一下端口，观察了一下开启的端口，其中包括很多个`IIS`的`http端口`
![](/images/记一次利用SQL调用powershell命令获取管理员密码/f6d329ec4188b94f6d3b2526cdae16eb8fb13475bd1895ae514126f6eeb5f112.png " ")

* 进了一个网站看了看，发现是需要登录的
![](/images/记一次利用SQL调用powershell命令获取管理员密码/9035058bcff8c31fded872bf5a4ec5ebe8ed581177f94eec509215e106ab737e.png " ")

* 从数据库里找到`admin`的密码，心想能不能从网站获取一个`shell`，进去看了之后我放弃了，全是英文，看着头疼
![](/images/记一次利用SQL调用powershell命令获取管理员密码/c47929bf0472ddb62e70964d4c9ba2dda23625f24c5368e841de8001a860ac9a.png " ")

* 看了一下其余的网站，全是`aspx`的，而且都是英文站，让我放弃了从网站入手的想法。继续翻着端口列表，突然看到一个`Apache`服务
![](/images/记一次利用SQL调用powershell命令获取管理员密码/ceb31fa9d731dbd1b8fc784eed8c0728e143598fa856348d7e838c77be7fc406.png " ")

* 心里很疑惑，这个服务器不是布置的`IIS`吗，怎么`Apache`也在上面，访问一下这个端口试试
![](/images/记一次利用SQL调用powershell命令获取管理员密码/cec002e929e35baa45cbaa03cfc76b656404a4358e85f6b8c38b6de9150819b7.png " ")

* 卧槽，居然是`wamp`刚搭建好的页面，难道运气这么好，又能登录`MySQL`了？
![](/images/记一次利用SQL调用powershell命令获取管理员密码/2258e81cb93bf6dca7f2a1938290063133ed658ae21afe09577bd08d1cbe059a.png " ")

* 果然事不如愿，居然访问不了，看来还是不行啊，又回去看了看`MSSQL`数据库，发现`dir`命令可以执行，通过`dir`命令，发现了`wamp`目录下的`WordPress`，尝试`admin`登录，竟然登录进去了
![](/images/记一次利用SQL调用powershell命令获取管理员密码/6e960b6038cf4b3da41a736939076b193eedaad0dd8ae44219cc90b6b298f36d.png " ")

* 然后百度了一下`WordPress拿shell`，利用`修改文件`内容写进一句话木马
![](/images/记一次利用SQL调用powershell命令获取管理员密码/a677a61709360ca6354219269ff849ca7edb9e0168edb1a93f1968927ff3a293.png " ")

* 用菜刀连接，果然能连接上了(不知道`shell路径`就自己本地搭建一个WordPress测试)
![](/images/记一次利用SQL调用powershell命令获取管理员密码/9316651077a0d03d7c4e113cc1afb563e30a9c6253b53837eeee23af90ef5cf5.png " ")

* 尝试用`cmd`创建用户试试
![](/images/记一次利用SQL调用powershell命令获取管理员密码/8b05fc007ef0f047b2f4e9af5f7ac82c1724d5ab60253b5ef33cbb3cf61dc311.png " ")

* 还是一样无法创建，可能就是因为管理员把`net命令给禁用了`或者给更改了什么的，反正通过创建用户这一条路是不可能的了，只有直接`读取Administrator的密码`才行，上传`getpass64.exe`上去执行，显示超时
![](/images/记一次利用SQL调用powershell命令获取管理员密码/df01958c03b65a8af396678173ebb3a1414639446ef159a924a8ce4f5fca1026.png " ")

* 然后想起是`2008的系统`，`powershell`应该能用，尝试了一下
![](/images/记一次利用SQL调用powershell命令获取管理员密码/daf4d84cb4af735eb7357afc183d079a69ed5392cae596256fa0b1207ae1a70c.png " ")

* 果然能用，然后就想能不能使用`powershell反弹shell`来获取一个权限
![](/images/记一次利用SQL调用powershell命令获取管理员密码/fb9ddc2868fd698cddcad7513f0907468d84f3a9c452362dd92abf05aaa2c59a.png " ")

* 但是好像是没`公网IP`的缘故，也失败了，一直收不到反弹的`shell`
![](/images/记一次利用SQL调用powershell命令获取管理员密码/adfc7a056633a2cf844e94d0956f01b95e70e557be8a337cc9ed1029913ab399.png " ")

---

## 4. 柳暗花明又一村

到这里就很难受了，我感觉能利用的都利用了，能试的方法都试了，但都搞不下来。难道就要放弃了么，可近在咫尺的胜利以及花费的时间让我放弃感到很吃亏，那就继续搞呗。

* 经过[黑无常](#)大佬的提醒，可以利用`msf`调用`mimikatz`来读取密码
![](/images/记一次利用SQL调用powershell命令获取管理员密码/9d4d8148e47911aa748f88bb893185744fdf4188c9e5304fabd6b7165312cad9.png " ")

* 但是我`msf`反弹不回`shell`啊，这个方法有个屁用啊，白高兴一场了
![](/images/记一次利用SQL调用powershell命令获取管理员密码/1dbd10e3024405c984b7196d596abf8bb33bb4f049b4e9782a3c7cf4f3cb2255.png " ")

* 然后一直在网上寻找其他方法，找到了可`利用powershell调用mimikatz来读取密码`
![](/images/记一次利用SQL调用powershell命令获取管理员密码/2217d5983649698f407db1a2abf24bb6c99cd7e0a92e90ad5624f3372a633dd9.png " ")

* 然后尝试在本地执行，本地执行都失败了，这就很尴尬了
![](/images/记一次利用SQL调用powershell命令获取管理员密码/e9b70c8ae39ed0a8b2edbd65d534c7a92621a6320586addf065a7e785eb3a86e.png " ")

* 然后又研究这个原理，终于知道了本地为什么会失败，这个`powershell命令`是在`网上下载powershell脚本`，然后再执行，失败的原因`下载链接不对`，所以导致失败，然后找到了`GitHub`上的脚本，再去本地执行，果然执行成功了
![](/images/记一次利用SQL调用powershell命令获取管理员密码/110f3b8b703d88f4c61a795b790ad78c97a564e2513653b2d12cdc96f91db4bf.png " ")

* 然后放在菜刀里执行，然后开始几次后面执行超时，后面执行时报了如下错误，大概是因为网速太慢或者菜刀执行命令不和cmd一样，具体为什么不太清楚
![](/images/记一次利用SQL调用powershell命令获取管理员密码/c44d6b3e8832d7f1889dd2ae9cb5ba56963b35e847cb3bd5318dee7502b47b11.png " ")

* 然后又想到了`MSSQL`(为什么总能想到它啊，大概是因为首先拿到权限就是因为它吧哈哈哈)，执行`powershell`命令试试
![](/images/记一次利用SQL调用powershell命令获取管理员密码/1f53e251af7fa76ecaf033027b62f7427424829f793578bafebfb71a69f83ddf.png " ")

* 果然可以执行，然后把刚才那段在本地执行的`powershell命令`带进去执行(因为有特殊字符，需要使用转义字符)
![](/images/记一次利用SQL调用powershell命令获取管理员密码/531a8951062bbfebe29a75714d431ad7ba464290c178834aeae63734feee910a.png " ")

* 哇，终于成功了，用了三天时间终于拿下来了，真是不容易，这次提权就算告一段落了
![](/images/记一次利用SQL调用powershell命令获取管理员密码/6a382dcdbe59bdd5bd71c2e27a191206e0a3a3b6c8a2193622ce744d8d796a3c.png " ")

---

## 5. END

这次提权算是学到了不少东西，比如`MSSQL`不只有`xp_cmdshell`可以执行系统命令，还有很多语句都能执行。还有`powershell反弹shell`、`powershell下载脚本执行`等等技术。最重要的是学到了一些道理：

* `不抛弃不放弃`这句话不只是CF的宣言，在现实中也是很好的道理
* `骚思路`一定要多，不能在一个地方吊死

<br />

---

{{< admonition info >}}
2020.05月更新：

该文章的写作时间看上去是19年，实际时间其实更早。从技术水平、写作水平上来说，的确是稍显稚嫩。但这篇文章应该是我入安全圈以来第一篇文章，所以一直保存到现在，里面很多话语不正确的地方也没有去改，当留作纪念吧，望读到该篇文章的大佬们轻喷。
{{< /admonition >}}
